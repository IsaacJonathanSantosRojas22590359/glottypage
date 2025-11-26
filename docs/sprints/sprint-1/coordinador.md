# Modelo Coordinador

## Descripción General
El modelo `Coordinador` representa a los coordinadores académicos dentro del sistema GLOTTY.  
Extiende `Authenticatable` para permitir autenticación propia y uso de guards personalizados.

Este modelo se encarga de:
- Gestionar los datos del coordinador.
- Manejar la autenticación con contraseña personalizada.
- Proteger información sensible.
- Integrarse con el sistema de login del proyecto.



# Tabla Asociada

```php
protected $table = 'coordinadores';
```

### Explicación
El modelo utiliza la tabla `coordinadores`, donde se almacena toda la información de los coordinadores.



# Atributos Asignables

```php
protected $fillable = [
    'rfc_coordinador', 'nombre_coordinador', 'apellidos_coordinador',
    'num_telefono', 'correo_coordinador', 'contraseña'
];
```

### Explicación
Estos campos pueden asignarse de forma masiva mediante:

```php
Coordinador::create($data);
```

Incluyen información personal, de contacto y las credenciales necesarias para autenticación.



# Atributos Ocultos

```php
protected $hidden = ['contraseña', 'remember_token'];
```

### Explicación
Oculta información sensible cuando el modelo se convierte a JSON o array.



# Autenticación Personalizada

## Método getAuthPassword()

```php
public function getAuthPassword()
{
    return $this->contraseña;
}
```

### Explicación
Laravel espera que la contraseña se llame `password`, pero en este proyecto se almacena como `contraseña`.  
Por ello, este método sobreescribe el comportamiento estándar y le indica a Laravel que utilice este atributo para la validación.



# Resumen General

El modelo `Coordinador`:

- Permite que los coordinadores inicien sesión con un guard específico.
- Define y protege los atributos necesarios para la gestión de coordinadores.
- Se adapta al diseño del proyecto mediante un campo de contraseña personalizado.
- Se integra con el flujo de autenticación del sistema GLOTTY.
