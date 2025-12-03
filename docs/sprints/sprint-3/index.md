# Resumen General del Sistema de Gestión Académica

**Historial de Versiones**
| Versión | Fecha | Autor | Descripción de los Cambios |
|---------|-------|-------|---------------------------|
| 1.0 | 15/01/2025 | Isaac Jonathan Santos Rojas | Documentación inicial del Sprint 3 |
| 1.1 | 20/01/2025 | Isaac Jonathan Santos Rojas | Agregados modelos Periodo y Preregistro |
| | | | |

## Arquitectura del Sistema

### Estructura MVC Implementada
El sistema sigue el patrón Modelo-Vista-Controlador (MVC) de Laravel, con una clara separación de responsabilidades:

- **Modelos**: Gestionan datos y lógica de negocio
- **Controladores**: Manejan la lógica de aplicación y flujos de trabajo
- **Vistas**: Presentan la interfaz de usuario

## Módulos Principales Implementados

### 1. Gestión de Aulas
**Modelo**: `Aula`
**Controlador**: `AulaController`

**Funcionalidades**:
- CRUD completo de aulas académicas
- Gestión de capacidad (15-200 estudiantes)
- Tipos de aula: regular, laboratorio, cómputo, audiovisual
- Sistema de disponibilidad y horarios ocupados
- Validación de capacidad vs grupos asignados
- API para consulta de aulas disponibles

**Estados**: disponible/no disponible

### 2. Gestión de Horarios Base
**Modelo**: `Horario`
**Controlador**: `HorarioController`

**Funcionalidades**:
- CRUD de horarios base (semanal/sabatino)
- Validación de días según tipo (semanal: L-V, sabatino: solo sábado)
- Prevención de edición/eliminación en periodos activos
- Sistema de activación/desactivación

**Estados**: activo/inactivo

### 3. Gestión de Periodos Académicos
**Modelo**: `Periodo`
**Controlador**: `PeriodoController`

**Funcionalidades**:
- CRUD de periodos académicos
- Gestión de estados del periodo (6 estados)
- Transiciones controladas entre estados
- Validación de dependencias (horarios, aulas, profesores)
- Automatización de estados de grupos y estudiantes

**Estados**: 
- configuración → preregistros_activos → preregistros_cerrados → en_curso → finalizado/cancelado

### 4. Gestión de Grupos
**Modelo**: `Grupo`
**Controlador**: `CoordinadorGrupoController`

**Funcionalidades**:
- CRUD de grupos académicos
- Asignación de estudiantes a grupos
- Validación de capacidad y conflictos de horario
- Estados automáticos según asignaciones
- Gestión de profesores y aulas

**Estados**: planificado → con_profesor → con_aula → activo → en_curso → finalizado/cancelado

### 5. Gestión de Preregistros
**Modelo**: `Preregistro`
**Controlador**: `PreregistroController`

**Funcionalidades**:
- Preregistro de estudiantes en periodos activos
- Gestión de estados de pago (incluye prórroga)
- Asignación automática a grupos
- Sincronización con estados del periodo
- Validación de unicidad por periodo

**Estados**: pendiente → asignado → cursando → finalizado/cancelado

## Características Técnicas Comunes

### Validaciones Implementadas
- **Validaciones de formulario**: Reglas Laravel estándar
- **Validaciones de negocio**: Lógica personalizada en modelos
- **Validaciones de integridad**: Prevención de operaciones inválidas
- **Validaciones de estado**: Control de transiciones entre estados

### Seguridad y Control de Acceso
- **Autenticación requerida** para todas las operaciones
- **Autorización por roles** (coordinador, estudiante)
- **Verificación de propiedad** de datos
- **Prevención de operaciones inválidas** según estado

### Manejo de Transacciones
- **Operaciones críticas** usan transacciones de base de datos
- **Consistencia garantizada** en operaciones múltiples
- **Rollback automático** en caso de errores

### Sistema de Logging
- **Registro de operaciones** importantes
- **Información contextual** para debugging
- **Auditoría** de cambios en el sistema

## Flujos de Trabajo Principales

### 1. Configuración de Periodo
```
Crear periodo → Agregar horarios → Activar preregistros
```

### 2. Preregistro de Estudiantes
```
Periodo activo → Preregistro estudiante → Pago → Asignación a grupo
```

### 3. Gestión de Grupos
```
Crear grupo → Asignar profesor/aula → Asignar estudiantes → Activar grupo
```

### 4. Ciclo de Vida Académico
```
Inicio periodo → Grupos activos → Estudiantes cursando → Finalización
```

## Estados y Transiciones

### Periodos (6 estados)
- **configuración**: Configuración inicial
- **preregistros_activos**: Aceptando preregistros
- **preregistros_cerrados**: Preregistros cerrados
- **en_curso**: Periodo activo
- **finalizado**: Periodo completado
- **cancelado**: Periodo cancelado

### Grupos (6 estados)
- **planificado**: Creado sin asignaciones
- **con_profesor**: Profesor asignado
- **con_aula**: Aula asignada
- **activo**: Listo para estudiantes
- **en_curso**: Impartiendo clases
- **finalizado/cancelado**: Estados finales

### Preregistros (5 estados)
- **pendiente**: Esperando asignación
- **asignado**: Asignado a grupo
- **cursando**: Activo en clases
- **finalizado**: Curso completado
- **cancelado**: Preregistro cancelado

## Integraciones y Relaciones

### Relaciones entre Modelos
- **Periodo** ↔ **HorarioPeriodo** ↔ **Horario** (Base)
- **Periodo** ↔ **Grupo** ↔ **Aula** ↔ **Profesor**
- **Periodo** ↔ **Preregistro** ↔ **Usuario** (Estudiante)
- **Grupo** ↔ **Preregistro** (Estudiantes asignados)

### Dependencias del Sistema
- **Para activar preregistros**: Horarios + Aulas + Profesores
- **Para activar grupos**: Profesor + Aula + Capacidad
- **Para asignar estudiantes**: Preregistro pagado + Capacidad disponible

## Características de Usabilidad

### Para Coordinadores
- Interfaces administrativas completas
- Filtros y búsquedas avanzadas
- Estadísticas en tiempo real
- Validaciones preventivas

### Para Estudiantes
- Interfaz simplificada de preregistro
- Seguimiento de estado de solicitudes
- Información clara de grupos asignados

## Métricas y Estadísticas

### Disponibles en el Sistema
- Ocupación de aulas y grupos
- Distribución por niveles
- Estados de preregistros y pagos
- Eficiencia de utilización de recursos

Este sistema proporciona una plataforma completa para la gestión académica de cursos de inglés, con validaciones robustas, automatizaciones inteligentes y una interfaz adecuada para los diferentes roles de usuario.