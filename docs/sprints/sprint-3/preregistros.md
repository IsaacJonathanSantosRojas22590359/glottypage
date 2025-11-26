# Documentación del Modelo Preregistro

## Descripción General
El modelo `Preregistro` representa las solicitudes de preregistro de estudiantes para cursos de inglés en el sistema. Gestiona toda la información relacionada con las inscripciones preliminares de estudiantes, incluyendo niveles solicitados, horarios preferidos, estados de pago y asignación a grupos.

## Estructura del Modelo

### Namespace y Definición de Clase
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Preregistro extends Model
{
    use HasFactory;
```

## Propiedades del Modelo

### Atributos Mass Assignable
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

### Casts de Atributos
```php
protected $casts = [
    'nivel_solicitado' => 'integer'
];
```

## Constantes del Sistema

### Estados del Preregistro
```php
const ESTADOS = [
    'pendiente' => 'Pendiente de Asignación',
    'asignado' => 'Asignado a Grupo',
    'cursando' => 'Cursando',
    'finalizado' => 'Finalizado',
    'cancelado' => 'Cancelado'
];
```

### Estados de Pago (Incluye Prórroga)
```php
const PAGO_ESTADOS = [
    'pendiente' => 'Pendiente de Pago',
    'prorroga' => 'En Prórroga',
    'pagado' => 'Pagado',
    'rechazado' => 'Pago Rechazado'
];
```

### Niveles Disponibles
```php
const NIVELES = [
    1 => 'Nivel 1',
    2 => 'Nivel 2', 
    3 => 'Nivel 3',
    4 => 'Nivel 4',
    5 => 'Nivel 5'
];
```

## Relaciones

### Relación con Usuario
```php
public function usuario()
{
    return $this->belongsTo(Usuario::class);
}
```

### Relación con Periodo
```php
public function periodo()
{
    return $this->belongsTo(Periodo::class);
}
```

### Relación con HorarioPreferido
```php
public function horarioPreferido()
{
    return $this->belongsTo(HorarioPeriodo::class, 'horario_preferido_id');
}
```

### Relación con GrupoAsignado
```php
public function grupoAsignado()
{
    return $this->belongsTo(Grupo::class, 'grupo_asignado_id');
}
```

## Scopes de Consulta

### Scope de Preregistros Pendientes
```php
public function scopePendientes($query)
{
    return $query->where('estado', 'pendiente');
}
```

### Scope de Preregistros Pagados
```php
public function scopePagados($query)
{
    return $query->where('pago_estado', 'pagado');
}
```

### Scope de Preregistros que Pueden Asignarse
```php
public function scopePuedenAsignarse($query)
{
    return $query->where('estado', 'pendiente')
                ->whereIn('pago_estado', ['pagado', 'prorroga']);
}
```

### Scope de Preregistros Activos
```php
public function scopeActivos($query)
{
    return $query->whereIn('estado', ['pendiente', 'asignado', 'cursando']);
}
```

### Scope de Preregistros con Prórroga
```php
public function scopeConProrroga($query)
{
    return $query->where('pago_estado', 'prorroga');
}
```

## Accessors (Getters Computados)

### Nivel Formateado
```php
public function getNivelFormateadoAttribute()
{
    return self::NIVELES[$this->nivel_solicitado] ?? "Nivel {$this->nivel_solicitado}";
}
```

### Estado Formateado
```php
public function getEstadoFormateadoAttribute()
{
    return self::ESTADOS[$this->estado] ?? $this->estado;
}
```

### Estado de Pago Formateado
```php
public function getPagoEstadoFormateadoAttribute()
{
    return self::PAGO_ESTADOS[$this->pago_estado] ?? $this->pago_estado;
}
```

### Color de Estado de Pago
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

## Métodos de Negocio

### Verificar si Puede ser Asignado
```php
public function puedeSerAsignado()
{
    return $this->estado === 'pendiente' && 
           in_array($this->pago_estado, ['pagado', 'prorroga']) &&
           $this->periodo->aceptaPreRegistros();
}
```

**Condiciones para asignación:**
- Estado debe ser 'pendiente'
- Pago debe estar 'pagado' o 'prorroga'
- El periodo debe aceptar preregistros

### Verificar si Está Listo para Cursar
```php
public function estaListoParaCursar()
{
    return $this->estado === 'asignado' && 
           in_array($this->pago_estado, ['pagado', 'prorroga']);
}
```

### Verificar si Puede ser Cancelado
```php
public function puedeSerCancelado()
{
    return in_array($this->estado, ['pendiente', 'asignado']) &&
           !$this->periodo->estaFinalizado();
}
```

### Verificar Estados Específicos
```php
public function estaCursando()
{
    return $this->estado === 'cursando';
}

public function estaFinalizado()
{
    return $this->estado === 'finalizado';
}

public function estaCancelado()
{
    return $this->estado === 'cancelado';
}

public function tieneProrroga()
{
    return $this->pago_estado === 'prorroga';
}
```

### Verificar si Está Vencido
```php
public function estaVencido()
{
    return $this->pago_estado === 'pendiente' && 
           $this->created_at->diffInDays(now()) > 7;
}
```

**Lógica de vencimiento:**
- Solo aplica para estado de pago 'pendiente'
- Considera vencido después de 7 días desde la creación

### Sincronizar con Estado del Periodo
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

**Automatización de estados:**
- Cambia a 'cursando' cuando el periodo inicia y el preregistro está listo
- Cambia a 'finalizado' cuando el periodo finaliza y estaba cursando

### Validar Transiciones de Estado
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

**Reglas de transición:**
- **pendiente** → asignado, cancelado
- **asignado** → cursando, cancelado  
- **cursando** → finalizado
- **finalizado** → (sin transiciones permitidas)
- **cancelado** → (sin transiciones permitidas)

## Flujos de Trabajo y Validaciones

### Ciclo de Vida del Preregistro

1. **Creación**: Estado inicial 'pendiente', pago 'pendiente'
2. **Pago**: Cambia a 'pagado' o 'prorroga'
3. **Asignación**: Cambia a 'asignado' cuando se asigna a grupo
4. **Inicio de Curso**: Cambia a 'cursando' cuando el periodo inicia
5. **Finalización**: Cambia a 'finalizado' cuando el periodo termina
6. **Cancelación**: Puede cancelarse en estados 'pendiente' o 'asignado'

### Validaciones de Integridad

**Para Asignación a Grupo:**
- Preregistro debe estar en estado 'pendiente'
- Pago debe estar 'pagado' o 'prorroga'
- Periodo debe aceptar preregistros

**Para Inicio de Curso:**
- Preregistro debe estar 'asignado'
- Pago debe estar 'pagado' o 'prorroga'
- Periodo debe estar 'en_curso'

**Para Cancelación:**
- Solo permitido en estados 'pendiente' o 'asignado'
- Periodo no debe estar finalizado

### Gestión de Prórrogas

**Características de la prórroga:**
- Estado de pago especial 'prorroga'
- Permite asignación a grupos igual que 'pagado'
- Color azul en interfaces para identificación visual
- Scope específico para consultas

## Integración con el Sistema

### Relación con Periodos
- Los preregistros están vinculados a periodos específicos
- Sincronización automática con cambios de estado del periodo
- Validación de que el periodo acepte preregistros

### Relación con Grupos
- Asignación opcional a grupos mediante `grupo_asignado_id`
- Información del grupo disponible a través de la relación

### Relación con Horarios
- Horario preferido seleccionado durante el preregistro
- Información del horario base disponible para mostrar al estudiante

Este modelo proporciona una gestión completa de los preregistros de estudiantes con validaciones robustas que garantizan la integridad de los datos y automatizaciones que sincronizan el estado del preregistro con el ciclo de vida del periodo académico.