# Periodo Académico - Modelo Principal

## Descripción General
El modelo Periodo representa el núcleo central del sistema GLOTTY después de la reestructuración arquitectónica del Sprint 4. Constituye la entidad raíz de la que dependen todas las operaciones académicas, estableciendo una jerarquía clara y coherente en la gestión institucional.

## Estructura del Modelo

### Definición de Campos
```php
protected $fillable = [
    'nombre_periodo',
    'fecha_inicio', 
    'fecha_fin',
    'estado'
];

protected $casts = [
    'fecha_inicio' => 'date',
    'fecha_fin' => 'date',
];
```

El modelo gestiona los periodos académicos mediante cuatro campos fundamentales. El campo 'nombre_periodo' identifica de manera única cada ciclo escolar, mientras que 'fecha_inicio' y 'fecha_fin' definen el rango temporal del periodo. El campo 'estado' controla el flujo de vida del periodo académico, permitiendo transiciones entre diferentes fases operativas.

### Estados del Periodo Académico
```php
const ESTADOS = [
    'configuracion' => 'En Configuración',
    'preregistros_activos' => 'Preregistros Abiertos',
    'preregistros_cerrados' => 'Preregistros Cerrados',
    'en_curso' => 'En Curso',
    'finalizado' => 'Finalizado',
    'cancelado' => 'Cancelado'
];
```

El sistema implementa seis estados que representan el ciclo de vida completo de un periodo académico. Cada estado tiene implicaciones específicas sobre las operaciones permitidas, creando un flujo de trabajo estructurado desde la configuración inicial hasta la finalización o cancelación.

## Relaciones Arquitectónicas

### Relación con Horarios
```php
public function horariosPeriodo() { 
    return $this->hasMany(HorarioPeriodo::class); 
}

public function horariosBase() { 
    return $this->belongsToMany(Horario::class, 'horarios_periodo', 'periodo_id', 'horario_base_id')
                ->withPivot('activo')
                ->withTimestamps(); 
}
```

La relación con horarios se implementa mediante el modelo pivote HorarioPeriodo, que permite la reutilización de horarios base entre diferentes periodos académicos. Esta arquitectura muchos-a-muchos optimiza la gestión temporal al permitir que un mismo horario pueda ser asociado a múltiples periodos con estados específicos.

### Relación con Grupos y Preregistros
```php
public function grupos() { 
    return $this->hasMany(Grupo::class); 
}

public function preregistros() { 
    return $this->hasMany(Preregistro::class); 
}
```

Los grupos y preregistros establecen relaciones de dependencia directa con el periodo académico. Esta estructura garantiza que toda la actividad académica quede contextualizada dentro de un periodo específico, proporcionando coherencia organizacional y facilitando la gestión histórica de datos.

## Sistema de Consultas por Estado

### Scopes de Filtrado
El modelo incluye scopes especializados para filtrar periodos según su estado operativo:

- scopeActivo: Periodos actualmente en curso
- scopeConPreRegistrosActivos: Periodos que aceptan preregistros
- scopeConPreRegistrosCerrados: Periodos con preregistros finalizados
- scopeFinalizados: Periodos académicamente completados
- scopeCancelados: Periodos cancelados antes de su finalización

Estos scopes facilitan la consulta de periodos según necesidades específicas del negocio, optimizando las operaciones de búsqueda y filtrado.

## Gestión del Ciclo de Vida

### Métodos de Verificación de Estado
El modelo proporciona métodos booleanos para verificar el estado actual del periodo:

- estaEnConfiguracion: Verifica si el periodo está en fase de configuración inicial
- preregistrosAbiertos: Determina si el periodo acepta preregistros de estudiantes
- preregistrosCerrados: Indica si el periodo ha cerrado los preregistros
- estaEnCurso: Confirma si el periodo está activo académicamente
- estaFinalizado: Verifica si el periodo ha concluido
- estaCancelado: Determina si el periodo fue cancelado

### Transiciones de Estado Controladas
```php
public function puedeCambiarA($nuevoEstado)
{
    if (in_array($this->estado, ['finalizado', 'cancelado'])) {
        return false;
    }

    if (in_array($nuevoEstado, ['finalizado', 'cancelado'])) {
        return true;
    }

    $transicionesPermitidas = [
        'configuracion' => ['preregistros_activos', 'preregistros_cerrados', 'en_curso'],
        'preregistros_activos' => ['configuracion', 'preregistros_cerrados', 'en_curso'],
        'preregistros_cerrados' => ['preregistros_activos', 'en_curso', 'configuracion'],
        'en_curso' => ['preregistros_cerrados', 'preregistros_activos', 'configuracion'],
    ];

    return in_array($nuevoEstado, $transicionesPermitidas[$this->estado] ?? []);
}
```

El método puedeCambiarA implementa un sistema de transiciones de estado controlado que permite flexibilidad operativa manteniendo la integridad del flujo de trabajo. Los estados finalizados y cancelados son terminales, mientras que los estados activos permiten transiciones bidireccionales según las necesidades académicas.

## Métodos de Utilidad

### Cálculo de Duración
```php
public function getDiasDuracionAttribute(): int
{
    return $this->fecha_inicio->diffInDays($this->fecha_fin);
}
```

Calcula automáticamente la duración en días del periodo académico, proporcionando información valiosa para la planificación y gestión temporal.

### Validaciones de Eliminación
```php
public function puedeEliminarse()
{
    return $this->estaEnConfiguracion() && 
        $this->grupos()->count() === 0 &&
        $this->preregistros()->count() === 0;
}
```

Restringe la eliminación de periodos a aquellos que se encuentran en estado de configuración y no tienen grupos o preregistros asociados, protegiendo la integridad de los datos académicos.

### Control de Operaciones
Los métodos permiteGestionHorarios y permitePreRegistrosEstudiantes controlan el acceso a funcionalidades críticas del sistema basándose en el estado actual del periodo, asegurando que las operaciones solo se realicen en contextos apropiados.

## Impacto Arquitectónico

La reestructuración centrada en el modelo Periodo transformó significativamente la arquitectura del sistema GLOTTY. Al establecer el periodo académico como entidad raíz, se logró:

- Coherencia organizacional mediante relaciones jerárquicas claras
- Contextualización de todas las operaciones académicas
- Reutilización eficiente de recursos como horarios entre periodos
- Gestión histórica estructurada de datos académicos
- Escalabilidad para futuras expansiones del sistema

Esta arquitectura garantiza que cada acción dentro del sistema quede referenciada a un periodo académico específico, proporcionando trazabilidad completa y facilitando la administración institucional.
