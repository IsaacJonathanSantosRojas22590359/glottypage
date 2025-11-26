# Documentación del Controlador AulaController

## Descripción General
El controlador AulaController gestiona toda la lógica relacionada con la administración de aulas académicas en el sistema. Proporciona funcionalidades completas para crear, editar, visualizar y eliminar aulas, así como gestionar su disponibilidad y asignación a grupos. Incluye validaciones avanzadas de capacidad, verificación de horarios ocupados y estadísticas de uso.

## Estructura del Controlador

### Namespace y Dependencias
```php
namespace App\Http\Controllers;

use App\Models\Aula;
use App\Models\Grupo;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\Rule;
```

## Métodos Principales

### Método: index()
Muestra una lista paginada de todas las aulas con sistema de filtros avanzado.

```php
public function index(Request $request)
{
    $query = Aula::query();
    
    // Filtro por edificio
    if ($request->has('edificio') && $request->edificio) {
        $query->where('edificio', $request->edificio);
    }
    
    // Filtro por tipo
    if ($request->has('tipo') && $request->tipo) {
        $query->where('tipo', $request->tipo);
    }
    
    // Filtro por disponibilidad
    if ($request->has('disponible') && $request->disponible !== '') {
        $query->where('disponible', $request->disponible);
    }
    
    $aulas = $query->orderBy('edificio')
                   ->orderBy('nombre')
                   ->get();
    
    // Datos para los filtros
    $edificios = Aula::distinct()->pluck('edificio')->sort();
    $tiposAula = Aula::TIPOS_AULA;
                   
    return view('coordinador.aulas.index', compact('aulas', 'edificios', 'tiposAula'));
}
```

### Método: create()
Muestra el formulario para crear una nueva aula.

```php
public function create()
{
    $tiposAula = Aula::TIPOS_AULA;
    return view('coordinador.aulas.create', compact('tiposAula'));
}
```

### Método: store()
Valida y almacena una nueva aula en la base de datos con manejo de transacciones.

```php
public function store(Request $request)
{
    $validatedData = $request->validate([
        'nombre' => 'required|string|max:100|unique:aulas,nombre',
        'edificio' => 'required|string|max:50',
        'capacidad' => 'required|integer|min:1|max:200',
        'tipo' => ['required', Rule::in(array_keys(Aula::TIPOS_AULA))],
        'equipamiento' => 'nullable|string|max:500',
        'disponible' => 'boolean'
    ]);

    try {
        DB::beginTransaction();

        Aula::create($validatedData);

        DB::commit();

        return redirect()->route('coordinador.aulas.index')
            ->with('success', 'Aula creada exitosamente.');

    } catch (\Exception $e) {
        DB::rollBack();
        
        return back()->with('error', 'Error al crear el aula: ' . $e->getMessage())
                    ->withInput();
    }
}
```

### Método: edit()
Muestra el formulario para editar un aula existente incluyendo información de uso.

```php
public function edit(Aula $aula)
{
    $tiposAula = Aula::TIPOS_AULA;
    
    // Obtener información de uso del aula
    $gruposAsignados = $aula->grupos()
        ->with(['periodo', 'horario'])
        ->whereNotIn('estado', ['cancelado'])
        ->get();
        
    $horariosOcupados = $aula->obtenerHorariosOcupados();
    
    return view('coordinador.aulas.edit', compact(
        'aula', 
        'tiposAula', 
        'gruposAsignados',
        'horariosOcupados'
    ));
}
```

### Método: update()
Actualiza un aula existente con validaciones de capacidad y manejo de transacciones.

```php
public function update(Request $request, Aula $aula)
{
    $validatedData = $request->validate([
        'nombre' => [
            'required',
            'string',
            'max:100',
            Rule::unique('aulas')->ignore($aula->id)
        ],
        'edificio' => 'required|string|max:50',
        'capacidad' => 'required|integer|min:1|max:200',
        'tipo' => ['required', Rule::in(array_keys(Aula::TIPOS_AULA))],
        'equipamiento' => 'nullable|string|max:500',
        'disponible' => 'boolean'
    ]);

    try {
        DB::beginTransaction();

        // Validar que la capacidad no sea menor a los grupos actuales
        if ($validatedData['capacidad'] < $aula->capacidad) {
            $gruposConMayorCapacidad = $aula->grupos()
                ->where('capacidad_maxima', '>', $validatedData['capacidad'])
                ->whereNotIn('estado', ['cancelado'])
                ->exists();
                
            if ($gruposConMayorCapacidad) {
                throw new \Exception('No se puede reducir la capacidad. Hay grupos asignados que requieren más capacidad.');
            }
        }

        $aula->update($validatedData);

        DB::commit();

        return redirect()->route('coordinador.aulas.index')
            ->with('success', 'Aula actualizada exitosamente.');

    } catch (\Exception $e) {
        DB::rollBack();
        
        return back()->with('error', 'Error al actualizar el aula: ' . $e->getMessage())
                    ->withInput();
    }
}
```

### Método: destroy()
Elimina un aula previa verificación de que no tenga grupos activos asignados.

```php
public function destroy(Aula $aula)
{
    try {
        // Verificar grupos activos
        if ($aula->grupos()->whereNotIn('estado', ['cancelado'])->count() > 0) {
            return redirect()->route('coordinador.aulas.index')
                ->with('error', 'No se puede eliminar el aula. Tiene grupos activos asignados.');
        }

        $aula->delete();

        return redirect()->route('coordinador.aulas.index')
            ->with('success', 'Aula eliminada correctamente.');

    } catch (\Exception $e) {
        return redirect()->route('coordinador.aulas.index')
            ->with('error', 'Error al eliminar el aula: ' . $e->getMessage());
    }
}
```

### Método: toggleDisponible()
Cambia el estado de disponibilidad del aula con validación de grupos activos.

```php
public function toggleDisponible(Aula $aula)
{
    try {
        // Validar que no tenga grupos activos si se va a desactivar
        if ($aula->disponible && $aula->grupos()->whereNotIn('estado', ['cancelado'])->count() > 0) {
            return redirect()->route('coordinador.aulas.index')
                ->with('error', 'No se puede desactivar el aula. Tiene grupos activos asignados.');
        }

        $aula->update([
            'disponible' => !$aula->disponible
        ]);

        $estado = $aula->disponible ? 'disponible' : 'no disponible';
        
        return redirect()->route('coordinador.aulas.index')
            ->with('success', "Aula marcada como {$estado}.");

    } catch (\Exception $e) {
        return redirect()->route('coordinador.aulas.index')
            ->with('error', 'Error al cambiar estado del aula: ' . $e->getMessage());
    }
}
```

### Método: show()
Muestra información detallada del aula incluyendo grupos asignados y estadísticas.

```php
public function show(Aula $aula)
{
    $gruposAsignados = $aula->grupos()
        ->with(['periodo', 'horario', 'profesor'])
        ->whereNotIn('estado', ['cancelado'])
        ->orderBy('horario_periodo_id')
        ->get();
        
    $horariosOcupados = $aula->obtenerHorariosOcupados();
    $estadisticas = $this->obtenerEstadisticasAula($aula);
    
    return view('coordinador.aulas.show', compact(
        'aula',
        'gruposAsignados', 
        'horariosOcupados',
        'estadisticas'
    ));
}
```

### Método: obtenerEstadisticasAula()
Método privado que calcula estadísticas de uso del aula.

```php
private function obtenerEstadisticasAula(Aula $aula)
{
    $gruposActivos = $aula->grupos()
        ->whereNotIn('estado', ['cancelado'])
        ->get();
        
    return [
        'total_grupos' => $gruposActivos->count(),
        'grupos_activos' => $gruposActivos->where('estado', 'activo')->count(),
        'ocupacion_promedio' => $gruposActivos->avg('porcentaje_ocupacion') ?? 0,
        'capacidad_utilizada' => $gruposActivos->sum('estudiantes_inscritos'),
        'horarios_ocupados' => $gruposActivos->unique('horario_periodo_id')->count()
    ];
}
```

### Método: disponiblesParaHorario()
API que devuelve aulas disponibles para un horario específico.

```php
public function disponiblesParaHorario(Request $request)
{
    $request->validate([
        'horario_periodo_id' => 'required|exists:horarios_periodo,id',
        'capacidad_requerida' => 'nullable|integer|min:1'
    ]);

    try {
        $aulas = Aula::disponiblesParaHorario(
            $request->horario_periodo_id,
            $request->capacidad_requerida
        );

        return response()->json([
            'success' => true,
            'aulas' => $aulas->map(function ($aula) {
                return [
                    'id' => $aula->id,
                    'nombre_completo' => $aula->nombre_completo,
                    'capacidad' => $aula->capacidad,
                    'tipo' => $aula->tipo_formateado,
                    'info_resumida' => $aula->info_resumida
                ];
            })
        ]);

    } catch (\Exception $e) {
        return response()->json([
            'success' => false,
            'message' => 'Error al obtener aulas disponibles: ' . $e->getMessage()
        ], 500);
    }
}
```

## Características de Seguridad y Validaciones

### Validaciones Implementadas
- **Nombre único** en el sistema para evitar duplicados
- **Capacidad válida** entre 1 y 200 estudiantes
- **Tipo de aula** dentro de los valores predefinidos
- **Verificación de grupos activos** antes de eliminar o desactivar
- **Validación de capacidad** al reducir el tamaño del aula

### Manejo de Transacciones
- Uso de `DB::beginTransaction()` y `DB::commit()` en operaciones críticas
- `DB::rollBack()` en caso de errores para mantener la integridad de datos

### Prevención de Conflictos
- No permite eliminar aulas con grupos activos
- No permite desactivar aulas con asignaciones vigentes
- Valida que la capacidad no sea menor a la requerida por grupos existentes

## Relaciones con Otros Modelos

### Relación con Grupo
El controlador utiliza la relación `grupos()` para:
- Verificar asignaciones activas
- Calcular estadísticas de uso
- Validar restricciones de capacidad

### Relación con Horario
A través del método `obtenerHorariosOcupados()` se consulta la disponibilidad del aula en diferentes periodos.

## Respuestas al Usuario

### Mensajes de Éxito
- Aula creada exitosamente
- Aula actualizada exitosamente  
- Aula eliminada correctamente
- Aula marcada como disponible/no disponible

### Mensajes de Error
- Error al crear/actualizar/eliminar el aula
- No se puede reducir la capacidad
- No se puede eliminar/desactivar el aula con grupos activos

Este controlador proporciona una gestión completa y segura de las aulas académicas con validaciones robustas y manejo apropiado de errores, garantizando la integridad de los datos y la consistencia del sistema.