# Modelo Profesor

## Descripción General
El modelo `Profesor` representa a los profesores dentro del sistema y extiende `Authenticatable`, lo cual permite que puedan autenticarse utilizando los mecanismos estándar de Laravel.

Este modelo gestiona:
- Campos asignables.
- Campos ocultos.
- Contraseña personalizada.
- Relación con grupos asignados.

---

# Propiedades del Modelo

## Tabla y Llave Primaria

```php
protected $table = 'profesores';
protected $primaryKey = 'id_profesor';
```

### Explicación
- El modelo usa la tabla `profesores`.
- La clave primaria es `id_profesor` en lugar del estándar `id`.

---

## Atributos Asignables

```php
protected $fillable = [
    'rfc_profesor', 'nombre_profesor', 'apellidos_profesor',
    'num_telefono', 'correo_profesor', 'contraseña'
];
```

### Explicación
Define los campos que pueden asignarse masivamente (mass assignment).  
Esto permite usar `Profesor::create()` de forma segura.

---

## Atributos Ocultos

```php
protected $hidden = ['contraseña', 'remember_token'];
```

### Explicación
Evita que la contraseña y el token de sesión aparezcan cuando se convierte el modelo a JSON o array.  
Protege información sensible.

---

# Autenticación Personalizada

## Método getAuthPassword()

```php
public function getAuthPassword()
{
    return $this->contraseña;
}
```

### Explicación
Laravel espera que el campo de contraseña se llame `password`.  
Como en este proyecto se llama `contraseña`, se sobrescribe este método para indicarle a Laravel qué atributo debe usar como contraseña.

---

# Relaciones

## Relación con grupos

```php
public function grupos()
{
    return $this->hasMany(Grupo::class, 'profesor_id', 'id_profesor');
}
```

### Explicación
Un profesor **puede tener muchos grupos asignados**.

- `Grupo::class`: modelo relacionado.
- `'profesor_id'`: clave foránea en la tabla de grupos.
- `'id_profesor'`: llave primaria del profesor.

Esto permite acceder así:

```php
$profesor->grupos;
```

---

# Resumen General

El modelo `Profesor`:
- Permite la autenticación para profesores.
- Define de forma segura los atributos que pueden asignarse.
- Mantiene confidenciales los valores sensibles.
- Proporciona una relación clara con los grupos que tiene asignados.
- Se integra correctamente con todas las operaciones CRUD del sistema.

Este modelo es fundamental para gestionar profesores dentro del sistema GLOTTY.
