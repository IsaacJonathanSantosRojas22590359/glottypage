```markdown
# Grupo - Gestión de Grupos Académicos

## Descripción General
El modelo Grupo representa la entidad central para la organización de estudiantes en el sistema GLOTTY después de la reestructuración del Sprint 4. Cada grupo queda contextualizado dentro de un periodo académico específico y establece relaciones con horarios, aulas y profesores, formando una estructura académica completa y coherente.

## Estructura de Datos

### Campos Principales y Configuración
```php
protected $fillable = [
    'nivel_ingles',
    'letra_grupo',
    'periodo_id',
    'horario_periodo_id',
    'aula_id',
    'profesor_id',
    'capacidad_maxima',
    'estudiantes_inscritos',
    'estado'
];

protected $casts = [
    'nivel_ingles' => 'integer',
    'capacidad_maxima' => 'integer',
    'estudiantes_inscritos' => 'integer'
];
```

El modelo gestiona la configuración completa de un grupo académico, incluyendo su nivel de inglés, identificación única mediante letra, relaciones contextuales con periodo y horario, asignación de recursos físicos y humanos, y control de capacidad e inscripciones.

### Sistema de Estados del Grupo
```php
const ESTADOS = [
    'planificado' => 'Planificado',
    'con_profesor' => 'Con Profesor', 
    'con_aula' => 'Con Aula',
    'activo' => 'Activo',
    'cancelado' => 'Cancelado'
];
```

Los grupos siguen un flujo de estados progresivo que representa su nivel de configuración. Desde la planificación inicial hasta la activación completa, cada estado indica qué recursos han sido asignados y qué operaciones están permitidas.

### Niveles de Inglés y Letras de Grupo
```php
const NIVELES = [
    1 => 'Nivel 1',
    2 => 'Nivel 2', 
    3 => 'Nivel 3',
    4 => 'Nivel 4',
    5 => 'Nivel 5'
];

const LETRAS_GRUPO = ['A', 'B', 'C', 'D', 'E', 'F'];
```

El sistema soporta cinco niveles de inglés y utiliza letras consecutivas para identificar grupos dentro del mismo nivel y periodo, garantizando nombres únicos y significativos.

## Relaciones Contextualizadas

### Relación con Periodo Académico
```php
public function periodo() 
{ 
    return $this->belongsTo(Periodo::class); 
}
```

Cada grupo pertenece obligatoriamente a un periodo académico específico, estableciendo el contexto temporal fundamental para todas sus operaciones. Esta relación permite la gestión coherente de grupos dentro de ciclos escolares definidos.

### Relación con Horario Contextualizado
```php
public function horario() 
{ 
    return $this->belongsTo(HorarioPeriodo::class, 'horario_periodo_id'); 
}
```

La relación con HorarioPeriodo asegura que el horario del grupo esté contextualizado dentro del periodo académico, utilizando el modelo pivote que mantiene instantáneas específicas por periodo.

### Relaciones con Recursos
```php
public function aula() 
{ 
    return $this->belongsTo(Aula::class, 'aula_id'); 
}

public function profesor() 
{ 
    return $this->belongsTo(Profesor::class, 'profesor_id', 'id_profesor');
}
```

Las relaciones con Aula y Profesor gestionan la asignación de recursos físicos y humanos al grupo, con validaciones integradas para evitar conflictos y garantizar la viabilidad operativa.

### Gestión de Estudiantes
```php
public function preregistros() 
{ 
    return $this->hasMany(Preregistro::class, 'grupo_asignado_id'); 
}

public function estudiantesActivos() 
{ 
    return $this->preregistros()->whereIn('estado', ['asignado', 'cursando']); 
}
```

El grupo mantiene relación con los preregistros de estudiantes asignados, proporcionando métodos especializados para acceder específicamente a los estudiantes activos.

## Sistema de Consultas Especializadas

### Scopes para Gestión Operativa
El modelo incluye scopes optimizados para las operaciones más comunes:

- activos: Grupos en estado operativo completo
- planificados: Grupos en fase de configuración inicial
- porNivel: Filtrado por nivel de inglés específico
- porPeriodo: Grupos dentro de un periodo académico
- conCapacidad: Grupos con disponibilidad para nuevos estudiantes

### Detección de Conflictos de Horario
```php
public function scopeSolapados($query, $horarioId, $aulaId = null, $profesorId = null)
{
    return $query->where('horario_periodo_id', $horarioId)
        ->where(function ($q) use ($aulaId, $profesorId) {
            if ($aulaId) {
                $q->where('aula_id', $aulaId);
            }
            if ($profesorId) {
                $q->orWhere('profesor_id', $profesorId);
            }
        })
        ->whereNotIn('estado', ['cancelado']);
}
```

Este scope identifica grupos que podrían generar conflictos por solapamiento de horarios, considerando tanto aulas como profesores, y excluyendo grupos cancelados del análisis.

## Métodos de Utilidad y Presentación

### Generación de Nombres y Formatos
```php
public function getNombreCompletoAttribute()
{
    return "{$this->nivel_ingles}-{$this->letra_grupo}";
}

public function getNivelFormateadoAttribute()
{
    return self::NIVELES[$this->nivel_ingles] ?? "Nivel {$this->nivel_ingles}";
}
```

Los accesores proporcionan representaciones formateadas del grupo, incluyendo nombres completos y niveles legibles, optimizando la presentación en interfaces de usuario.

### Gestión de Capacidad y Ocupación
```php
public function getCapacidadDisponibleAttribute()
{
    return $this->capacidad_maxima - $this->estudiantes_inscritos;
}

public function getPorcentajeOcupacionAttribute()
{
    if ($this->capacidad_maxima === 0) return 0;
    return round(($this->estudiantes_inscritos / $this->capacidad_maxima) * 100, 2);
}
```

Estos métodos calculan automáticamente la capacidad disponible y el porcentaje de ocupación del grupo, facilitando la toma de decisiones en procesos de asignación.

## Lógica de Negocio Avanzada

### Validaciones de Asignación de Recursos

#### Validación de Aula
```php
public function validarAsignacionAula($aulaId)
{
    $aula = Aula::find($aulaId);
    if (!$aula) {
        throw new \Exception('El aula seleccionada no existe.');
    }

    if (!$aula->disponible) {
        throw new \Exception("El aula {$aula->nombre_completo} no está disponible.");
    }

    if (!$aula->soportaCapacidad($this->capacidad_maxima)) {
        throw new \Exception("El aula {$aula->nombre_completo} tiene capacidad para {$aula->capacidad} estudiantes, pero el grupo requiere {$this->capacidad_maxima}.");
    }

    if (!$aula->estaDisponibleEnHorario($this->horario_periodo_id)) {
        throw new \Exception("El aula {$aula->nombre_completo} ya está ocupada en este horario.");
    }

    return true;
}
```

El método realiza validaciones exhaustivas para la asignación de aulas, verificando existencia, disponibilidad global, capacidad suficiente y disponibilidad en el horario específico.

#### Validación de Profesor
```php
public function validarAsignacionProfesor($profesorId)
{
    $profesor = Profesor::find($profesorId);
    if (!$profesor) {
        throw new \Exception('El profesor seleccionado no existe.');
    }

    $conflictos = self::where('profesor_id', $profesorId)
        ->where('horario_periodo_id', $this->horario_periodo_id)
        ->where('id', '!=', $this->id)
        ->whereNotIn('estado', ['cancelado'])
        ->exists();

    if ($conflictos) {
        throw new \Exception("El profesor {$profesor->nombre_profesor} ya tiene un grupo asignado en este horario.");
    }

    return true;
}
```

Valida la asignación de profesores verificando existencia y detectando conflictos de horario con otros grupos activos.

### Gestión de Estudiantes

#### Asignación de Estudiantes
```php
public function asignarEstudiante($preregistroId)
{
    if (!$this->tieneCapacidad()) {
        throw new \Exception('El grupo no tiene capacidad disponible.');
    }

    $preregistro = Preregistro::find($preregistroId);
    if (!$preregistro || !$preregistro->puedeSerAsignado()) {
        throw new \Exception('El estudiante no puede ser asignado a un grupo.');
    }

    if ($preregistro->grupo_asignado_id && $preregistro->grupo_asignado_id != $this->id) {
        throw new \Exception('El estudiante ya está asignado a otro grupo.');
    }

    \DB::transaction(function () use ($preregistro) {
        $preregistro->update([
            'grupo_asignado_id' => $this->id,
            'estado' => 'asignado'
        ]);

        $this->increment('estudiantes_inscritos');
    });

    return true;
}
```

El método gestiona la asignación de estudiantes con validaciones completas y transacciones atómicas, asegurando la consistencia de datos.

#### Remoción de Estudiantes
```php
public function removerEstudiante($preregistroId)
{
    $preregistro = Preregistro::find($preregistroId);
    if (!$preregistro || $preregistro->grupo_asignado_id !== $this->id) {
        throw new \Exception('El estudiante no está asignado a este grupo.');
    }

    \DB::transaction(function () use ($preregistro) {
        $preregistro->update([
            'grupo_asignado_id' => null,
            'estado' => 'pendiente'
        ]);

        if ($this->estudiantes_inscritos > 0) {
            $this->decrement('estudiantes_inscritos');
        }
    });

    return true;
}
```

Permite la remoción segura de estudiantes mediante transacciones que mantienen la consistencia entre el grupo y el preregistro.

## Sistema de Validación Automática

### Hook de Validación Pre-Guardado
```php
protected static function boot()
{
    parent::boot();

    static::saving(function ($grupo) {
        if ($grupo->aula_id && !$grupo->aulaSoportaCapacidad()) {
            throw new \Exception("El aula no tiene suficiente capacidad para el grupo.");
        }

        if ($grupo->tieneConflictosHorario()) {
            $conflictos = $grupo->obtenerConflictos();
            $mensaje = "Conflicto de horario detectado: ";
            
            foreach ($conflictos as $conflicto) {
                if ($conflicto->aula_id == $grupo->aula_id) {
                    $mensaje .= "Aula ocupada por grupo {$conflicto->nombre_completo}. ";
                }
                if ($conflicto->profesor_id == $grupo->profesor_id) {
                    $mensaje .= "Profesor asignado a grupo {$conflicto->nombre_completo}. ";
                }
            }
            
            throw new \Exception($mensaje);
        }
    });
}
```

El hook automático valida la capacidad del aula y detecta conflictos de horario antes de guardar cualquier cambio en el grupo, previniendo inconsistencias en la configuración académica.

## Gestión de Letras Disponibles

### Asignación Automática de Letras
```php
public static function letrasDisponibles($nivel, $periodoId)
{
    $letrasOcupadas = self::where('nivel_ingles', $nivel)
        ->where('periodo_id', $periodoId)
        ->pluck('letra_grupo')
        ->toArray();

    return array_diff(self::LETRAS_GRUPO, $letrasOcupadas);
}
```

Este método identifica las letras disponibles para nuevos grupos dentro de un nivel y periodo específicos, facilitando la creación de grupos con identificadores únicos.

## Impacto de la Reestructuración

La integración obligatoria con el modelo Periodo transformó significativamente la gestión de grupos:

- Contextualización completa dentro de ciclos académicos específicos
- Coordinación automática con el estado del periodo académico
- Validaciones basadas en el contexto temporal del periodo
- Capacidad de generar reportes históricos por periodo
- Prevención de operaciones fuera de contexto temporal

Esta arquitectura asegura que cada grupo opere dentro del contexto temporal correcto, proporcionando coherencia organizacional y facilitando la gestión académica institucional a través de relaciones jerárquicas claras y validadas.
```