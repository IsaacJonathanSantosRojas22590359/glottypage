# Documentación del Sistema de Autenticación Unificado

Este documento describe el funcionamiento del sistema de autenticación implementado en el proyecto GLOTTY. Se explica el flujo general, la lógica que sigue el controlador y la estructura del inicio de sesión con múltiples roles (alumno, profesor y coordinador).

---

## Introducción

El sistema de autenticación del proyecto GLOTTY fue diseñado para permitir que tres tipos de usuarios ingresen al sistema desde un único formulario. Para lograr esta funcionalidad, se utilizan distintos guards de Laravel, así como validaciones y verificaciones manuales de credenciales que permiten determinar qué tipo de usuario intenta acceder.

El AuthController centraliza todo el proceso de autenticación, registro de alumnos y cierre de sesión. A continuación se detalla su funcionamiento.

---

# Controlador de Autenticación (AuthController)

El AuthController administra las siguientes funciones principales:

- Inicio de sesión unificado para alumnos, profesores y coordinadores.
- Registro exclusivo de alumnos.
- Cierre de sesión para cualquier tipo de usuario autenticado.
- Redirección hacia el dashboard correspondiente según el rol.
- Obtención del nombre y datos identificadores del usuario autenticado.
- Visualización del dashboard del coordinador.

A continuación se detalla cada sección del controlador.

---

## Inicio de Sesión Unificado

El método `login` recibe las credenciales del usuario y verifica la coincidencia con cada uno de los modelos correspondientes. Este proceso se ejecuta en el siguiente orden:

1. Coordinador  
2. Profesor  
3. Alumno  

El sistema compara el correo ingresado con los campos correctos de cada tabla y verifica la contraseña utilizando `Hash::check()`. Cuando se encuentra una coincidencia válida, se inicia sesión utilizando el guard apropiado.

Si ninguno de los tres tipos coincide, se devuelve un error indicando que las credenciales son incorrectas.

Código del método:

```php
public function login(Request $request)
{
    ...
}
```

---

## Redirección de Usuarios

El método `redirectByGuard` determina la ruta correcta para enviar al usuario una vez que inició sesión. Cada tabla posee su propio dashboard definido en las rutas del proyecto.

Además, este método almacena en la sesión información útil como:

- Rol del usuario autenticado
- Nombre completo
- Identificador principal (RFC, número de control o correo institucional)

Estas funciones permiten personalizar la interfaz según el tipo de usuario.

Código del método:

```php
private function redirectByGuard($guard)
{
    ...
}
```

---

## Obtención del Nombre del Usuario

El método `getUserName` retorna el nombre completo del usuario según el guard con el que autenticó. Cada tabla posee campos distintos, por lo que es necesario manejar cada tipo por separado.

Ejemplo:

```php
private function getUserName($user, $guard)
{
    ...
}
```

---

## Obtención del Identificador del Usuario

El método `getUserIdentifier` devuelve un identificador clave del usuario, el cual puede variar dependiendo del rol:

- Alumnos: número de control o correo institucional  
- Profesores: RFC  
- Coordinadores: RFC  

Código:

```php
private function getUserIdentifier($user, $guard)
{
    ...
}
```

---

## Registro de Alumnos

El sistema únicamente permite registrar alumnos. Los coordinadores y profesores deben ser creados mediante otros procesos administrativos.

El método `register` valida los datos, crea un registro en la tabla de usuarios y autentica automáticamente al alumno usando el guard correspondiente.

Código:

```php
public function register(Request $request)
{
    ...
}
```

---

## Cierre de Sesión

El método `logout` cierra la sesión del usuario sin importar con qué guard haya ingresado. Recorre los tres posibles guards y cierra sesión en el que esté activo.

Luego invalida la sesión y redirige al inicio.

Código:

```php
public function logout(Request $request)
{
    ...
}
```

---

## Dashboard del Coordinador

El método `coordinadorDashboard` obtiene al coordinador autenticado mediante su guard y envía la información necesaria hacia la vista correspondiente. También prepara estadísticas básicas que posteriormente podrán ser alimentadas con datos reales del sistema.

Código:

```php
public function coordinadorDashboard()
{
    ...
}
```

---

## Vista de Registro

El método `showRegisterForm` simplemente muestra la vista de registro de alumnos.

Código:

```php
public function showRegisterForm()
{
    return view('register');
}
```

---

## Conclusión

Ela AuthController constituye el núcleo de la autenticación del sistema GLOTTY. Su diseño permite gestionar múltiples roles desde un único punto, asegurando una estructura flexible y escalable. La implementación basada en guards facilita la separación clara entre tipos de usuarios, mientras que el manejo detallado de sesiones y datos del usuario contribuye a una experiencia más personalizada.

Este módulo constituye uno de los pilares fundamentales del Sprint 1, asegurando la seguridad y correcto control de acceso al sistema.
