# Controlador ProfesorController

## Descripción General
El controlador `ProfesorController` gestiona las operaciones CRUD relacionadas con la administración de profesores. Contiene métodos para listar, registrar, editar, actualizar y eliminar profesores, incluyendo validaciones y manejo de errores.

---

# Métodos del Controlador

## 1. index()

### Descripción
Obtiene todos los profesores registrados y los envía a la vista correspondiente.

### Flujo
1. Recupera todos los registros con `Profesor::all()`.
2. Retorna la vista `coordinador.profesores`.

### Propósito
Mostrar una lista completa de profesores disponibles en el sistema.

---

## 2. create()

### Descripción
Retorna la vista con el formulario para registrar un nuevo profesor.

### Propósito
Permitir capturar la información de un profesor desde cero.

---

## 3. store(Request $request)

### Descripción
Valida y registra un nuevo profesor en la base de datos.

### Validaciones
- `rfc_profesor`: requerido, único, máximo 255 caracteres.
- `nombre_profesor`: requerido.
- `correo_profesor`: requerido, email, único.
- `password`: mínimo 8 caracteres, requiere confirmación.

### Flujo
1. Valida los datos.
2. Crea el profesor con `Profesor::create()`.
3. Encripta la contraseña usando `Hash::make`.
4. Redirige con mensaje de éxito o error.

---

## 4. edit($id)

### Descripción
Obtiene un profesor por su ID y lo envía a la vista para editar sus datos.

### Propósito
Modificar la información almacenada previamente.

---

## 5. update(Request $request, $id)

### Descripción
Actualiza la información de un profesor, incluyendo la validación dinámica que evita conflictos con otros registros.

### Validaciones
- `rfc_profesor`: único excepto el actual.
- `correo_profesor`: único excepto el actual.
- `password`: opcional, mínimo 8 caracteres si se proporciona.

### Flujo
1. Se validan los datos.
2. Se prepara el arreglo de nuevos datos.
3. Se encripta la contraseña solo si fue proporcionada.
4. Se actualiza el registro.
5. Se redirige con la respuesta correspondiente.

---

## 6. destroy($id)

### Descripción
Elimina un profesor si no tiene grupos asignados.

### Flujo
1. Busca el profesor.
2. Verifica si tiene grupos asignados mediante `grupos()->exists()`.
3. Bloquea la eliminación si existen grupos relacionados.
4. Elimina el registro si no hay dependencias.

### Propósito
Mantener la integridad referencial del sistema.

---

# Resumen General

El controlador `ProfesorController` implementa un CRUD completo, asegurando:
- Validaciones consistentes.
- Encriptación segura de contraseñas.
- Protección contra eliminación cuando existen dependencias.
- Manejo claro de errores y mensajes al usuario.
- Vistas organizadas para cada operación específica.

Este módulo garantiza una administración de profesores robusta, segura y fácil de mantener.
