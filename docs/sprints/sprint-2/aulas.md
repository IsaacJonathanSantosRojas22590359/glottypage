# Documentación del Modelo Aula

## Descripción General
El modelo `Aula` representa los espacios físicos académicos dentro del sistema de gestión educativa. Gestiona la información de aulas, laboratorios, salas de cómputo y otros espacios de enseñanza, proporcionando funcionalidades avanzadas para la gestión de disponibilidad, capacidad y asignaciones.

## Estructura del Modelo

### Namespace y Definición de Clase
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Aula extends Model
{
    use HasFactory;
```

## Propiedades del Modelo

### Atributos Mass Assignable
```php
protected $fillable = [
    'nombre',
    'edificio', 
    'capacidad',
    'tipo',
    'equipamiento',
    'disponible'
];
```

### Casts de Atributos
```php
protected $casts = [
    'disponible' => 'boolean',
    'capacidad' => 'integer'
];
```

### Constantes de Tipos de Aula
```php
const TIPOS_AULA = [
    'regular' => 'Aula Regular',
    'laboratorio' => 'Laboratorio',
    'computo' => 'Sala de Cómputo', 
    'audiovisual' => 'Aula Audiovisual',
];
```

## Relaciones

### Relación con Grupos
```php
public function grupos()
{
    return $this->hasMany(Grupo::class, 'aula_id');
}
```
**Descripción**: Establece una relación uno-a-muchos con el modelo `Grupo`, donde un aula puede tener múltiples grupos asignados.

## Scopes de Consulta

### Scope de Aulas Disponibles
```php
public function scopeDisponibles($query)
{
    return $query->where('disponible', true);
}
```
**Uso**: Filtra solo las aulas que están marcadas como disponibles.

### Scope por Tipo de Aula
```php
public function scopePorTipo($query, $tipo)
{
    return $query->where('tipo', $tipo);
}
```
**Uso**: Filtra aulas por tipo específico (regular, laboratorio, etc.).

### Scope por Edificio
```php
public function scopePorEdificio($query, $edificio)
{
    return $query->where('edificio', $edificio);
}
```
**Uso**: Filtra aulas por edificio específico.

## Métodos de Disponibilidad y Validación

### Verificar Disponibilidad en Horario Específico
```php
public function estaDisponibleEnHorario($horarioPeriodoId)
{
    // Si el aula no está disponible globalmente
    if (!$this->disponible) {
        return false;
    }

    // Verificar si ya está ocupada en ese horario
    $ocupada = $this->grupos()
        ->where('horario_periodo_id', $horarioPeriodoId)
        ->whereNotIn('estado', ['cancelado'])
        ->exists();

    return !$ocupada;
}
```
**Propósito**: Determina si el aula está disponible para un horario específico.
**Retorna**: `true` si está disponible, `false` si está ocupada o no disponible.

### Obtener Horarios Ocupados
```php
public function obtenerHorariosOcupados()
{
    return $this->grupos()
        ->whereNotIn('estado', ['cancelado'])
        ->with('horario')
        ->get()
        ->pluck('horario');
}
```
**Propósito**: Recupera todos los horarios en los que el aula está actualmente ocupada.
**Retorna**: Colección de modelos `Horario` ocupados.

### Verificar Capacidad Soportada
```php
public function soportaCapacidad($capacidadRequerida)
{
    return $this->capacidad >= $capacidadRequerida;
}
```
**Propósito**: Valida si el aula puede acomodar un número específico de estudiantes.
**Retorna**: `true` si la capacidad es suficiente, `false` en caso contrario.

## Accessors (Getters Computados)

### Nombre Completo del Aula
```php
public function getNombreCompletoAttribute()
{
    return "{$this->edificio}-{$this->nombre}";
}
```
**Ejemplo**: Si `edificio` es "A" y `nombre` es "101", retorna "A-101".

### Tipo de Aula Formateado
```php
public function getTipoFormateadoAttribute()
{
    return self::TIPOS_AULA[$this->tipo] ?? $this->tipo;
}
```
**Propósito**: Convierte el código del tipo de aula a su descripción legible.
**Ejemplo**: Convierte "laboratorio" a "Laboratorio".

### Información Resumida del Aula
```php
public function getInfoResumidaAttribute()
{
    $disponibilidad = $this->disponible ? 'Disponible' : 'No disponible';
    return "{$this->nombre_completo} - {$this->tipo_formateado} ({$this->capacidad} pers.) - {$disponibilidad}";
}
```
**Ejemplo**: "A-101 - Laboratorio (30 pers.) - Disponible"

## Métodos Estáticos

### Obtener Aulas Disponibles para Horario
```php
public static function disponiblesParaHorario($horarioPeriodoId, $capacidadRequerida = null)
{
    $query = self::disponibles()
        ->whereDoesntHave('grupos', function ($q) use ($horarioPeriodoId) {
            $q->where('horario_periodo_id', $horarioPeriodoId)
              ->whereNotIn('estado', ['cancelado']);
        });

    if ($capacidadRequerida) {
        $query->where('capacidad', '>=', $capacidadRequerida);
    }

    return $query->get();
}
```
**Propósito**: Busca aulas disponibles para un horario específico, opcionalmente filtrando por capacidad mínima.
**Parámetros**:
- `$horarioPeriodoId`: ID del horario a verificar
- `$capacidadRequerida`: Capacidad mínima requerida (opcional)

### Obtener Estadísticas del Sistema
```php
public static function obtenerEstadisticas()
{
    return [
        'total' => self::count(),
        'disponibles' => self::where('disponible', true)->count(),
        'por_tipo' => self::groupBy('tipo')
                        ->selectRaw('tipo, count(*) as total')
                        ->pluck('total', 'tipo'),
        'por_edificio' => self::groupBy('edificio')
                            ->selectRaw('edificio, count(*) as total')
                            ->pluck('total', 'edificio')
    ];
}
```
**Propósito**: Genera estadísticas agregadas sobre las aulas del sistema.
**Retorna**: Array con:
- `total`: Número total de aulas
- `disponibles`: Número de aulas disponibles
- `por_tipo`: Conteo de aulas por tipo
- `por_edificio`: Conteo de aulas por edificio

## Flujos de Trabajo y Casos de Uso

### 1. Asignación de Grupos a Aulas
```php
// Verificar disponibilidad y capacidad
$aula = Aula::find(1);
if ($aula->estaDisponibleEnHorario($horarioId) && $aula->soportaCapacidad(30)) {
    // Asignar grupo al aula
    $grupo->aula_id = $aula->id;
    $grupo->save();
}
```

### 2. Búsqueda de Aulas Disponibles
```php
// Encontrar aulas disponibles para un horario con capacidad mínima
$aulasDisponibles = Aula::disponiblesParaHorario($horarioId, 25);
```

### 3. Gestión de Disponibilidad
```php
// Obtener horarios ocupados para mostrar conflictos
$horariosOcupados = $aula->obtenerHorariosOcupados();
```

### 4. Reportes y Estadísticas
```php
// Generar reporte general de aulas
$estadisticas = Aula::obtenerEstadisticas();
```

## Validaciones y Restricciones de Negocio

### Reglas de Validación Implícitas
- La capacidad debe ser un número entero positivo
- El tipo debe ser uno de los valores definidos en `TIPOS_AULA`
- La disponibilidad se maneja como booleano

### Integridad Referencial
- Las aulas con grupos asignados no deben eliminarse directamente
- Los cambios de disponibilidad deben validarse contra grupos activos

## Mejores Prácticas de Uso

### Para Consultas de Disponibilidad
```php
// Correcto - Usar métodos del modelo
$aula->estaDisponibleEnHorario($horarioId);

// Incorrecto - Implementar lógica manualmente
```

### Para Filtrados
```php
// Correcto - Usar scopes
Aula::disponibles()->porTipo('laboratorio')->get();

// Incorrecto - Condiciones manuales
```

Este modelo proporciona una base sólida para la gestión de espacios académicos, con funcionalidades completas para disponibilidad, capacidad y relaciones con otros componentes del sistema.