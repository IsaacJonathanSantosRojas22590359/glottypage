# Sprint 1 - Autenticación y CRUD de Profesores

**Período**: 24 Octubre - 5 Noviembre 2024  
**Estado**: ✅ Completado  
**Equipo**: Axolotl Solutions

## 🎯 Objetivo del Sprint
"Desarrollar el sistema de autenticación unificada y el módulo CRUD básico para la gestión de profesores"

## 📊 Métricas
- **Duración**: 7 días
- **Ítems Completados**: 2 de 2 (100%)
- **Definición de Hecho**: Código escrito, probado, integrado y documentado

## 🔧 Tecnologías Implementadas
- **Backend**: Laravel, Eloquent ORM
- **Frontend**: Blade, Tailwind CSS  
- **Base de Datos**: PostgreSQL
- **Autenticación**: MiddleWare

## 📝 Entregables Principales

### 1. Sistema de Autenticación Unificada
Login seguro para 3 tipos de usuario:
- Alumnos
- Profesores  
- Coordinadores

### 2. CRUD Completo de Profesores
- Crear nuevos profesores
- Listar profesores existentes
- Editar información
- Eliminar con validaciones

## 🎨 Diseños Implementados

### Formulario de Login

![Login](imagenes/glotty-login.jpg)

### Formulario de Registro
![Registro Externo](imagenes/glotty-registro_externo.jpg)
![Registro Interno](imagenes/glotty-registro_externo.jpg)

## 📋 Código Principal

### AuthController.php
```php
public function login(Request $request) {
    // Lógica de autenticación unificada
    // para 3 tipos de usuario
}