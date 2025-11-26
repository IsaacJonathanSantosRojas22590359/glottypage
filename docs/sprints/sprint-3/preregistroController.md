# Documentación del Controlador PreregistroController

## Descripción General
El controlador `PreregistroController` gestiona toda la lógica relacionada con los preregistros de estudiantes en el sistema. Proporciona funcionalidades para que los alumnos realicen preregistros en periodos activos, consulten sus preregistros existentes y visualicen el estado de sus solicitudes.

## Estructura del Controlador

### Namespace y Dependencias
```php
namespace App\Http\Controllers;

use App\Models\Preregistro;
use App\Models\Periodo;
use App\Models\HorarioPeriodo;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
```

## Métodos Principales

### Método: create()
Muestra el formulario para que el estudiante realice un nuevo preregistro.

```php
public function create()
{
    // Obtener periodo activo para preregistros
    $periodoActivo = Periodo::conPreRegistrosActivos()->first();
    
    if (!$periodoActivo) {
        return redirect()->route('alumno.dashboard')
            ->with('error', 'No hay un periodo activo para preregistro en este momento.');
    }

    // Obtener horarios del periodo activo 
    $horarios = HorarioPeriodo::where('periodo_id', $periodoActivo->id)
        ->where('activo', true)
        ->with('horarioBase')
        ->get();

    // Verificar si ya tiene preregistro activo
    $preregistroExistente = Preregistro::where('usuario_id', Auth::id())
        ->where('periodo_id', $periodoActivo->id)
        ->whereIn('estado', ['pendiente', 'asignado', 'cursando'])
        ->first();

    if ($preregistroExistente) {
        return redirect()->route('alumno.dashboard')
            ->with('info', 'Ya tienes un preregistro activo para este periodo.');
    }

    return view('alumno.preregistro.create', compact('periodoActivo', 'horarios'));
}
```

**Funcionalidades del método:**
- Verifica la existencia de un periodo activo para preregistros
- Obtiene los horarios disponibles del periodo activo
- Valida que el estudiante no tenga un preregistro activo en el mismo periodo
- Retorna la vista del formulario de preregistro

### Método: store(Request $request)
Procesa y almacena un nuevo preregistro realizado por el estudiante.

```php
public function store(Request $request)
{
    // Obtener periodo activo para preregistros
    $periodoActivo = Periodo::conPreRegistrosActivos()->first();
    
    if (!$periodoActivo) {
        return back()->with('error', 'No hay un periodo activo para preregistro.');
    }

    // Validar que no tenga preregistro activo
    $preregistroExistente = Preregistro::where('usuario_id', Auth::id())
        ->where('periodo_id', $periodoActivo->id)
        ->whereIn('estado', ['pendiente', 'asignado', 'cursando'])
        ->first();

    if ($preregistroExistente) {
        return redirect()->route('alumno.dashboard')
            ->with('error', 'Ya tienes un preregistro activo para este periodo.');
    }

    $validatedData = $request->validate([
        'nivel_solicitado' => 'required|integer|between:1,5',
        'horario_preferido_id' => 'required|exists:horarios_periodo,id',
        'semestre_actual' => 'required|integer',
    ]);

    try {
        Preregistro::create([
            'usuario_id' => Auth::id(),
            'periodo_id' => $periodoActivo->id,
            'nivel_solicitado' => $validatedData['nivel_solicitado'],
            'horario_preferido_id' => $validatedData['horario_preferido_id'],
            'semestre_actual' => $validatedData['semestre_actual'],
            'estado' => 'pendiente',
            'pago_estado' => 'pendiente'
        ]);

        return redirect()->route('alumno.preregistro.index')
            ->with('success', 'Preregistro realizado exitosamente. Serás asignado a un grupo próximamente.');

    } catch (\Exception $e) {
        return back()->with('error', 'Error al realizar el preregistro: ' . $e->getMessage())
            ->withInput();
    }
}
```

**Validaciones implementadas:**
- **nivel_solicitado**: Número entero entre 1 y 5
- **horario_preferido_id**: Debe existir en la tabla de horarios_periodo
- **semestre_actual**: Número entero requerido

**Flujo del método:**
1. Verifica periodo activo para preregistros
2. Valida que no exista preregistro activo previo
3. Valida datos del formulario
4. Crea el registro de preregistro
5. Redirige con mensaje de éxito o error

### Método: index()
Muestra la lista de preregistros del estudiante autenticado.

```php
public function index()
{
    $preregistros = Preregistro::with([
            'periodo', 
            'horarioPreferido.horarioBase',
            'grupoAsignado'
        ])
        ->where('usuario_id', Auth::id())
        ->orderBy('created_at', 'desc')
        ->get();

    return view('alumno.preregistro.index', compact('preregistros'));
}
```

**Características:**
- Carga relaciones necesarias para mostrar información completa
- Filtra solo los preregistros del usuario autenticado
- Ordena por fecha de creación descendente
- Retorna vista con la colección de preregistros

### Método: show(Preregistro $preregistro)
Muestra los detalles de un preregistro específico.

```php
public function show(Preregistro $preregistro)
{
    // Verificar que el preregistro pertenezca al usuario
    if ($preregistro->usuario_id !== Auth::id()) {
        abort(403);
    }

    $preregistro->load([
        'periodo',
        'horarioPreferido.horarioBase', 
        'grupoAsignado.profesor',
        'grupoAsignado.aula'
    ]);

    return view('alumno.preregistro.show', compact('preregistro'));
}
```

**Seguridad implementada:**
- Verificación de propiedad del preregistro
- Código 403 si el preregistro no pertenece al usuario

**Relaciones cargadas:**
- Periodo del preregistro
- Horario preferido con su horario base
- Grupo asignado con profesor y aula

## Características de Seguridad

### Control de Acceso
- **Autenticación requerida**: Todos los métodos requieren usuario autenticado
- **Propiedad de datos**: Los estudiantes solo pueden ver sus propios preregistros
- **Verificación de periodo**: Solo permite preregistros en periodos activos

### Validaciones de Negocio

#### Para Crear Preregistro:
- Solo un preregistro activo por periodo por estudiante
- Periodo debe estar en estado "preregistros_activos"
- Horarios deben estar activos en el periodo

#### Estados de Preregistro Considerados Activos:
- **pendiente**: Preregistro realizado, esperando asignación
- **asignado**: Estudiante asignado a un grupo
- **cursando**: Estudiante activo en un grupo

### Manejo de Errores
- Mensajes descriptivos para el usuario
- Redirección apropiada según el contexto
- Preservación de datos del formulario en caso de error

## Flujos de Trabajo

### Flujo de Preregistro Exitoso:
1. Estudiante accede al formulario de preregistro
2. Sistema verifica periodo activo y preregistros existentes
3. Estudiante completa formulario con nivel y horario preferido
4. Sistema crea preregistro con estado "pendiente"
5. Estudiante es redirigido a la lista de preregistros con mensaje de éxito

### Flujo con Restricciones:
1. Si no hay periodo activo: muestra error y redirige al dashboard
2. Si ya tiene preregistro activo: muestra información y redirige al dashboard
3. Si hay error en validación: muestra error y mantiene datos del formulario

## Relaciones con Otros Componentes

### Integración con Modelo Periodo:
- Usa el scope `conPreRegistrosActivos()` para obtener periodos válidos
- Relaciona preregistro con periodo específico

### Integración con Modelo HorarioPeriodo:
- Proporciona horarios disponibles para selección
- Relaciona preregistro con horario preferido

### Integración con Modelo Grupo:
- Permite visualizar grupo asignado cuando existe
- Carga información de profesor y aula del grupo

Este controlador proporciona una interfaz segura y amigable para que los estudiantes realicen sus preregistros, con validaciones robustas que garantizan la integridad de los datos y previenen duplicados o operaciones inválidas.