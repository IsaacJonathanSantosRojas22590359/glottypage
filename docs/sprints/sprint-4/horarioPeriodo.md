# HorarioPeriodo - Modelo Pivote de Reestructuración

## Descripción General
El modelo HorarioPeriodo constituye el componente fundamental de la reestructuración arquitectónica del Sprint 4. Funciona como modelo pivote que establece la relación muchos-a-muchos entre periodos académicos y horarios base, permitiendo la reutilización contextual de horarios mientras mantiene instantáneas específicas por periodo.

## Estructura de Datos

### Tabla y Campos Principales
```php
protected $table = 'horarios_periodo';

protected $fillable = [
    'periodo_id',
    'horario_base_id', 
    'nombre',
    'tipo',
    'dias',
    'hora_inicio',
    'hora_fin',
    'activo'
];
```

La tabla horarios_periodo actúa como tabla pivote extendida que almacena tanto las relaciones como copias instantáneas de los datos del horario base. Esta estrategia permite que los horarios base puedan ser reutilizados en múltiples periodos académicos mientras se preserva la información específica de cada instancia.

### Sistema de Tipado de Datos
```php
protected $casts = [
    'dias' => 'array',
    'hora_inicio' => 'datetime:H:i:s',
    'hora_fin' => 'datetime:H:i:s',
    'activo' => 'boolean',
];
```

El modelo implementa un sistema de conversión de tipos avanzado que transforma automáticamente el campo 'dias' de formato JSON a array nativo de PHP, facilitando la manipulación de datos. Los campos de tiempo se convierten a objetos DateTime con formato específico, mientras que el campo 'activo' se maneja como booleano para control de estado.

## Relaciones del Modelo

### Relación con Periodo Académico
```php
public function periodo()
{
    return $this->belongsTo(Periodo::class);
}
```

Cada instancia de HorarioPeriodo pertenece a un periodo académico específico, estableciendo la dependencia contextual fundamental de la reestructuración. Esta relación garantiza que todos los horarios operen dentro del contexto de un periodo académico definido.

### Relación con Horario Base
```php
public function horarioBase()
{
    return $this->belongsTo(Horario::class, 'horario_base_id');
}
```

La relación con el horario base permite mantener el vínculo con la definición original del horario, facilitando la trazabilidad y posibles actualizaciones mientras se preserva la instantánea específica del periodo.

### Relación con Grupos
```php
public function grupos()
{
    return $this->hasMany(Grupo::class, 'horario_periodo_id');
}
```

Los grupos establecen relación directa con las instancias específicas de HorarioPeriodo, asegurando que cada grupo quede asociado a un horario contextualizado dentro de un periodo académico particular.

## Sistema de Instantáneas

### Estrategia de Copia de Datos
El modelo implementa una estrategia de instantáneas que copia los campos críticos del horario base ('nombre', 'tipo', 'dias', 'hora_inicio', 'hora_fin') en cada instancia de HorarioPeriodo. Este enfoque proporciona:

- Inmutabilidad de datos históricos una vez asignados a grupos
- Independencia de modificaciones futuras en horarios base
- Preservación del contexto temporal específico de cada periodo
- Capacidad de auditoría y trazabilidad completa

### Métodos de Formateo de Datos

#### Formateo de Días de la Semana
```php
public function getDiasFormateadosAttribute()
{
    if (!$this->dias) return '';

    $diasMap = [
        'Lunes' => 'Lun',
        'Martes' => 'Mar', 
        'Miércoles' => 'Mié',
        'Jueves' => 'Jue',
        'Viernes' => 'Vie',
        'Sábado' => 'Sáb'
    ];

    return collect($this->dias)->map(function ($dia) use ($diasMap) {
        return $diasMap[$dia] ?? $dia;
    })->implode(', ');
}
```

Este método transforma el array de días almacenado en formato completo a abreviaturas estandarizadas, optimizando la presentación visual en interfaces de usuario sin perder la información original.

#### Representación Completa del Horario
```php
public function getHorarioCompletoAttribute()
{
    return "{$this->dias_formateados} {$this->hora_inicio->format('H:i')} - {$this->hora_fin->format('H:i')}";
}
```

Proporciona una representación legible del horario que combina días formateados con el rango horario, facilitando la identificación rápida en listados y selecciones.

## Métodos de Utilidad

### Cálculo de Duración
```php
public function getDuracionAttribute()
{
    return $this->hora_inicio->diffInHours($this->hora_fin);
}
```

Calcula automáticamente la duración en horas del bloque horario, proporcionando información valiosa para la planificación académica y validaciones de disponibilidad.

### Filtrado por Estado Activo
```php
public function scopeActivos($query)
{
    return $query->where('activo', true);
}
```

Permite filtrar exclusivamente los horarios activos dentro de un periodo, optimizando las consultas para operaciones de asignación y gestión en curso.

### Descripción Contextual
```php
public function getDescripcionCompletaAttribute()
{
    return $this->nombre . ' - ' . $this->periodo->nombre_periodo;
}
```

Genera una descripción que contextualiza el horario dentro de su periodo académico, mejorando la identificación en interfaces administrativas.

## Impacto Arquitectónico

La implementación del modelo HorarioPeriodo resolvió desafíos críticos de la arquitectura anterior:

- Separación de preocupaciones entre definición base y contexto operativo
- Preservación de datos históricos contra modificaciones futuras
- Reutilización eficiente de horarios base entre múltiples periodos
- Contextualización completa de todas las operaciones académicas
- Capacidad de auditoría y trazabilidad temporal

Esta arquitectura permite que un horario base definido una vez pueda ser reutilizado en múltiples periodos académicos con estados específicos, mientras se mantienen instantáneas inmutables para cada uso contextual. El modelo actúa como puente entre la definición estática de horarios y la ejecución dinámica dentro de periodos académicos específicos, estableciendo las bases para un sistema escalable y mantenible.
