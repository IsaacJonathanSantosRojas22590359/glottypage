# Documentación del Modelo Periodo

## Descripción General
El modelo `Periodo` representa los periodos académicos en el sistema de gestión educativa. Gestiona la información temporal de los ciclos escolares, incluyendo fechas de inicio y fin, estados del periodo, y todas las relaciones asociadas con horarios, grupos y preregistros.

## Estructura del Modelo

### Namespace y Definición de Clase
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Periodo extends Model
{
    use HasFactory;
```

## Propiedades del Modelo

### Atributos Mass Assignable
```php
protected $fillable = [
    'nombre_periodo',
    'fecha_inicio', 
    'fecha_fin',
    'estado'
];
```

### Casts de Atributos
```php
protected $casts = [
    'fecha_inicio' => 'date',
    'fecha_fin' => 'date',
];
```

## Constantes del Sistema

### Estados del Periodo
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

## Relaciones

### Relación con HorariosPeriodo
```php
public function horariosPeriodo() 
{ 
    return $this->hasMany(HorarioPeriodo::class); 
}
```

### Relación con Horarios Base
```php
public function horariosBase() 
{ 
    return $this->belongsToMany(Horario::class, 'horarios_periodo', 'periodo_id', 'horario_base_id')
                ->withPivot('activo')
                ->withTimestamps(); 
}
```

### Relación con Grupos
```php
public function grupos() 
{ 
    return $this->hasMany(Grupo::class); 
}
```

### Relación con Preregistros
```php
public function preregistros() 
{ 
    return $this->hasMany(Preregistro::class); 
}
```

## Scopes de Consulta

### Scope de Periodos Activos
```php
public function scopeActivo($query) 
{ 
    return $query->where('estado', 'en_curso'); 
}
```

### Scope de Periodos con Preregistros Activos
```php
public function scopeConPreRegistrosActivos($query) 
{ 
    return $query->where('estado', 'preregistros_activos'); 
}
```

### Scope de Periodos con Preregistros Cerrados
```php
public function scopeConPreRegistrosCerrados($query) 
{ 
    return $query->where('estado', 'preregistros_cerrados'); 
}
```

### Scope de Periodos Finalizados
```php
public function scopeFinalizados($query) 
{ 
    return $query->where('estado', 'finalizado'); 
}
```

### Scope de Periodos Cancelados
```php
public function scopeCancelados($query) 
{ 
    return $query->where('estado', 'cancelado'); 
}
```

## Métodos de Estado

### Verificar Estado de Configuración
```php
public function estaEnConfiguracion() 
{ 
    return $this->estado === 'configuracion'; 
}
```

### Verificar Preregistros Abiertos
```php
public function preregistrosAbiertos() 
{ 
    return $this->estado === 'preregistros_activos'; 
}
```

### Verificar Preregistros Cerrados
```php
public function preregistrosCerrados() 
{ 
    return $this->estado === 'preregistros_cerrados'; 
}
```

### Verificar Estado en Curso
```php
public function estaEnCurso() 
{ 
    return $this->estado === 'en_curso'; 
}
```

### Verificar Estado Finalizado
```php
public function estaFinalizado() 
{ 
    return $this->estado === 'finalizado'; 
}
```

### Verificar Estado Cancelado
```php
public function estaCancelado() 
{ 
    return $this->estado === 'cancelado'; 
}
```

## Métodos de Utilidad

### Calcular Días de Duración
```php
public function getDiasDuracionAttribute(): int
{
    return $this->fecha_inicio->diffInDays($this->fecha_fin);
}
```

### Verificar si Puede Eliminarse
```php
public function puedeEliminarse()
{
    return $this->estaEnConfiguracion() && 
        $this->grupos()->count() === 0 &&
        $this->preregistros()->count() === 0;
}
```

### Validar Cambio de Estado
```php
public function puedeCambiarA($nuevoEstado)
{
    // Estados finales: no se puede cambiar desde ellos
    if (in_array($this->estado, ['finalizado', 'cancelado'])) {
        return false;
    }

    // Estados que no pueden ser destino desde cualquier estado
    if (in_array($nuevoEstado, ['finalizado', 'cancelado'])) {
        return true; // Permitir finalizar/cancelar desde cualquier estado activo
    }

    // Transiciones principales permitidas (con flexibilidad)
    $transicionesPermitidas = [
        'configuracion' => ['preregistros_activos', 'preregistros_cerrados', 'en_curso'],
        'preregistros_activos' => ['configuracion', 'preregistros_cerrados', 'en_curso'],
        'preregistros_cerrados' => ['preregistros_activos', 'en_curso', 'configuracion'],
        'en_curso' => ['preregistros_cerrados', 'preregistros_activos', 'configuracion'],
    ];

    return in_array($nuevoEstado, $transicionesPermitidas[$this->estado] ?? []);
}
```

## Accessors (Getters Computados)

### Estado Legible
```php
public function getEstadoLegibleAttribute()
{
    return self::ESTADOS[$this->estado] ?? $this->estado;
}
```

## Métodos de Validación de Negocio

### Verificar si Acepta Preregistros
```php
public function aceptaPreRegistros()
{
    return $this->preregistrosAbiertos();
}
```

### Verificar si Permite Gestión de Horarios
```php
public function permiteGestionHorarios()
{
    return !$this->estaFinalizado() && !$this->estaCancelado();
}
```

### Verificar si Permite Preregistros de Estudiantes
```php
public function permitePreRegistrosEstudiantes()
{
    return $this->preregistrosAbiertos();
}
```

## Flujos de Trabajo y Validaciones

### Estados del Periodo y sus Significados

**configuracion**: Periodo en preparación, permite configuración de horarios y grupos
**preregistros_activos**: Periodo aceptando preregistros de estudiantes
**preregistros_cerrados**: Preregistros cerrados, permite asignación de estudiantes a grupos
**en_curso**: Periodo activo con clases en desarrollo
**finalizado**: Periodo completado, solo consulta
**cancelado**: Periodo cancelado, no permite operaciones

### Validaciones de Transición de Estados

El método `puedeCambiarA()` implementa las siguientes reglas:

- Estados finales (`finalizado`, `cancelado`) no permiten cambios
- Se puede finalizar o cancelar desde cualquier estado activo
- Transiciones flexibles entre estados de configuración y preregistros
- No se permite retroceder desde estados avanzados a configuración básica

### Restricciones Operativas

**Para Eliminación**:
- Solo periodos en estado "configuracion"
- Sin grupos asociados
- Sin preregistros existentes

**Para Gestión de Horarios**:
- No permitido en estados "finalizado" o "cancelado"

**Para Preregistros de Estudiantes**:
- Solo permitido en estado "preregistros_activos"

## Ejemplos de Uso

### Creación de Periodo
```php
$periodo = Periodo::create([
    'nombre_periodo' => 'Primavera 2024',
    'fecha_inicio' => '2024-01-15',
    'fecha_fin' => '2024-05-20',
    'estado' => 'configuracion'
]);
```

### Verificación de Estado
```php
if ($periodo->preregistrosAbiertos()) {
    // Permitir preregistro de estudiantes
}

if ($periodo->permiteGestionHorarios()) {
    // Permitir gestión de horarios
}
```

### Cambio de Estado
```php
if ($periodo->puedeCambiarA('preregistros_activos')) {
    $periodo->update(['estado' => 'preregistros_activos']);
}
```

Este modelo proporciona una gestión completa de periodos académicos con validaciones robustas que garantizan la integridad de las transiciones de estado y previenen operaciones inválidas según el estado actual del periodo.