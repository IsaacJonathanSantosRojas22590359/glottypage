```markdown
# PeriodoController - Controlador Principal del Sistema

## Descripción General
El PeriodoController constituye el núcleo central del sistema GLOTTY después de la reestructuración arquitectónica del Sprint 4. Como controlador principal, gestiona el ciclo de vida completo de los periodos académicos y coordina todas las operaciones relacionadas con la gestión institucional.

## Gestión del Ciclo de Vida del Periodo

### Listado y Consulta de Periodos
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

El método index implementa un sistema de consulta avanzado que incluye conteos relacionados y filtrado por estado. Proporciona una visión completa de todos los periodos académicos con métricas clave para la toma de decisiones.

### Creación de Nuevos Periodos
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

La creación de periodos implementa transacciones de base de datos para garantizar la consistencia y registra eventos detallados para auditoría. Las validaciones aseguran que los periodos se creen con fechas futuras y nombres únicos.

### Visualización Detallada del Periodo
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

El método show proporciona una vista completa del periodo que incluye horarios contextualizados, estadísticas detalladas y horarios base disponibles para agregar, facilitando la gestión integral del periodo académico.

## Gestión de Horarios Contextualizados

### Agregar Horarios desde Plantillas
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

Este método implementa la estrategia de reutilización de horarios base mediante la creación de instancias contextualizadas en HorarioPeriodo. La lógica evita duplicados y genera nombres descriptivos que incluyen referencia al periodo.

### Gestión de Horarios del Periodo
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

La eliminación de horarios incluye validaciones de integridad referencial que previenen la eliminación de horarios con grupos asignados, protegiendo la consistencia de los datos académicos.

## Transiciones de Estado Controladas

### Activación de Preregistros con Validaciones
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

La activación de preregistros implementa validaciones exhaustivas que garantizan la viabilidad operativa del periodo, verificando la existencia de horarios activos, aulas disponibles y profesores registrados.

### Inicio del Periodo con Automatización
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

El inicio del periodo automatiza la transición de estados de preregistros y grupos, actualizando masivamente a los estudiantes asignados al estado "cursando" y a los grupos al estado "en_curso".

### Sistema Unificado de Cambio de Estado
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

El método cambiarEstado proporciona una interfaz unificada para todas las transiciones de estado, incorporando lógica específica para cada transición y manteniendo la consistencia de datos mediante transacciones.

## Utilidades y Métodos de Soporte

### Generación de Nombres de Horarios Contextualizados
```php
private function generarNombreHorarioPeriodo(Horario $horario, Periodo $periodo)
{
    $base = str_replace('Plantilla', '', $horario->nombre);
    $base = trim($base);
    
    return "{$base} - {$periodo->nombre_periodo}";
}
```

Este método genera nombres descriptivos para los horarios contextualizados que combinan información del horario base con referencia al periodo académico.

### Obtención de Estadísticas Detalladas
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

Proporciona métricas completas del periodo para dashboards y reportes, incluyendo información sobre grupos, preregistros, estados de pago y horarios activos.

## Impacto Arquitectónico

El PeriodoController representa la materialización de la reestructuración arquitectónica centrada en periodos académicos:

- Centralización de todas las operaciones académicas alrededor del periodo
- Coordinación automática entre estados del periodo y entidades relacionadas
- Validaciones de viabilidad operativa antes de transiciones críticas
- Gestión transaccional para mantener consistencia de datos
- Sistema de logging comprehensivo para auditoría y troubleshooting
- Reutilización eficiente de recursos mediante horarios contextualizados

Esta arquitectura garantiza que cada operación del sistema quede referenciada a un periodo académico específico, proporcionando trazabilidad completa y facilitando la gestión institucional mediante relaciones jerárquicas claras.
```