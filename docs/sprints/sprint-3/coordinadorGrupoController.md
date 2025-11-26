# Documentación del Controlador CoordinadorGrupoController

## Descripción General
El controlador `CoordinadorGrupoController` gestiona toda la lógica relacionada con la administración de grupos académicos en el sistema. Proporciona funcionalidades completas para crear, editar, visualizar y gestionar grupos, incluyendo la asignación de estudiantes, gestión de capacidad y control de estados.

## Estructura del Controlador

### Namespace y Dependencias
```php
namespace App\Http\Controllers;

use App\Models\Grupo;
use App\Models\Periodo;
use App\Models\HorarioPeriodo;
use App\Models\Aula;
use App\Models\Profesor;
use App\Models\Preregistro;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
```

## Métodos Principales

### Método: index(Request $request)
Muestra la lista paginada de grupos con sistema de filtros avanzados y estadísticas.

```php
public function index(Request $request)
{
    $query = Grupo::with(['periodo', 'horario', 'aula', 'profesor', 'estudiantesActivos'])
        ->latest();

    // Aplicación de filtros
    if ($request->filled('nivel')) {
        $query->where('nivel_ingles', $request->nivel);
    }

    if ($request->filled('estado')) {
        $query->where('estado', $request->estado);
    }

    if ($request->filled('periodo_id')) {
        $query->where('periodo_id', $request->periodo_id);
    }

    $grupos = $query->paginate(20);
    $periodos = Periodo::all();
    $periodoActivo = Periodo::conPreRegistrosActivos()->first();

    $estadisticas = $this->calcularEstadisticas($request);

    return view('coordinador.grupos.index', compact(
        'grupos', 
        'periodos', 
        'periodoActivo',
        'estadisticas' 
    ));
}
```

**Características**:
- Carga relaciones necesarias para optimización
- Sistema de filtros por nivel, estado y periodo
- Paginación de 20 elementos por página
- Cálculo de estadísticas en tiempo real

### Método: calcularEstadisticas(Request $request)
Método privado que calcula métricas detalladas sobre los grupos.

```php
private function calcularEstadisticas(Request $request)
{
    $query = Grupo::query();

    // Aplicar mismos filtros que en el index
    if ($request->filled('nivel')) {
        $query->where('nivel_ingles', $request->nivel);
    }

    if ($request->filled('estado')) {
        $query->where('estado', $request->estado);
    }

    if ($request->filled('periodo_id')) {
        $query->where('periodo_id', $request->periodo_id);
    }

    $grupos = $query->get();

    return [
        'total' => $grupos->count(),
        'planificados' => $grupos->where('estado', 'planificado')->count(),
        'con_profesor' => $grupos->where('estado', 'con_profesor')->count(),
        'con_aula' => $grupos->where('estado', 'con_aula')->count(),
        'activos' => $grupos->where('estado', 'activo')->count(),
        'cancelados' => $grupos->where('estado', 'cancelado')->count(),
        'capacidad_total' => $grupos->sum('capacidad_maxima'),
        'capacidad_utilizada' => $grupos->sum('estudiantes_inscritos'),
        'ocupacion_promedio' => $grupos->avg('porcentaje_ocupacion') ?? 0
    ];
}
```

### Método: create()
Muestra el formulario para crear un nuevo grupo.

```php
public function create()
{
    $periodos = Periodo::whereIn('estado', ['configuracion', 'preregistros_activos', 'preregistros_cerrados'])->get();
    $horarios = HorarioPeriodo::where('activo', true)->get();
    
    $aulas = Aula::where('disponible', true)->get();
    $profesores = Profesor::all();
    
    $periodoActivo = Periodo::conPreRegistrosActivos()->first();

    return view('coordinador.grupos.create', compact(
        'periodos',
        'horarios',
        'aulas',
        'profesores',
        'periodoActivo'
    ));
}
```

### Método: store(Request $request)
Valida y almacena un nuevo grupo en el sistema con validaciones complejas.

```php
public function store(Request $request)
{
    $request->validate([
        'nivel_ingles' => 'required|integer|between:1,5',
        'letra_grupo' => 'required|string|size:1|in:A,B,C,D,E,F',
        'periodo_id' => 'required|exists:periodos,id',
        'horario_periodo_id' => 'required|exists:horarios_periodo,id',
        'aula_id' => 'nullable|exists:aulas,id',
        'profesor_id' => 'nullable|exists:profesores,id_profesor',
        'capacidad_maxima' => 'required|integer|min:15|max:40'
    ]);

    try {
        // Verificar unicidad del grupo
        $grupoExistente = Grupo::where('periodo_id', $request->periodo_id)
            ->where('nivel_ingles', $request->nivel_ingles)
            ->where('letra_grupo', $request->letra_grupo)
            ->first();

        if ($grupoExistente) {
            return back()->with('error', 
                "Ya existe el grupo {$request->nivel_ingles}-{$request->letra_grupo} en este periodo."
            )->withInput();
        }

        // Validaciones adicionales usando transacción
        return DB::transaction(function () use ($request) {
            // Crear instancia para validaciones
            $grupo = new Grupo();
            $grupo->fill($request->all());
            
            // Validar aula si está asignada
            if ($request->aula_id) {
                $grupo->validarAsignacionAula($request->aula_id);
            }

            // Validar profesor si está asignado
            if ($request->profesor_id) {
                $grupo->validarAsignacionProfesor($request->profesor_id);
            }

            // Determinar estado automáticamente
            $estado = $this->determinarEstadoInicial($request);
            
            // Crear grupo
            $grupo = Grupo::create([
                'nivel_ingles' => $request->nivel_ingles,
                'letra_grupo' => $request->letra_grupo,
                'periodo_id' => $request->periodo_id,
                'horario_periodo_id' => $request->horario_periodo_id,
                'aula_id' => $request->aula_id,
                'profesor_id' => $request->profesor_id,
                'capacidad_maxima' => $request->capacidad_maxima,
                'estado' => $estado,
                'estudiantes_inscritos' => 0
            ]);

            return redirect()->route('coordinador.grupos.show', $grupo->id)
                ->with('success', "Grupo {$grupo->nombre_completo} creado exitosamente.");
                
        });

    } catch (\Exception $e) {
        return back()->with('error', 'Error al crear grupo: ' . $e->getMessage())->withInput();
    }
}
```

### Método: show($id)
Muestra los detalles completos de un grupo específico.

```php
public function show($id)
{
    $grupo = Grupo::with([
        'periodo',
        'horario',
        'aula',
        'profesor',
        'preregistros.usuario',
        'estudiantesActivos.usuario'
    ])->findOrFail($id);

    // Estudiantes disponibles para asignar
    $estudiantesDisponibles = Preregistro::with(['usuario', 'horarioPreferido'])
        ->where('periodo_id', $grupo->periodo_id)
        ->where('nivel_solicitado', $grupo->nivel_ingles)
        ->where('estado', 'pendiente')
        ->whereIn('pago_estado', ['pagado', 'prorroga'])
        ->get();

    return view('coordinador.grupos.show', compact(
        'grupo',
        'estudiantesDisponibles'
    ));
}
```

### Método: edit($id)
Muestra el formulario para editar un grupo existente.

```php
public function edit($id)
{
    $grupo = Grupo::findOrFail($id);
    $periodos = Periodo::whereIn('estado', ['configuracion', 'preregistros_activos', 'preregistros_cerrados'])->get();
    $horarios = HorarioPeriodo::where('activo', true)->get();
    
    $aulas = Aula::where('disponible', true)->get();
    $profesores = Profesor::all();

    return view('coordinador.grupos.edit', compact(
        'grupo',
        'periodos',
        'horarios',
        'aulas',
        'profesores'
    ));
}
```

### Método: update(Request $request, $id)
Actualiza un grupo existente con validaciones de integridad.

```php
public function update(Request $request, $id)
{
    $grupo = Grupo::findOrFail($id);

    $request->validate([
        'horario_periodo_id' => 'required|exists:horarios_periodo,id',
        'aula_id' => 'nullable|exists:aulas,id',
        'profesor_id' => 'nullable|exists:profesores,id_profesor',
        'capacidad_maxima' => 'required|integer|min:15|max:40',
    ]);

    try {
        // Verificar que la capacidad no sea menor a los estudiantes inscritos
        if ($request->capacidad_maxima < $grupo->estudiantes_inscritos) {
            return back()->with('error', 
                "La capacidad no puede ser menor a los estudiantes inscritos ({$grupo->estudiantes_inscritos})."
            )->withInput();
        }

        // Validaciones adicionales usando transacción
        return DB::transaction(function () use ($request, $grupo) {
            // Validar aula si está asignada o cambió
            if ($request->aula_id && $request->aula_id != $grupo->aula_id) {
                $grupo->validarAsignacionAula($request->aula_id);
            }

            // Validar profesor si está asignado o cambió  
            if ($request->profesor_id && $request->profesor_id != $grupo->profesor_id) {
                $grupo->validarAsignacionProfesor($request->profesor_id);
            }

            // Determinar estado automáticamente
            $estado = $this->determinarEstado($request, $grupo);

            $grupo->update([
                'horario_periodo_id' => $request->horario_periodo_id,
                'aula_id' => $request->aula_id,
                'profesor_id' => $request->profesor_id,
                'capacidad_maxima' => $request->capacidad_maxima,
                'estado' => $estado
            ]);

            return redirect()->route('coordinador.grupos.index')
                ->with('success', "Grupo {$grupo->nombre_completo} actualizado exitosamente.");
        });

    } catch (\Exception $e) {
        return back()->with('error', 'Error al actualizar grupo: ' . $e->getMessage())->withInput();
    }
}
```

### Métodos de Gestión de Estado

#### determinarEstadoInicial(Request $request)
Determina el estado inicial al crear un grupo.

```php
private function determinarEstadoInicial(Request $request)
{
    if ($request->profesor_id && $request->aula_id) {
        return 'activo';
    } elseif ($request->profesor_id) {
        return 'con_profesor';
    } elseif ($request->aula_id) {
        return 'con_aula';
    } else {
        return 'planificado';
    }
}
```

#### determinarEstado(Request $request, Grupo $grupo)
Determina el estado automáticamente al actualizar, respetando estados finales.

```php
private function determinarEstado(Request $request, Grupo $grupo)
{
    // Si el grupo ya estaba activo o cancelado, mantener ese estado
    $estadosFinales = ['activo', 'cancelado'];
    
    // Si ya está activo o cancelado, mantener ese estado
    if (in_array($grupo->estado, $estadosFinales)) {
        return $grupo->estado;
    }

    // Manejar valores nulos/vacíos
    $tieneProfesor = ($request->profesor_id !== null && $request->profesor_id !== '');
    $tieneAula = ($request->aula_id !== null && $request->aula_id !== '');

    // Lógica automática de estados
    if ($tieneProfesor && $tieneAula) {
        return 'activo';
    } elseif ($tieneProfesor) {
        return 'con_profesor';
    } elseif ($tieneAula) {
        return 'con_aula';
    } else {
        return 'planificado';
    }
}
```

### Métodos de Gestión de Estudiantes

#### asignarEstudiante(Request $request, $id)
Asigna un estudiante a un grupo específico.

```php
public function asignarEstudiante(Request $request, $id)
{
    $grupo = Grupo::findOrFail($id);
    
    $request->validate([
        'preregistro_id' => 'required|exists:preregistros,id'
    ]);

    try {
        $preregistro = Preregistro::findOrFail($request->preregistro_id);

        if (!$preregistro->puedeSerAsignado()) {
            return back()->with('error', 'Este estudiante no puede ser asignado a un grupo.');
        }

        if (!$grupo->tieneCapacidad()) {
            return back()->with('error', 'El grupo no tiene capacidad disponible.');
        }

        $grupo->asignarEstudiante($preregistro->id);

        return back()->with('success', 'Estudiante asignado al grupo exitosamente.');

    } catch (\Exception $e) {
        return back()->with('error', $e->getMessage());
    }
}
```

#### removerEstudiante(Request $request, $id)
Remueve un estudiante de un grupo.

```php
public function removerEstudiante(Request $request, $id)
{
    $grupo = Grupo::findOrFail($id);
    
    $request->validate([
        'preregistro_id' => 'required|exists:preregistros,id'
    ]);

    try {
        $grupo->removerEstudiante($request->preregistro_id);

        return back()->with('success', 'Estudiante removido del grupo exitosamente.');

    } catch (\Exception $e) {
        return back()->with('error', $e->getMessage());
    }
}
```

### Método: cambiarEstado(Request $request, $id)
Permite cambiar manualmente el estado de un grupo con validaciones.

```php
public function cambiarEstado(Request $request, $id)
{
    $grupo = Grupo::findOrFail($id);

    $request->validate([
        'estado' => 'required|in:planificado,con_profesor,con_aula,activo,cancelado'
    ]);

    try {
        // Validaciones específicas por estado
        if ($request->estado === 'activo' && !$grupo->puedeSerActivo()) {
            return back()->with('error', 
                'El grupo necesita profesor, aula y capacidad disponible para activarse.'
            );
        }

        if ($request->estado === 'cancelado' && !$grupo->puedeSerCancelado()) {
            return back()->with('error', 
                'No se puede cancelar un grupo con estudiantes inscritos.'
            );
        }

        $grupo->update(['estado' => $request->estado]);

        return back()->with('success', 'Estado del grupo actualizado exitosamente.');

    } catch (\Exception $e) {
        return back()->with('error', 'Error al cambiar estado: ' . $e->getMessage());
    }
}
```

## Características de Seguridad y Validaciones

### Validaciones Implementadas
- **Unicidad de grupo**: Mismo nivel y letra en un periodo
- **Capacidad mínima/máxima**: Entre 15 y 40 estudiantes
- **Integridad de capacidad**: No reducir capacidad por debajo de inscritos actuales
- **Validación de relaciones**: Aula y profesor existen y están disponibles
- **Estados válidos**: Transiciones de estado controladas

### Manejo de Transacciones
Uso de transacciones de base de datos para operaciones críticas que involucran múltiples validaciones y actualizaciones.

### Estados del Grupo
1. **planificado**: Grupo creado sin asignaciones
2. **con_profesor**: Tiene profesor asignado
3. **con_aula**: Tiene aula asignada  
4. **activo**: Tiene profesor, aula y capacidad disponible
5. **cancelado**: Grupo desactivado (no permite estudiantes)

## Flujos de Trabajo Principales

### Creación de Grupo
1. Validar datos del formulario
2. Verificar unicidad del grupo
3. Validar asignaciones de aula y profesor
4. Determinar estado automáticamente
5. Crear grupo con transacción

### Actualización de Grupo
1. Validar capacidad vs estudiantes inscritos
2. Validar cambios en aula y profesor
3. Recalcular estado automáticamente
4. Actualizar con transacción

### Gestión de Estudiantes
1. Verificar que el estudiante puede ser asignado
2. Verificar capacidad del grupo
3. Ejecutar asignación/remoción
4. Actualizar contadores

Este controlador proporciona una gestión completa de grupos académicos con validaciones robustas que garantizan la integridad de los datos y previenen conflictos en las asignaciones.