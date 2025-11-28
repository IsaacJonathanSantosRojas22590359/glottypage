# Sistema GLOTTY - Documentación Técnica

## Descripción del Proyecto
Bienvenido a la documentación técnica oficial del Sistema GLOTTY, una plataforma de gestión académica desarrollada en Laravel por el equipo Axolotl Solutions. Este sistema está diseñado para instituciones educativas que requieren una solución integral para la administración de procesos académicos.

## Estado del Proyecto

### Sprints Completados
- **Sprint 1**: Sistema de Autenticación y Gestión de Personal - Completado
- **Sprint 2**: Gestión de Infraestructura Académica - Completado  
- **Sprint 3**: Implementación de Procesos Académicos - Completado
- **Sprint 4**: Reestructuración Arquitectónica - Completado

### Próximos Sprints
- **Sprint 5**: Panel de Profesor para Gestión de Calificaciones - En Planificación
- **Sprint 6**: Despliegue en Ambiente de Producción - En Planificación

## Equipo de Desarrollo
**Axolotl Solutions** está compuesto por 8 profesionales especializados en desarrollo de software:

- Daniel Martinez Hernandez - Scrum Master / Product Owner
- Mauricio Darío Sandoval Mandujano - Desarrollador Backend
- Aníbal Atzi Zarate Hernández - Diseñador UI/UX
- Leonardo Daniel Montes Barajas - Diseñador Frontend
- Claudio Axel Ponce Guerrero - Ingeniero de Calidad
- Juan Enrique Basurto Luna - Analista de Sistemas
- Omar García Perrusquia - Especialista en Pruebas
- Isaac Jonathan Santos Rojas - Documentador Técnico

## Metodología de Desarrollo
El proyecto sigue la metodología Scrum con sprints de 2 semanas de duración. Cada iteración incluye planificación, desarrollo, revisión y retrospectiva, garantizando la entrega continua de valor y la adaptación a los requerimientos del cliente.

## Arquitectura del Sistema

### Tecnologías Principales
- **Backend**: Laravel 10.x, PHP 8.x
- **Frontend**: Blade Templates, Tailwind CSS, JavaScript
- **Base de Datos**: MySQL con Eloquent ORM
- **Autenticación**: Sistema multi-rol con Guards de Laravel
- **Control de Versiones**: Git con GitHub

### Módulos Implementados
- **Módulo de Seguridad**: Autenticación y autorización multi-rol
- **Módulo Académico**: Gestión de periodos, grupos y preregistros
- **Módulo de Infraestructura**: Administración de aulas y horarios
- **Módulo de Personal**: Gestión de profesores y coordinadores

## Navegación de Documentación

### Documentación por Sprint
- [Sprint 1 - Autenticación y Gestión de Personal](sprints/sprint-1/index.md)
- [Sprint 2 - Infraestructura Académica](sprints/sprint-2/index.md)
- [Sprint 3 - Procesos Académicos](sprints/sprint-3/index.md)
- [Sprint 4 - Reestructuración Arquitectónica](sprints/sprint-4/index.md)

### Documentación General
- [Evolución del Proyecto](evolucion.md)
- [Equipo de Desarrollo](equipo.md)
- [Próximos Sprints](sprints/proximos-sprints.md)

## Características Destacadas

### Arquitectura Centrada en Periodos Académicos
Después de la reestructuración del Sprint 4, el sistema establece el periodo académico como entidad raíz de la que dependen todas las operaciones, garantizando coherencia organizacional y escalabilidad.

### Sistema de Autenticación Unificado
Implementa un mecanismo de login multi-rol que permite el acceso seguro para tres tipos de usuario: alumnos, profesores y coordinadores, mediante el uso de Guards de Laravel.

### Gestión Contextualizada
Todas las entidades del sistema (grupos, horarios, preregistros) operan dentro del contexto de un periodo académico específico, facilitando la gestión histórica y la trazabilidad.

## Estado de Calidad
Todos los módulos desarrollados han sido verificados mediante pruebas manuales exhaustivas. Actualmente nos encontramos en proceso de implementación de pruebas automatizadas y preparación para el despliegue en producción.

## Conclusión
El Sistema GLOTTY ha evolucionado desde una base fundamental de seguridad hacia una plataforma académica robusta con arquitectura centrada en periodos académicos. La documentación técnica proporciona información detallada sobre cada componente del sistema para facilitar el mantenimiento y futuras expansiones.

*Documentación mantenida por Isaac Jonathan Santos Rojas - Axolotl Solutions*
*Última actualización: Noviembre 2025*