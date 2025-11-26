# Documentación del Modelo Horario

## Descripción General
El modelo `Horario` representa los horarios base del sistema académico, definiendo las configuraciones de días y horas disponibles para la asignación de clases. Gestiona tanto horarios semanales como sabatinos, proporcionando funcionalidades para validación, formateo y verificación de uso en periodos activos.

## Estructura del Modelo

### Namespace y Definición de Clase
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Horario extends Model
{
    use HasFactory;
```

## Propiedades del Modelo

### Atributos Mass Assignable
```php
protected $fillable = [
    'nombre',
    'tipo', 
    'dias',
    'hora_inicio',
    'hora_fin',
    'activo'
];
```

### Casts de Atributos
```php
protected $casts = [
    'dias' => 'array',
    'hora_inicio' => 'datetime:H:i',
    'hora_fin' => 'datetime:H:i',
    'activo' => 'boolean'
];
```

**Explicación de Casts**:
- `dias`: Convierte el JSON de la base de datos a array de PHP
- `hora_inicio` y `hora_fin`: Formatean las horas como objetos Carbon para manipulación temporal
- `activo`: Convierte el valor a booleano

## Relaciones

### Relación con HorariosPeriodo
```php
public function horariosPeriodo()
{
    return $this->hasMany(HorarioPeriodo::class, 'horario_base_id');
}
```
**Descripción**: Establece una relación uno-a-muchos con el modelo `HorarioPeriodo`, donde un horario base puede tener múltiples instancias en diferentes periodos académicos.

### Relación con Periodos Activos
```php
public function periodosActivos()
{
    return $this->belongsToMany(Periodo::class, 'horarios_periodo', 'horario_base_id', 'periodo_id')
                ->wherePivot('activo', true)
                ->withTimestamps();
}
```
**Descripción**: Define una relación muchos-a-muchos con el modelo `Periodo` a través de la tabla pivote `horarios_periodo`, filtrando solo los periodos activos.

## Scopes de Consulta

### Scope de Horarios Activos
```php
public function scopeActivos($query)
{
    return $query->where('activo', true);
}
```
**Uso**: Filtra solo los horarios que están marcados como activos en el sistema.

### Scope de Horarios Semanales
```php
public function scopeSemanales($query)
{
    return $query->where('tipo', 'semanal');
}
```
**Uso**: Filtra horarios de tipo semanal (Lunes a Viernes).

### Scope de Horarios Sabatinos
```php
public function scopeSabatinos($query)
{
    return $query->where('tipo', 'sabatino');
}
```
**Uso**: Filtra horarios de tipo sabatino (exclusivamente Sábados).

## Métodos de Utilidad

### Verificar Estado Activo
```php
public function estaActivo()
{
    return $this->activo;
}
```
**Propósito**: Verifica si el horario está activo en el sistema.
**Retorna**: `true` si está activo, `false` si está inactivo.

### Obtener Días Formateados
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
        'Sábado' => 'Sáb',
        'Domingo' => 'Dom'
    ];

    return collect($this->dias)->map(function ($dia) use ($diasMap) {
        return $diasMap[$dia] ?? $dia;
    })->implode(', ');
}
```
**Propósito**: Convierte el array de días a una cadena formateada con abreviaturas.
**Ejemplo**: Convierte `['Lunes', 'Miércoles', 'Viernes']` a `"Lun, Mié, Vie"`

### Obtener Horario Completo
```php
public function getHorarioCompletoAttribute()
{
    return "{$this->dias_formateados} {$this->hora_inicio->format('H:i')} - {$this->hora_fin->format('H:i')}";
}
```
**Propósito**: Genera una representación completa y legible del horario.
**Ejemplo**: `"Lun, Mié, Vie 08:00 - 10:00"`

### Calcular Duración
```php
public function getDuracionAttribute()
{
    return $this->hora_inicio->diffInHours($this->hora_fin);
}
```
**Propósito**: Calcula la duración total del horario en horas.
**Retorna**: Número entero representando las horas de duración.

## Métodos de Validación de Negocio

### Verificar si se Puede Eliminar
```php
public function sePuedeEliminar()
{
    // No se puede eliminar si está siendo usado en algún periodo
    return $this->horariosPeriodo()->count() === 0;
}
```
**Propósito**: Determina si el horario puede ser eliminado del sistema.
**Retorna**: `true` si no tiene horarios de periodo asociados, `false` si está en uso.

### Verificar Uso en Periodos Activos
```php
public function enUsoEnPeriodosActivos()
{
    return $this->horariosPeriodo()
                ->whereHas('periodo', function($query) {
                    $query->where('estado', '!=', 'finalizado');
                })
                ->exists();
}
```
**Propósito**: Verifica si el horario está siendo utilizado en periodos que no están finalizados.
**Retorna**: `true` si está en uso en periodos activos, `false` en caso contrario.

## Flujos de Trabajo y Casos de Uso

### 1. Creación de Nuevo Horario
```php
$horario = Horario::create([
    'nombre' => 'Matutino Lunes-Miércoles',
    'tipo' => 'semanal',
    'dias' => ['Lunes', 'Miércoles'],
    'hora_inicio' => '08:00',
    'hora_fin' => '10:00',
    'activo' => true
]);
```

### 2. Búsqueda de Horarios Disponibles
```php
// Obtener horarios semanales activos
$horariosSemanales = Horario::activos()->semanales()->get();

// Obtener horarios sabatinos
$horariosSabatinos = Horario::sabatinos()->get();
```

### 3. Validación para Eliminación
```php
$horario = Horario::find(1);

if ($horario->sePuedeEliminar()) {
    $horario->delete();
    return response()->json(['message' => 'Horario eliminado correctamente']);
} else {
    return response()->json(['error' => 'No se puede eliminar el horario: está en uso'], 400);
}
```

### 4. Formateo para Interfaz de Usuario
```php
$horario = Horario::find(1);
echo $horario->horario_completo; // "Lun, Mié 08:00 - 10:00"
echo $horario->duracion; // 2 (horas)
```

### 5. Verificación de Uso en Periodos
```php
if ($horario->enUsoEnPeriodosActivos()) {
    return redirect()->back()->with('error', 'No se puede modificar el horario: está en uso en periodos activos');
}
```

## Reglas de Validación de Negocio

### Para Horarios Semanales
- No pueden incluir el día "Sábado" en el array de días
- Deben incluir al menos un día entre Lunes y Viernes

### Para Horarios Sabatinos
- Deben incluir exclusivamente el día "Sábado"
- No pueden incluir días de la semana (Lunes a Viernes)

### Validaciones de Tiempo
- `hora_fin` debe ser posterior a `hora_inicio`
- La duración mínima debe ser de 1 hora
- No debe haber superposición de horarios con el mismo conjunto de días

## Mejores Prácticas de Uso

### Para Consultas de Horarios
```php
// ✅ Correcto - Usar scopes definidos
$horarios = Horario::activos()->semanales()->get();

// ❌ Incorrecto - Condiciones manuales
$horarios = Horario::where('activo', true)->where('tipo', 'semanal')->get();
```

### Para Validaciones de Negocio
```php
// ✅ Correcto - Usar métodos del modelo
if ($horario->enUsoEnPeriodosActivos()) {
    // Manejar restricción
}

// ❌ Incorrecto - Implementar lógica en el controlador
```

### Para Formateo de Datos
```php
// ✅ Correcto - Usar accessors
$horarioCompleto = $horario->horario_completo;

// ❌ Incorrecto - Formatear manualmente en vistas
```

## Integración con el Sistema

### En Controladores
El modelo Horario se utiliza principalmente en:
- `HorarioController`: Para operaciones CRUD básicas
- `HorarioPeriodoController`: Para gestión de horarios por periodo
- `GrupoController`: Para asignación de horarios a grupos

### En Vistas
Los accessors como `dias_formateados` y `horario_completo` facilitan la presentación de datos sin necesidad de lógica adicional en las vistas.

Este modelo proporciona una base robusta para la gestión de horarios académicos, con validaciones de integridad y métodos utilitarios que simplifican el desarrollo de funcionalidades relacionadas con la asignación de tiempos en el sistema educativo.