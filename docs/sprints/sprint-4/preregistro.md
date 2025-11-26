# Preregistro - Gestión de Solicitudes Académicas

## Descripción General
El modelo Preregistro gestiona el proceso completo de solicitudes de inscripción de estudiantes dentro del sistema GLOTTY. Después de la reestructuración del Sprint 4, este modelo establece una relación obligatoria con el periodo académico, contextualizando todas las solicitudes dentro de un ciclo escolar específico y sincronizando su estado con el flujo del periodo.

## Estructura de Datos

### Campos Principales y Relaciones
```php
protected $fillable = [
    'usuario_id',
    'periodo_id',
    'nivel_solicitado',
    'horario_preferido_id',
    'semestre_carrera',
    'grupo_asignado_id',
    'estado',
    'pago_estado'
];
```

El modelo captura toda la información necesaria para el proceso de preregistro, incluyendo la identificación del estudiante, el periodo académico objetivo, el nivel solicitado, preferencias de horario, y el estado tanto académico como financiero de la solicitud.

### Sistema de Estados

#### Estados Académicos del Preregistro
```php
const ESTADOS = [
    'pendiente' => 'Pendiente de Asignación',
    'asignado' => 'Asignado a Grupo',
    'cursando' => 'Cursando',
    'finalizado' => 'Finalizado',
    'cancelado' => 'Cancelado'
];
```

El flujo académico sigue una progresión lógica desde la solicitud inicial hasta la finalización o cancelación. Cada estado representa una fase específica en el ciclo de vida del preregistro, con transiciones controladas que aseguran la integridad del proceso.

#### Estados de Pago con Prórroga
```php
const PAGO_ESTADOS = [
    'pendiente' => 'Pendiente de Pago',
    'prorroga' => 'En Prórroga',
    'pagado' => 'Pagado',
    'rechazado' => 'Pago Rechazado'
];
```

El sistema de pagos incorpora un estado de prórroga que permite flexibilidad en los plazos de pago mientras mantiene el control sobre el proceso financiero. Esta adición responde a necesidades operativas reales sin comprometer la gestión financiera.

## Relaciones Contextualizadas

### Relación con Periodo Académico
```php
public function periodo()
{
    return $this->belongsTo(Periodo::class);
}
```

La relación obligatoria con el modelo Periodo constituye el cambio arquitectónico más significativo. Cada preregistro queda vinculado a un periodo académico específico, permitiendo la sincronización automática de estados y la contextualización de todas las operaciones.

### Relación con Horario Preferido
```php
public function horarioPreferido()
{
    return $this->belongsTo(HorarioPeriodo::class, 'horario_preferido_id');
}
```

La preferencia de horario se establece mediante relación con HorarioPeriodo, asegurando que las opciones presentadas al estudiante estén contextualizadas dentro del periodo académico activo.

### Relación con Grupo Asignado
```php
public function grupoAsignado()
{
    return $this->belongsTo(Grupo::class, 'grupo_asignado_id');
}
```

Una vez asignado, el preregistro establece relación con el grupo específico donde el estudiante cursará, completando el proceso de inscripción.

## Sistema de Consultas Especializadas

### Scopes para Gestión Operativa
El modelo incluye scopes optimizados para las operaciones más comunes:

- pendientes: Filtra solicitudes pendientes de asignación
- pagados: Identifica preregistros con pago completado
- puedenAsignarse: Combina estado pendiente con estados de pago válidos (incluyendo prórroga)
- activos: Agrupa todos los estados académicos activos
- conProrroga: Específico para gestionar casos con extensión de pago

Estos scopes facilitan la implementación de lógica de negocio compleja mediante consultas simples y legibles.

## Lógica de Negocio Avanzada

### Validaciones de Asignación
```php
public function puedeSerAsignado()
{
    return $this->estado === 'pendiente' && 
           in_array($this->pago_estado, ['pagado', 'prorroga']) &&
           $this->periodo->aceptaPreRegistros();
}
```

El método valida múltiples condiciones para determinar si un preregistro puede ser asignado a un grupo: estado académico pendiente, pago en estado válido (incluyendo prórroga), y periodo académico que acepte preregistros activamente.

### Sincronización con Periodo Académico
```php
public function sincronizarConPeriodo()
{
    if ($this->periodo->estaEnCurso() && $this->estaListoParaCursar()) {
        $this->update(['estado' => 'cursando']);
    }
    
    if ($this->periodo->estaFinalizado() && $this->estado === 'cursando') {
        $this->update(['estado' => 'finalizado']);
    }
}
```

Este método implementa la sincronización automática entre el estado del preregistro y el estado del periodo académico. Cuando un periodo inicia, los preregistros asignados y con pago en orden cambian automáticamente a estado "cursando". Al finalizar el periodo, los preregistros en curso se marcan como finalizados.

### Gestión de Vencimientos y Prórrogas
```php
public function estaVencido()
{
    return $this->pago_estado === 'pendiente' && 
           $this->created_at->diffInDays(now()) > 7;
}
```

El sistema diferencia claramente entre preregistros pendientes de pago (sujetos a vencimiento) y aquellos en prórroga (exentos de vencimiento automático), proporcionando flexibilidad operativa mientras mantiene controles financieros.

## Transiciones de Estado Controladas

### Matriz de Transiciones Permitidas
```php
public function puedeCambiarA($nuevoEstado)
{
    $transicionesPermitidas = [
        'pendiente' => ['asignado', 'cancelado'],
        'asignado' => ['cursando', 'cancelado'],
        'cursando' => ['finalizado'],
        'finalizado' => [],
        'cancelado' => []
    ];

    return in_array($nuevoEstado, $transicionesPermitidas[$this->estado]);
}
```

El modelo implementa un sistema de transiciones de estado estricto que previene cambios inválidos en el flujo académico. Los estados finales (finalizado, cancelado) son terminales, mientras que los estados intermedios permiten solo transiciones lógicas y controladas.

## Utilidades de Presentación

### Formateo de Datos para Interfaz
Los accesores getNivelFormateadoAttribute, getEstadoFormateadoAttribute y getPagoEstadoFormateadoAttribute transforman los valores técnicos almacenados en la base de datos en representaciones legibles para el usuario final, mejorando la experiencia en interfaces administrativas.

### Sistema de Colores para Estados de Pago
```php
public function getColorPagoAttribute()
{
    return match($this->pago_estado) {
        'pendiente' => 'yellow',
        'prorroga' => 'blue',
        'pagado' => 'green',
        'rechazado' => 'red',
        default => 'gray'
    };
}
```

Proporciona una representación visual consistente de los estados de pago a través de colores estandarizados, facilitando la identificación rápida de situaciones que requieren atención en dashboards administrativos.

## Impacto de la Reestructuración

La integración obligatoria con el modelo Periodo transformó significativamente la gestión de preregistros:

- Contextualización automática dentro de ciclos académicos específicos
- Sincronización bidireccional entre estados de preregistro y periodo
- Validaciones basadas en el estado operativo del periodo académico
- Capacidad de generar reportes históricos por periodo
- Prevención de operaciones fuera de contexto temporal

Esta arquitectura asegura que cada preregistro opere dentro del contexto temporal correcto, proporcionando coherencia organizacional y facilitando la gestión académica institucional.