# Documentación del Controlador PeriodoController

## Descripción General
El controlador `PeriodoController` gestiona toda la lógica relacionada con la administración de periodos académicos en el sistema. Proporciona funcionalidades completas para crear, editar, visualizar y gestionar periodos, incluyendo la gestión de horarios, transiciones de estado y validaciones complejas de negocio.

## Estructura del Controlador

### Namespace y Dependencias
```php
namespace App\Http\Controllers;

use App\Models\Periodo;
use App\Models\Horario;
use App\Models\HorarioPeriodo;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Carbon\Carbon;
```

## Métodos Principales

### Método: index()
Muestra la lista paginada de periodos con conteos estadísticos y filtros por estado.

```php
public function index()
{
    $query = Periodo::withCount([
        'grupos',
        'preregistros',
        'preregistros as preregistros_pagados_count' => function($query) {
            $query->where('pago_estado', 'pagado');
        },
        'horariosPeriodo as horarios_activos_count' => function($query) {
            $query->where('activo', true);
        }
    ])->orderBy('fecha_inicio', 'desc');

    if (request('estado')) {
        $query->where('estado', request('estado'));
    }

    $periodos = $query->paginate(10);

    return view('coordinador.periodos.index', compact('periodos'));
}
```

### Método: create()
Muestra el formulario para crear un nuevo periodo.

```php
public function create()
{
    return view('coordinador.periodos.create');
}
```

### Método: store(Request $request)
Valida y almacena un nuevo periodo en el sistema.

```php
public function store(Request $request)
{
    $request->validate([
        'nombre_periodo' => 'required|string|max:50|unique:periodos,nombre_periodo',
        'fecha_inicio' => 'required|date|after:today',
        'fecha_fin' => 'required|date|after:fecha_inicio',
    ]);

    try {
        DB::beginTransaction();

        $periodo = Periodo::create([
            'nombre_periodo' => $request->nombre_periodo,
            'fecha_inicio' => $request->fecha_inicio,
            'fecha_fin' => $request->fecha_fin,
            'estado' => 'configuracion'
        ]);

        DB::commit();

        return redirect()->route('coordinador.periodos.show', $periodo)
            ->with('success', 'Período creado exitosamente. Ahora puedes agregar horarios desde plantillas.');

    } catch (\Exception $e) {
        DB::rollBack();
        
        Log::error('Error creando período: ' . $e->getMessage(), [
            'request_data' => $request->all(),
            'error_trace' => $e->getTraceAsString()
        ]);
        
        return redirect()->back()
            ->with('error', 'Error al crear el período: ' . $e->getMessage())
            ->withInput();
    }
}
```

### Método: show(Periodo $periodo)
Muestra los detalles completos de un periodo específico.

```php
public function show(Periodo $periodo)
{
    $horariosPeriodo = $periodo->horariosPeriodo()
                            ->with('horarioBase')
                            ->orderBy('tipo')
                            ->orderBy('hora_inicio')
                            ->get();
    
    $estadisticas = $this->obtenerEstadisticasDetalladas($periodo->id);
    
    $horariosBaseDisponibles = Horario::where('activo', true)
        ->whereNotIn('id', function($query) use ($periodo) {
            $query->select('horario_base_id')
                  ->from('horarios_periodo')
                  ->where('periodo_id', $periodo->id);
        })
        ->get();

    return view('coordinador.periodos.show', compact(
        'periodo', 
        'horariosPeriodo', 
        'estadisticas',
        'horariosBaseDisponibles'
    ));
}
```

### Método: edit(Periodo $periodo)
Muestra el formulario para editar un periodo existente.

```php
public function edit(Periodo $periodo)
{
    if (!$periodo->estaEnConfiguracion()) {
        return redirect()->route('coordinador.periodos.index')
            ->with('warning', 'Solo se pueden editar periodos en estado "Configuración"');
    }

    return view('coordinador.periodos.edit', compact('periodo'));
}
```

### Método: update(Request $request, Periodo $periodo)
Actualiza un periodo existente con validaciones de estado.

```php
public function update(Request $request, Periodo $periodo)
{
    if (!$periodo->estaEnConfiguracion()) {
        return redirect()->route('coordinador.periodos.index')
            ->with('warning', 'Solo se pueden editar periodos en estado "Configuración"');
    }

    $request->validate([
        'fecha_inicio' => 'required|date',
        'fecha_fin' => 'required|date|after:fecha_inicio',
    ]);

    try {
        $periodo->update([
            'fecha_inicio' => $request->fecha_inicio,
            'fecha_fin' => $request->fecha_fin,
        ]);

        Log::info('Periodo actualizado (solo fechas)', [
            'periodo_id' => $periodo->id,
            'fecha_inicio' => $periodo->fecha_inicio,
            'fecha_fin' => $periodo->fecha_fin
        ]);

        return redirect()->route('coordinador.periodos.show', $periodo)
            ->with('success', 'Fechas del periodo actualizadas exitosamente.');

    } catch (\Exception $e) {
        Log::error('Error actualizando periodo: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al actualizar el periodo.');
    }
}
```

### Método: destroy(Periodo $periodo)
Elimina un periodo del sistema con validaciones estrictas.

```php
public function destroy(Periodo $periodo)
{
    if (!$periodo->estaEnConfiguracion()) {
        return redirect()->route('coordinador.periodos.index')
            ->with('warning', 'Solo se pueden eliminar periodos en estado "Configuración"');
    }

    if (!$periodo->puedeEliminarse()) {
        return redirect()->route('coordinador.periodos.index')
            ->with('warning', 'No se puede eliminar el periodo porque tiene grupos o pre-registros asociados.');
    }

    try {
        $nombrePeriodo = $periodo->nombre_periodo;
        $periodo->delete();

        Log::info('Periodo eliminado', ['periodo_nombre' => $nombrePeriodo]);

        return redirect()->route('coordinador.periodos.index')
            ->with('success', "Periodo \"{$nombrePeriodo}\" eliminado exitosamente.");

    } catch (\Exception $e) {
        Log::error('Error eliminando periodo: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al eliminar el periodo.');
    }
}
```

## Métodos de Gestión de Horarios

### Método: agregarHorarios(Request $request, Periodo $periodo)
Agrega horarios base a un periodo específico.

```php
public function agregarHorarios(Request $request, Periodo $periodo)
{
    if ($periodo->estaFinalizado() || $periodo->estaCancelado()) {
        return redirect()->back()
            ->with('error', 'No se pueden agregar horarios en periodos finalizados o cancelados');
    }

    $request->validate([
        'horarios_base_ids' => 'required|array|min:1',
        'horarios_base_ids.*' => 'exists:horarios,id'
    ]);

    try {
        DB::transaction(function () use ($periodo, $request) {
            foreach ($request->horarios_base_ids as $horarioBaseId) {
                if (!$periodo->horariosPeriodo()->where('horario_base_id', $horarioBaseId)->exists()) {
                    $horarioBase = Horario::find($horarioBaseId);
                    
                    HorarioPeriodo::create([
                        'periodo_id' => $periodo->id,
                        'horario_base_id' => $horarioBaseId,
                        'nombre' => $this->generarNombreHorarioPeriodo($horarioBase, $periodo),
                        'tipo' => $horarioBase->tipo,
                        'dias' => $horarioBase->dias,
                        'hora_inicio' => $horarioBase->hora_inicio,
                        'hora_fin' => $horarioBase->hora_fin,
                        'activo' => true
                    ]);
                }
            }
        });

        Log::info('Horarios agregados a periodo', [
            'periodo_id' => $periodo->id,
            'estado_actual' => $periodo->estado,
            'horarios_count' => count($request->horarios_base_ids)
        ]);

        return redirect()->route('coordinador.periodos.show', $periodo)
            ->with('success', 'Horarios agregados exitosamente al periodo.');

    } catch (\Exception $e) {
        Log::error('Error agregando horarios: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al agregar horarios: ' . $e->getMessage());
    }
}
```

### Método: eliminarHorarioPeriodo(Periodo $periodo, HorarioPeriodo $horarioPeriodo)
Elimina un horario específico de un periodo.

```php
public function eliminarHorarioPeriodo(Periodo $periodo, HorarioPeriodo $horarioPeriodo)
{
    if ($periodo->estaFinalizado() || $periodo->estaCancelado()) {
        return redirect()->back()
            ->with('error', 'No se pueden eliminar horarios en periodos finalizados o cancelados');
    }

    if ($horarioPeriodo->periodo_id !== $periodo->id) {
        return redirect()->back()
            ->with('error', 'El horario no pertenece a este periodo.');
    }

    if ($horarioPeriodo->grupos()->exists()) {
        return redirect()->back()
            ->with('error', 'No se puede eliminar el horario porque tiene grupos asignados.');
    }

    try {
        $nombreHorario = $horarioPeriodo->nombre;
        $horarioPeriodo->delete();

        Log::info('Horario periodo eliminado', [
            'periodo_id' => $periodo->id,
            'estado_actual' => $periodo->estado,
            'horario_periodo_id' => $horarioPeriodo->id,
            'nombre' => $nombreHorario
        ]);

        return redirect()->route('coordinador.periodos.show', $periodo)
            ->with('success', "Horario \"{$nombreHorario}\" eliminado del periodo.");

    } catch (\Exception $e) {
        Log::error('Error eliminando horario periodo: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al eliminar el horario.');
    }
}
```

### Método: toggleHorarioPeriodo(Periodo $periodo, HorarioPeriodo $horarioPeriodo)
Activa o desactiva un horario en un periodo.

```php
public function toggleHorarioPeriodo(Periodo $periodo, HorarioPeriodo $horarioPeriodo)
{
    if ($periodo->estaFinalizado() || $periodo->estaCancelado()) {
        return redirect()->back()
            ->with('error', 'No se pueden modificar horarios en periodos finalizados o cancelados');
    }

    if ($horarioPeriodo->periodo_id !== $periodo->id) {
        return redirect()->back()
            ->with('error', 'El horario no pertenece a este periodo.');
    }

    try {
        $nuevoEstado = !$horarioPeriodo->activo;
        $horarioPeriodo->update(['activo' => $nuevoEstado]);

        $estadoTexto = $nuevoEstado ? 'activado' : 'desactivado';
        
        Log::info('Horario periodo toggled', [
            'periodo_id' => $periodo->id,
            'estado_actual' => $periodo->estado,
            'horario_periodo_id' => $horarioPeriodo->id,
            'nuevo_estado' => $estadoTexto
        ]);

        return redirect()->route('coordinador.periodos.show', $periodo)
            ->with('success', "Horario {$estadoTexto} correctamente.");

    } catch (\Exception $e) {
        Log::error('Error cambiando estado horario periodo: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al cambiar estado del horario.');
    }
}
```

## Métodos de Gestión de Estado

### Método: activarPreregistros(Periodo $periodo)
Activa los preregistros para un periodo con validaciones completas.

```php
public function activarPreregistros(Periodo $periodo)
{
    if (!$periodo->puedeCambiarA('preregistros_activos')) {
        return redirect()->back()->with('error', 
            "No se puede activar preregistros desde '{$periodo->estado_legible}'"
        );
    }

    if ($periodo->horariosPeriodo()->where('activo', true)->count() === 0) {
        return redirect()->back()
            ->with('error', 'No se pueden activar preregistros: Debe tener al menos un horario activo en el período.');
    }

    if (!\App\Models\Aula::where('disponible', true)->exists()) {
        return redirect()->back()
            ->with('error', 'No se pueden activar preregistros: Debe crear aulas disponibles en el sistema.');
    }

    if (!\App\Models\Profesor::exists()) {
        return redirect()->back()
            ->with('error', 'No se pueden activar preregistros: Debe tener profesores registrados en el sistema.');
    }

    try {
        $periodo->update(['estado' => 'preregistros_activos']);
        
        Log::info('Pre-registros activados', [
            'periodo_id' => $periodo->id,
            'estado_anterior' => $periodo->getOriginal('estado'),
            'horarios_activos' => $periodo->horariosPeriodo()->where('activo', true)->count(),
            'aulas_disponibles' => \App\Models\Aula::where('disponible', true)->count(),
            'total_profesores' => \App\Models\Profesor::count()
        ]);
        
        return redirect()->route('coordinador.periodos.index')
            ->with('success', 'Pre-registros activados. Los estudiantes ya pueden registrarse.');

    } catch (\Exception $e) {
        Log::error('Error activando pre-registros: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al activar pre-registros.');
    }
}
```

### Método: cerrarPreregistros(Periodo $periodo)
Cierra los preregistros de un periodo.

```php
public function cerrarPreregistros(Periodo $periodo)
{
    if (!$periodo->puedeCambiarA('preregistros_cerrados')) {
        return redirect()->back()->with('error', 
            "No se puede cerrar preregistros desde '{$periodo->estado_legible}'"
        );
    }

    if ($periodo->preregistros()->count() === 0) {
        return redirect()->back()
            ->with('warning', '¿Está seguro? El período no tiene preregistros. ¿Desea continuar?')
            ->with('confirmar_cierre', true);
    }

    try {
        $periodo->update(['estado' => 'preregistros_cerrados']);
        
        Log::info('Pre-registros cerrados', [
            'periodo_id' => $periodo->id,
            'estado_anterior' => $periodo->getOriginal('estado'),
            'total_preregistros' => $periodo->preregistros()->count()
        ]);
        
        return redirect()->route('coordinador.periodos.index')
            ->with('success', 'Pre-registros cerrados. Se procederá con la asignación de grupos.');

    } catch (\Exception $e) {
        Log::error('Error cerrando pre-registros: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al cerrar pre-registros.');
    }
}
```

### Método: iniciarPeriodo(Periodo $periodo)
Inicia un periodo académico con automatización de estados.

```php
public function iniciarPeriodo(Periodo $periodo)
{
    if (!$periodo->puedeCambiarA('en_curso')) {
        return redirect()->back()->with('error', 
            "No se puede iniciar periodo desde '{$periodo->estado_legible}'"
        );
    }

    try {
        DB::transaction(function () use ($periodo) {
            $estadoAnterior = $periodo->estado;
            $periodo->update(['estado' => 'en_curso']);
            
            $preregistrosActualizados = $periodo->preregistros()
                ->where('estado', 'asignado')
                ->whereNotNull('grupo_asignado_id')
                ->update(['estado' => 'cursando']);
            
            $gruposActualizados = $periodo->grupos()
                ->where('estado', 'activo')
                ->update(['estado' => 'en_curso']);

            Log::info('Periodo iniciado con automatización', [
                'periodo_id' => $periodo->id,
                'estado_anterior' => $estadoAnterior,
                'preregistros_actualizados' => $preregistrosActualizados,
                'grupos_actualizados' => $gruposActualizados
            ]);
        });

        return redirect()->route('coordinador.periodos.index')
            ->with('success', 'Periodo iniciado exitosamente. Los estudiantes asignados ahora están en curso.');

    } catch (\Exception $e) {
        Log::error('Error iniciando periodo: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al iniciar periodo.');
    }
}
```

### Método: finalizarPeriodo(Periodo $periodo)
Finaliza un periodo académico.

```php
public function finalizarPeriodo(Periodo $periodo)
{
    if (!$periodo->puedeCambiarA('finalizado')) {
        return redirect()->back()->with('error', 
            "No se puede finalizar periodo desde '{$periodo->estado_legible}'"
        );
    }

    try {
        DB::transaction(function () use ($periodo) {
            $estadoAnterior = $periodo->estado;
            $periodo->update(['estado' => 'finalizado']);
            
            if ($estadoAnterior === 'en_curso') {
                $periodo->grupos()->where('estado', 'en_curso')->update(['estado' => 'finalizado']);
                $periodo->preregistros()->where('estado', 'cursando')->update(['estado' => 'finalizado']);
            }
        });

        Log::info('Periodo finalizado', [
            'periodo_id' => $periodo->id,
            'estado_anterior' => $periodo->getOriginal('estado')
        ]);
        
        return redirect()->route('coordinador.periodos.index')
            ->with('success', 'Periodo finalizado exitosamente.');

    } catch (\Exception $e) {
        Log::error('Error finalizando periodo: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al finalizar periodo.');
    }
}
```

### Método: cancelarPeriodo(Periodo $periodo)
Cancela un periodo académico.

```php
public function cancelarPeriodo(Periodo $periodo)
{
    if (!$periodo->puedeCambiarA('cancelado')) {
        return redirect()->back()->with('error', 
            "No se puede cancelar periodo desde '{$periodo->estado_legible}'"
        );
    }

    try {
        DB::transaction(function () use ($periodo) {
            $estadoAnterior = $periodo->estado;
            $periodo->update(['estado' => 'cancelado']);
            
            if (in_array($estadoAnterior, ['en_curso', 'preregistros_abiertos', 'preregistros_cerrados'])) {
                $periodo->grupos()->update(['estado' => 'cancelado']);
                $periodo->preregistros()
                    ->whereIn('estado', ['asignado', 'cursando'])
                    ->update([
                        'estado' => 'pendiente',
                        'grupo_asignado_id' => null
                    ]);
            }
        });

        Log::info('Periodo cancelado', [
            'periodo_id' => $periodo->id,
            'estado_anterior' => $periodo->getOriginal('estado')
        ]);
        
        return redirect()->route('coordinador.periodos.index')
            ->with('success', 'Periodo cancelado exitosamente.');

    } catch (\Exception $e) {
        Log::error('Error cancelando periodo: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al cancelar periodo.');
    }
}
```

### Método: cambiarEstado(Request $request, Periodo $periodo)
Método genérico para cambiar el estado de un periodo con validaciones completas.

```php
public function cambiarEstado(Request $request, Periodo $periodo)
{
    $request->validate([
        'nuevo_estado' => 'required|in:configuracion,preregistros_activos,preregistros_cerrados,en_curso,finalizado,cancelado'
    ]);

    $nuevoEstado = $request->nuevo_estado;

    if (!$periodo->puedeCambiarA($nuevoEstado)) {
        return redirect()->back()->with('error', 
            "No se puede cambiar de '{$periodo->estado_legible}' a '" . 
            ($periodo::ESTADOS[$nuevoEstado] ?? $nuevoEstado) . "'"
        );
    }

    if ($nuevoEstado === 'preregistros_activos') {
        if ($periodo->horariosPeriodo()->where('activo', true)->count() === 0) {
            return redirect()->back()
                ->with('error', 'No se pueden activar preregistros: Debe tener al menos un horario activo en el período.');
        }

        if (!\App\Models\Aula::where('disponible', true)->exists()) {
            return redirect()->back()
                ->with('error', 'No se pueden activar preregistros: Debe crear aulas disponibles en el sistema.');
        }

        if (!\App\Models\Profesor::exists()) {
            return redirect()->back()
                ->with('error', 'No se pueden activar preregistros: Debe tener profesores registrados en el sistema.');
        }
    }

    try {
        DB::transaction(function () use ($periodo, $nuevoEstado) {
            $estadoActual = $periodo->estado;
            
            switch ($nuevoEstado) {
                case 'en_curso':
                    $periodo->preregistros()
                        ->where('estado', 'asignado')
                        ->whereNotNull('grupo_asignado_id')
                        ->update(['estado' => 'cursando']);
                    
                    $periodo->grupos()
                        ->where('estado', 'activo')
                        ->update(['estado' => 'en_curso']);
                    break;
                    
                case 'finalizado':
                    if ($estadoActual === 'en_curso') {
                        $periodo->grupos()->where('estado', 'en_curso')->update(['estado' => 'finalizado']);
                        $periodo->preregistros()->where('estado', 'cursando')->update(['estado' => 'finalizado']);
                    }
                    break;
                    
                case 'cancelado':
                    $periodo->grupos()->update(['estado' => 'cancelado']);
                    $periodo->preregistros()
                        ->whereIn('estado', ['asignado', 'cursando'])
                        ->update([
                            'estado' => 'pendiente',
                            'grupo_asignado_id' => null
                        ]);
                    break;
            }

            $periodo->update(['estado' => $nuevoEstado]);
        });

        Log::info("Estado cambiado", [
            'periodo_id' => $periodo->id,
            'estado_anterior' => $periodo->getOriginal('estado'),
            'nuevo_estado' => $nuevoEstado
        ]);

        return redirect()->route('coordinador.periodos.index')
            ->with('success', "Estado actualizado: " . $periodo->estado_legible);

    } catch (\Exception $e) {
        Log::error('Error cambiando estado: ' . $e->getMessage());
        return redirect()->back()->with('error', 'Error al cambiar estado: ' . $e->getMessage());
    }
}
```

## Métodos Privados

### generarNombreHorarioPeriodo(Horario $horario, Periodo $periodo)
Genera un nombre único para el horario del periodo.

```php
private function generarNombreHorarioPeriodo(Horario $horario, Periodo $periodo)
{
    $base = str_replace('Plantilla', '', $horario->nombre);
    $base = trim($base);
    
    return "{$base} - {$periodo->nombre_periodo}";
}
```

### obtenerEstadisticasDetalladas($periodoId)
Calcula estadísticas detalladas para un periodo específico.

```php
private function obtenerEstadisticasDetalladas($periodoId)
{
    return [
        'total_grupos' => DB::table('grupos')->where('periodo_id', $periodoId)->count(),
        'grupos_activos' => DB::table('grupos')->where('periodo_id', $periodoId)->where('estado', 'activo')->count(),
        'total_preregistros' => DB::table('preregistros')->where('periodo_id', $periodoId)->count(),
        'preregistros_pagados' => DB::table('preregistros')->where('periodo_id', $periodoId)->where('pago_estado', 'pagado')->count(),
        'preregistros_con_prorroga' => DB::table('preregistros')->where('periodo_id', $periodoId)->where('pago_estado', 'prorroga')->count(),
        'estudiantes_activos' => DB::table('preregistros')->where('periodo_id', $periodoId)->where('estado', 'cursando')->count(),
        'horarios_activos' => DB::table('horarios_periodo')->where('periodo_id', $periodoId)->where('activo', true)->count(),
    ];
}
```

## Características de Seguridad y Validaciones

### Validaciones Implementadas
- **Estado coherente**: Solo permite operaciones en estados válidos
- **Dependencias del sistema**: Verifica horarios, aulas y profesores antes de activar preregistros
- **Integridad referencial**: Previene eliminación de horarios con grupos asignados
- **Transiciones de estado**: Valida cambios de estado según reglas de negocio

### Manejo de Transacciones
- Operaciones críticas usan transacciones de base de datos
- Garantiza consistencia en cambios de estado y actualizaciones masivas

### Logging y Auditoría
- Registro detallado de todas las operaciones importantes
- Información contextual para debugging y auditoría

Este controlador proporciona una gestión completa de periodos académicos con validaciones robustas que garantizan la integridad del sistema y previenen operaciones inválidas según el estado actual del periodo.
