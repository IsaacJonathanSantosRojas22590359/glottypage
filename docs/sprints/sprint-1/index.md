# Sprint 1: Implementación de Autenticación y Gestión de Personal

**Fecha:** 24 Octubre - 5 Noviembre 2024  
**Estado:** Completado  
**Equipo:** Axolotl Solutions

**Historial de Versiones**
| Versión | Fecha | Autor | Descripción de los Cambios |
|---------|-------|-------|---------------------------|
| 1.0 | 26/10/2025 | Isaac Jonathan Santos Rojas | Documentación inicial del Sprint 1 |
| 1.1 | 06/11/2025 | Isaac Jonathan Santos Rojas | Agregados diagramas y detalles técnicos |
| | | | |

## Objetivo
Establecer los fundamentos de seguridad del sistema y la administración de recursos humanos académicos.

## Actividades Desarrolladas
Se implementó el sistema de autenticación y autorización, junto con la gestión inicial de los perfiles académicos.

## Arquitectura del Sprint 1
El sistema de autenticación de GLOTTY implementa un sistema unificado multi-rol que permite el acceso seguro a tres tipos de usuarios mediante un único punto de entrada. Utiliza el sistema de Guards de Laravel para mantener sesiones separadas y seguras.

### Controladores Desarrollados
- **`AuthController.php`** - Gestión centralizada de autenticación para todos los tipos de usuario
- **`ProfesorController.php`** - Operaciones CRUD para la administración del cuerpo docente

### Modelos Implementados
- **`Usuario.php`** - Modelo base para la gestión de alumnos internos y externos
- **`Profesor.php`** - Modelo para la gestión de información del personal docente
- **`Coordinador.php`** - Modelo para la administración de usuarios coordinadores

## Sistema de Autenticación Unificado

### Características Principales
- **Login multi-rol** para 3 tipos de usuario: Alumnos, Profesores y Coordinadores
- **Sistema de Guards** de Laravel para separación de sesiones
- **Registro seguro** de alumnos con validaciones completas
- **Redirección inteligente** a dashboards específicos


### Flujo de Autenticación

![Flujo de Autenticación](imagenes/flujoAut.png)

### Explicación del Flujo de Autenticación

El diagrama representa el proceso completo que sigue el sistema GLOTTY cuando un usuario intenta iniciar sesión. Se muestran las interacciones entre cuatro componentes principales:

**1. Inicio del proceso**

El usuario proporciona su correo y contraseña.  
Estas credenciales se envían al AuthController, que es el responsable de procesarlas.

**2. Búsqueda del usuario en los modelos**

El controlador primero consulta los modelos para intentar identificar qué tipo de usuario está intentando iniciar sesión.  
La búsqueda inicia con Coordinador, luego Profesor y finalmente Alumno.

**3. Validación por tipo de usuario**

El sistema sigue una secuencia de verificaciones.


1. **Coordinador válido**

    Si se encuentra un registro en el modelo de Coordinador y la contraseña coincide, el AuthController inicia sesión usando el guard coordinador.

    Esto permite mantener su sesión separada de otros roles.

2. **Profesor válido**

    Si no es coordinador, el sistema evalúa la tabla de profesores.
    Si encuentra una coincidencia:
    El profesor inicia sesión con su propio guard, que posee rutas y vistas independientes.

3. **Alumno válido**

    Si no es profesor, revisa el modelo de alumnos.
    Si coincide:
    El guard web se usa para los alumnos, ya que es el estándar en Laravel.

4. **Credenciales inválidas**

    Si no coincide con ninguno de los tres modelos, entonces:
    Se envía un mensaje indicando que el correo o la contraseña no son correctos.