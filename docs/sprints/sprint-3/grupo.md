# Documentación del Modelo Grupo

## Descripción General
El modelo `Grupo` representa los grupos académicos en el sistema de gestión educativa. Gestiona la información de grupos de inglés, incluyendo su nivel, letra, periodo, horario, aula asignada, profesor y estudiantes inscritos. Implementa un sistema completo de estados y validaciones de negocio para garantizar la integridad de las asignaciones.

## Estructura del Modelo

### Namespace y Definición de Clase
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

class Grupo extends Model
{
    use HasFactory;
```

## Propiedades del Modelo

### Atributos Mass Assignable
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
```

### Casts de Atributos
```php
protected $casts = [
    'nivel_ingles' => 'integer',
    'capacidad_maxima' => 'integer',
    'estudiantes_inscritos' => 'integer'
];
```

## Constantes del Sistema

### Estados del Grupo
```php
const ESTADOS = [
    'planificado' => 'Planificado',
    'con_profesor' => 'Con Profesor', 
    'con_aula' => 'Con Aula',
    'activo' => 'Activo',
    'cancelado' => 'Cancelado'
];
```

### Niveles de Inglés
```php
const NIVELES = [
    1 => 'Nivel 1',
    2 => 'Nivel 2', 
    3 => 'Nivel 3',
    4 => 'Nivel 4',
    5 => 'Nivel 5'
];
```

### Letras de Grupo Disponibles
```php
const LETRAS_GRUPO = ['A', 'B', 'C', 'D', 'E', 'F'];
```

## Relaciones

### Relación con Periodo
```php
public function periodo() 
{ 
    return $this->belongsTo(Periodo::class); 
}
```

### Relación con HorarioPeriodo
```php
public function horario() 
{ 
    return $this->belongsTo(HorarioPeriodo::class, 'horario_periodo_id'); 
}
```

### Relación con Aula
```php
public function aula() 
{ 
    return $this->belongsTo(Aula::class, 'aula_id'); 
}
```

### Relación con Profesor
```php
public function profesor() 
{ 
    return $this->belongsTo(Profesor::class, 'profesor_id', 'id_profesor');
}
```

### Relación con Preregistros
```php
public function preregistros() 
{ 
    return $this->hasMany(Preregistro::class, 'grupo_asignado_id'); 
}
```

### Relación con Estudiantes Activos
```php
public function estudiantesActivos() 
{ 
    return $this->preregistros()->whereIn('estado', ['asignado', 'cursando']); 
}
```

## Scopes de Consulta

### Scope de Grupos Activos
```php
public function scopeActivos($query) 
{ 
    return $query->where('estado', 'activo'); 
}
```

### Scope de Grupos Planificados
```php
public function scopePlanificados($query) 
{ 
    return $query->where('estado', 'planificado'); 
}
```

### Scope por Nivel
```php
public function scopePorNivel($query, $nivel) 
{ 
    return $query->where('nivel_ingles', $nivel); 
}
```

### Scope por Periodo
```php
public function scopePorPeriodo($query, $periodoId) 
{ 
    return $query->where('periodo_id', $periodoId); 
}
```

### Scope con Capacidad Disponible
```php
public function scopeConCapacidad($query) 
{ 
    return $query->whereRaw('estudiantes_inscritos < capacidad_maxima'); 
}
```

### Scope de Grupos Solapados
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

## Accessors (Getters Computados)

### Nombre Completo del Grupo
```php
public function getNombreCompletoAttribute()
{
    return "{$this->nivel_ingles}-{$this->letra_grupo}";
}
```

### Nivel Formateado
```php
public function getNivelFormateadoAttribute()
{
    return self::NIVELES[$this->nivel_ingles] ?? "Nivel {$this->nivel_ingles}";
}
```

### Estado Formateado
```php
public function getEstadoFormateadoAttribute()
{
    return self::ESTADOS[$this->estado] ?? $this->estado;
}
```

### Capacidad Disponible
```php
public function getCapacidadDisponibleAttribute()
{
    return $this->capacidad_maxima - $this->estudiantes_inscritos;
}
```

### Porcentaje de Ocupación
```php
public function getPorcentajeOcupacionAttribute()
{
    if ($this->capacidad_maxima === 0) return 0;
    return round(($this->estudiantes_inscritos / $this->capacidad_maxima) * 100, 2);
}
```

### Clase CSS para Estado
```php
public function getClaseEstadoAttribute()
{
    return match($this->estado) {
        'planificado' => 'bg-yellow-100 text-yellow-800',
        'con_profesor' => 'bg-blue-100 text-blue-800', 
        'con_aula' => 'bg-purple-100 text-purple-800',
        'activo' => 'bg-green-100 text-green-800',
        'cancelado' => 'bg-red-100 text-red-800',
        default => 'bg-gray-100 text-gray-800'
    };
}
```

### Estado Legible
```php
public function getEstadoLegibleAttribute()
{
    return self::ESTADOS[$this->estado] ?? $this->estado;
}
```

## Métodos de Negocio

### Verificar Capacidad Disponible
```php
public function tieneCapacidad()
{
    return $this->estudiantes_inscritos < $this->capacidad_maxima;
}
```

### Verificar si Puede ser Activo
```php
public function puedeSerActivo()
{
    return $this->profesor_id && $this->aula_id && $this->tieneCapacidad();
}
```

### Verificar si Puede ser Cancelado
```php
public function puedeSerCancelado()
{
    return $this->estudiantes_inscritos === 0 && $this->estado !== 'cancelado';
}
```

### Verificar si Puede ser Eliminado
```php
public function puedeEliminarse()
{
    return $this->estudiantes_inscritos === 0 && $this->estado === 'planificado';
}
```

### Verificar Capacidad del Aula
```php
public function aulaSoportaCapacidad()
{
    if (!$this->aula) return false;
    return $this->aula->capacidad >= $this->capacidad_maxima;
}
```

### Verificar Conflictos de Horario
```php
public function tieneConflictosHorario()
{
    return self::solapados($this->horario_periodo_id, $this->aula_id, $this->profesor_id)
        ->where('id', '!=', $this->id)
        ->exists();
}
```

### Obtener Conflictos Detallados
```php
public function obtenerConflictos()
{
    return self::solapados($this->horario_periodo_id, $this->aula_id, $this->profesor_id)
        ->where('id', '!=', $this->id)
        ->with(['periodo', 'horario', 'aula', 'profesor'])
        ->get();
}
```

## Métodos de Gestión de Estudiantes

### Asignar Estudiante al Grupo
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

    // Verificar que el estudiante no esté ya en un grupo activo
    if ($preregistro->grupo_asignado_id && $preregistro->grupo_asignado_id != $this->id) {
        throw new \Exception('El estudiante ya está asignado a otro grupo.');
    }

    \DB::transaction(function () use ($preregistro) {
        // Actualizar preregistro
        $preregistro->update([
            'grupo_asignado_id' => $this->id,
            'estado' => 'asignado'
        ]);

        // Actualizar contador del grupo
        $this->increment('estudiantes_inscritos');
    });

    return true;
}
```

### Remover Estudiante del Grupo
```php
public function removerEstudiante($preregistroId)
{
    $preregistro = Preregistro::find($preregistroId);
    if (!$preregistro || $preregistro->grupo_asignado_id !== $this->id) {
        throw new \Exception('El estudiante no está asignado a este grupo.');
    }

    \DB::transaction(function () use ($preregistro) {
        // Actualizar preregistro
        $preregistro->update([
            'grupo_asignado_id' => null,
            'estado' => 'pendiente'
        ]);

        // Actualizar contador del grupo
        if ($this->estudiantes_inscritos > 0) {
            $this->decrement('estudiantes_inscritos');
        }
    });

    return true;
}
```

## Métodos de Validación

### Validar Asignación de Aula
```php
public function validarAsignacionAula($aulaId)
{
    $aula = Aula::find($aulaId);
    if (!$aula) {
        throw new \Exception('El aula seleccionada no existe.');
    }

    // Verificar que el aula esté disponible globalmente
    if (!$aula->disponible) {
        throw new \Exception("El aula {$aula->nombre_completo} no está disponible.");
    }

    // Verificar capacidad
    if (!$aula->soportaCapacidad($this->capacidad_maxima)) {
        throw new \Exception("El aula {$aula->nombre_completo} tiene capacidad para {$aula->capacidad} estudiantes, pero el grupo requiere {$this->capacidad_maxima}.");
    }

    // Verificar disponibilidad en horario (usando grupos existentes)
    if (!$aula->estaDisponibleEnHorario($this->horario_periodo_id)) {
        throw new \Exception("El aula {$aula->nombre_completo} ya está ocupada en este horario.");
    }

    return true;
}
```

### Validar Asignación de Profesor
```php
public function validarAsignacionProfesor($profesorId)
{
    $profesor = Profesor::find($profesorId);
    if (!$profesor) {
        throw new \Exception('El profesor seleccionado no existe.');
    }

    // Verificar conflictos de horario (usando grupos existentes)
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

## Métodos Estáticos

### Obtener Letras Disponibles
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

## Hook del Modelo

### Validaciones Automáticas al Guardar
```php
protected static function boot()
{
    parent::boot();

    static::saving(function ($grupo) {
        // Validar capacidad del aula si está asignada
        if ($grupo->aula_id && !$grupo->aulaSoportaCapacidad()) {
            throw new \Exception("El aula no tiene suficiente capacidad para el grupo.");
        }

        // Validar conflictos de horario
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

## Características de Seguridad

### Validaciones Implementadas
- **Capacidad del aula**: Verifica que el aula pueda soportar la capacidad del grupo
- **Conflictos de horario**: Detecta solapamientos de aula y profesor
- **Integridad de estudiantes**: Previene asignaciones duplicadas
- **Estados coherentes**: Valida transiciones de estado

### Manejo de Transacciones
- Operaciones críticas como asignación/remoción de estudiantes usan transacciones
- Garantiza consistencia entre preregistros y contadores de grupo

### Validaciones en Tiempo Real
- Hook `saving` valida automáticamente antes de guardar
- Métodos específicos para validaciones de asignación

Este modelo proporciona una gestión completa de grupos académicos con validaciones robustas que garantizan la integridad de los datos y previenen conflictos en las asignaciones de estudiantes, aulas y profesores.