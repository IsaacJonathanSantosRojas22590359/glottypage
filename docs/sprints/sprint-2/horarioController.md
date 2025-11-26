# Documentación del Controlador HorarioController

## Descripción General
El `HorarioController` es un controlador Laravel que gestiona toda la lógica relacionada con la creación, edición, visualización y eliminación de horarios base en el sistema académico. Maneja tanto horarios semanales como sabatinos, incluyendo validaciones complejas y restricciones de integridad referencial.

## Estructura del Controlador

### Namespace y Dependencias
```php
namespace App\Http\Controllers;

use App\Models\Horario;
use App\Models\HorarioPeriodo;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
```

## Métodos Principales

### Método: index()
**Propósito**: Muestra el listado de todos los horarios registrados con estadísticas generales.

```php
public function index()
{
    $horarios = Horario::withCount(['horariosPeriodo'])
                      ->orderBy('tipo')
                      ->orderBy('hora_inicio')
                      ->get();

    $estadisticas = [
        'total' => $horarios->count(),
        'activos' => $horarios->where('activo', true)->count(),
        'semanales' => $horarios->where('tipo', 'semanal')->count(),
        'sabatinos' => $horarios->where('tipo', 'sabatino')->count(),
    ];

    return view('coordinador.horarios.index', compact('horarios', 'estadisticas'));
}
```

**Características**:
- Carga el conteo de horarios de periodo relacionados
- Ordena por tipo y hora de inicio
- Proporciona estadísticas para dashboards

### Método: create()
**Propósito**: Muestra el formulario para crear un nuevo horario base.

```php
public function create()
{
    $diasSemana = ['Lunes', 'Martes', 'Miércoles', 'Jueves', 'Viernes'];
    $diasSabado = ['Sábado'];
    
    return view('coordinador.horarios.create', compact('diasSemana', 'diasSabado'));
}
```

**Parámetros para la vista**:
- `diasSemana`: Días disponibles para horarios semanales
- `diasSabado`: Días disponibles para horarios sabatinos

### Método: store()
**Propósito**: Valida y almacena un nuevo horario base en el sistema.

```php
public function store(Request $request)
{
    \Log::info('Datos del formulario:', $request->all());
    \Log::info('Horarios existentes:', Horario::pluck('nombre')->toArray());

    $request->validate([
        'nombre' => 'required|string|max:255|unique:horarios,nombre',
        'tipo' => 'required|in:semanal,sabatino',
        'dias' => 'required|array|min:1',
        'dias.*' => 'string|in:Lunes,Martes,Miércoles,Jueves,Viernes,Sábado',
        'hora_inicio' => 'required|date_format:H:i',
        'hora_fin' => 'required|date_format:H:i|after:hora_inicio',
    ]);

    // Validaciones específicas por tipo de horario
    if ($request->tipo === 'sabatino') {
        if (!in_array('Sábado', $request->dias)) {
            return redirect()->back()
                ->with('error', 'Los horarios sabatinos deben incluir el día Sábado.')
                ->withInput();
        }
        if (count($request->dias) > 1) {
            return redirect()->back()
                ->with('error', 'Los horarios sabatinos solo pueden incluir el día Sábado.')
                ->withInput();
        }
    }

    if ($request->tipo === 'semanal' && in_array('Sábado', $request->dias)) {
        return redirect()->back()
            ->with('error', 'Los horarios semanales no pueden incluir Sábado.')
            ->withInput();
    }

    try {
        $horario = Horario::create([
            'nombre' => $request->nombre,
            'tipo' => $request->tipo,
            'dias' => $request->dias,
            'hora_inicio' => $request->hora_inicio,
            'hora_fin' => $request->hora_fin,
            'activo' => $request->activo ?? true
        ]);

        Log::info('Horario base creado', [
            'horario_id' => $horario->id,
            'nombre' => $horario->nombre,
            'tipo' => $horario->tipo
        ]);

        return redirect()->route('coordinador.horarios.index')
            ->with('success', 'Horario base creado exitosamente.');

    } catch (\Exception $e) {
        Log::error('Error creando horario base: ' . $e->getMessage());
        return redirect()->back()
            ->with('error', 'Error al crear el horario base.')
            ->withInput();
    }
}
```

**Validaciones Implementadas**:
- **Nombre único** en el sistema
- **Tipo válido** (semanal/sabatino)
- **Días coherentes** con el tipo de horario
- **Horas válidas** con formato correcto

### Método: show()
**Propósito**: Muestra los detalles de un horario específico y los periodos donde está asignado.

```php
public function show(Horario $horario)
{
    $periodosAsignados = $horario->periodosActivos()
                               ->withCount(['grupos'])
                               ->get();

    return view('coordinador.horarios.show', compact('horario', 'periodosAsignados'));
}
```

**Información mostrada**:
- Detalles completos del horario
- Periodos activos que utilizan el horario
- Conteo de grupos por periodo

### Método: edit()
**Propósito**: Muestra el formulario para editar un horario existente.

```php
public function edit(Horario $horario)
{
    if ($horario->enUsoEnPeriodosActivos()) {
        return redirect()->route('coordinador.horarios.index')
            ->with('warning', 'No se puede editar un horario que está en uso en periodos activos.');
    }

    $diasSemana = ['Lunes', 'Martes', 'Miércoles', 'Jueves', 'Viernes'];
    $diasSabado = ['Sábado'];
    
    return view('coordinador.horarios.edit', compact('horario', 'diasSemana', 'diasSabado'));
}
```

**Restricción de Seguridad**: Previene la edición de horarios que están en uso en periodos activos.

### Método: update()
**Propósito**: Actualiza un horario existente con validaciones.

```php
public function update(Request $request, Horario $horario)
{
    if ($horario->enUsoEnPeriodosActivos()) {
        return redirect()->route('coordinador.horarios.index')
            ->with('warning', 'No se puede editar un horario que está en uso en periodos activos.');
    }

    $request->validate([
        'nombre' => 'required|string|max:255|unique:horarios,nombre,' . $horario->id,
        'tipo' => 'required|in:semanal,sabatino',
        'dias' => 'required|array|min:1',
        'dias.*' => 'string|in:Lunes,Martes,Miércoles,Jueves,Viernes,Sábado',
        'hora_inicio' => 'required|date_format:H:i',
        'hora_fin' => 'required|date_format:H:i|after:hora_inicio',
    ]);

    // Validaciones específicas por tipo (igual que store)
    // ...

    try {
        $horario->update([
            'nombre' => $request->nombre,
            'tipo' => $request->tipo,
            'dias' => $request->dias,
            'hora_inicio' => $request->hora_inicio,
            'hora_fin' => $request->hora_fin,
            'activo' => $request->activo ?? $horario->activo
        ]);

        Log::info('Horario base actualizado', [
            'horario_id' => $horario->id,
            'nombre' => $horario->nombre
        ]);

        return redirect()->route('coordinador.horarios.index')
            ->with('success', 'Horario base actualizado exitosamente.');

    } catch (\Exception $e) {
        Log::error('Error actualizando horario base: ' . $e->getMessage());
        return redirect()->back()
            ->with('error', 'Error al actualizar el horario base.')
            ->withInput();
    }
}
```

### Método: destroy()
**Propósito**: Elimina un horario del sistema con validaciones de integridad.

```php
public function destroy(Horario $horario)
{
    if (!$horario->sePuedeEliminar()) {
        return redirect()->route('coordinador.horarios.index')
            ->with('warning', 'No se puede eliminar el horario porque está siendo usado en periodos.');
    }

    try {
        $nombreHorario = $horario->nombre;
        $horario->delete();

        Log::info('Horario base eliminado', ['horario_nombre' => $nombreHorario]);

        return redirect()->route('coordinador.horarios.index')
            ->with('success', "Horario \"{$nombreHorario}\" eliminado exitosamente.");

    } catch (\Exception $e) {
        Log::error('Error eliminando horario base: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al eliminar el horario base.');
    }
}
```

**Validación de Eliminación**: Usa el método `sePuedeEliminar()` del modelo para verificar que no hay referencias activas.

### Método: toggleActivo()
**Propósito**: Cambia el estado activo/inactivo de un horario.

```php
public function toggleActivo(Horario $horario)
{
    try {
        $nuevoEstado = !$horario->activo;
        $horario->update(['activo' => $nuevoEstado]);

        $estadoTexto = $nuevoEstado ? 'activado' : 'desactivado';

        Log::info('Estado de horario cambiado', [
            'horario_id' => $horario->id,
            'nuevo_estado' => $nuevoEstado
        ]);

        return redirect()->route('coordinador.horarios.index')
            ->with('success', "Horario {$estadoTexto} exitosamente.");

    } catch (\Exception $e) {
        Log::error('Error cambiando estado del horario: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al cambiar el estado del horario.');
    }
}
```

## Características de Seguridad

### Validaciones de Integridad Referencial
- **No edición** de horarios en uso en periodos activos
- **No eliminación** de horarios con periodos asociados
- **Validación de tipo** coherente con días seleccionados

### Manejo de Logs
```php
Log::info('Horario base creado', [...]);  // Para operaciones exitosas
Log::error('Error creando horario base: ...'); // Para errores
```

### Mensajes de Respuesta
- **Éxito**: Mensajes descriptivos con confirmación de operación
- **Error**: Mensajes específicos con redirección al formulario
- **Advertencia**: Para restricciones de negocio

## Flujos de Validación Específicos

### Para Horarios Sabatinos
1. Debe incluir exclusivamente el día "Sábado"
2. No puede tener más de un día seleccionado
3. No puede incluir días de la semana

### Para Horarios Semanales
1. No puede incluir el día "Sábado"
2. Debe incluir al menos un día de Lunes a Viernes
3. Puede tener múltiples días seleccionados

## Mejores Prácticas Implementadas

### Uso de Route Model Binding
```php
public function edit(Horario $horario)  // Laravel resuelve automáticamente el modelo
```

### Validaciones en Dos Niveles
1. **Validación de Formulario**: Reglas básicas de Laravel
2. **Validación de Negocio**: Lógica específica del dominio

### Manejo de Excepciones
- Try-catch en operaciones críticas
- Logging detallado para debugging
- Respuestas apropiadas al usuario

### Métodos RESTful
- `index`, `create`, `store`, `show`, `edit`, `update`, `destroy`
- Método adicional `toggleActivo` para funcionalidad específica

Este controlador proporciona una gestión completa y segura de los horarios base del sistema, con validaciones robustas que garantizan la integridad de los datos y previenen conflictos en las asignaciones académicas.