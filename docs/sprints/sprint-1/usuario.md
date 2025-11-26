# Modelo Usuario

## Descripción General
El modelo `Usuario` representa a los diferentes tipos de usuarios del sistema GLOTTY.  
Extiende `Authenticatable`, lo que permite que pueda autenticarse mediante los mecanismos de Laravel.

Este modelo administra:
- Información personal del usuario.
- Tipos de usuario (alumno, profesor, coordinador, etc.).
- Relación con preregistros.
- Campos asignables y sensibles.

---

# Tabla y Atributos Principales

## Nombre de la tabla

```php
protected $table = 'usuarios';
```

El modelo utiliza la tabla `usuarios`.

---

# Atributos Asignables

```php
protected $fillable = [
    'correo_personal',
    'nombre_completo',
    'numero_telefonico',
    'genero',
    'fecha_nacimiento',
    'tipo_usuario',
    'correo_institucional',
    'numero_control',
    'especialidad',
    'contraseña'
];
```

### Explicación
Estos campos pueden rellenarse mediante asignación masiva, por ejemplo:

```php
Usuario::create($data);
```

Incluyen información general del usuario, escolaridad, e incluso su tipo dentro del sistema.

---

# Atributos Ocultos

```php
protected $hidden = [
    'contraseña'
];
```

### Explicación
Evita que el campo `contraseña` aparezca en respuestas JSON o exportaciones, protegiendo información sensible.

---

# Relaciones

## Preregistros asociados

```php
public function preregistros()
{
    return $this->hasMany(Preregistro::class, 'usuario_id');
}
```

### Explicación
Un usuario puede tener **muchos preregistros**.  
La clave foránea utilizada es `usuario_id`.

Uso típico:

```php
$usuario->preregistros;
```

---

# Resumen General

El modelo `Usuario`:

- Representa a cualquier usuario del sistema GLOTTY.
- Permite autenticación gracias a `Authenticatable`.
- Define información personal, escolar y administrativa.
- Mantiene protegidos los datos sensibles como la contraseña.
- Se relaciona correctamente con los preregistros.
- Funciona como base para varios procesos del sistema, incluyendo login, preregistro y gestión de datos.
