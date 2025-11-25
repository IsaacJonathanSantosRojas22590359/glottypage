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

## 📋 Código

### AuthController.php
```php
CODIGO PRINCIPAL AUTH CONTROLLER: <?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use App\Models\Usuario;
use App\Models\Profesor;
use App\Models\Coordinador;
use App\Models\Preregistro;

class AuthController extends Controller
{
    /**
     * LOGIN UNIFICADO CON GUARDS - VERSIÓN CORREGIDA
     */
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|string|email',
            'password' => 'required|string',
        ]);

        $email = $request->email;
        $password = $request->password;

        // ✅ CORREGIDO: Intentar autenticar como COORDINADOR (verificación manual)
        $coordinador = Coordinador::where('correo_coordinador', $email)->first();
        if ($coordinador && Hash::check($password, $coordinador->contraseña)) {
            Auth::guard('coordinador')->login($coordinador);
            return $this->redirectByGuard('coordinador');
        }

        // ✅ CORREGIDO: Intentar autenticar como PROFESOR (verificación manual)
        $profesor = Profesor::where('correo_profesor', $email)->first();
        if ($profesor && Hash::check($password, $profesor->contraseña)) {
            Auth::guard('profesor')->login($profesor);
            return $this->redirectByGuard('profesor');
        }

        // ✅ Intentar autenticar como ALUMNO (con correo personal o institucional)
        $usuario = Usuario::where('correo_personal', $email)
                         ->orWhere('correo_institucional', $email)
                         ->first();

        if ($usuario && Hash::check($password, $usuario->contraseña)) {
            Auth::guard('web')->login($usuario);
            return $this->redirectByGuard('web');
        }

        return back()->withErrors([
            'email' => 'Credenciales incorrectas.',
        ]);
    }

    /**
     * Redirige según el guard autenticado
     */
    private function redirectByGuard($guard)
    {
        $user = Auth::guard($guard)->user();
        
        $routes = [
            'web' => 'alumno.dashboard',        // Alumnos
            'profesor' => 'profesor.dashboard', // Profesores  
            'coordinador' => 'coordinador.dashboard' // Coordinadores
        ];

        // Guardar información en sesión
        session([
            'user_role' => $guard === 'web' ? 'alumno' : $guard,
            'user_fullname' => $this->getUserName($user, $guard),
            'user_identifier' => $this->getUserIdentifier($user, $guard)
        ]);

        return redirect()->route($routes[$guard]);
    }

    private function getUserName($user, $guard)
    {
        switch ($guard) {
            case 'web': return $user->nombre_completo;
            case 'profesor': return $user->nombre_profesor . ' ' . $user->apellidos_profesor;
            case 'coordinador': return $user->nombre_coordinador . ' ' . $user->apellidos_coordinador;
            default: return 'Usuario';
        }
    }

    private function getUserIdentifier($user, $guard)
    {
        switch ($guard) {
            case 'web': return $user->numero_control ?? $user->correo_institucional;
            case 'profesor': return $user->rfc_profesor;
            case 'coordinador': return $user->rfc_coordinador;
            default: return '';
        }
    }

    /**
     * REGISTRO (solo para alumnos) - ESTE ESTÁ BIEN
     */
    public function register(Request $request)
    {
        $validatedData = $request->validate([
            'correo_personal' => 'required|string|email|max:255|unique:usuarios',
            'nombre_completo' => 'required|string|max:255',
            'numero_telefonico' => 'nullable|string|max:20',
            'genero' => 'nullable|in:M,F,Otro',
            'fecha_nacimiento' => 'nullable|date',
            'tipo_usuario' => 'required|in:interno,externo',
            'correo_institucional' => 'nullable|string|email|max:255|unique:usuarios',
            'numero_control' => 'nullable|string|max:50|unique:usuarios',
            'especialidad' => 'nullable|string|max:100',
            'password' => 'required|string|min:8|confirmed',
        ]);

        try {
            $usuario = Usuario::create([
                'correo_personal' => $validatedData['correo_personal'],
                'nombre_completo' => $validatedData['nombre_completo'],
                'numero_telefonico' => $validatedData['numero_telefonico'],
                'genero' => $validatedData['genero'],
                'fecha_nacimiento' => $validatedData['fecha_nacimiento'],
                'tipo_usuario' => $validatedData['tipo_usuario'],
                'correo_institucional' => $validatedData['correo_institucional'],
                'numero_control' => $validatedData['numero_control'],
                'especialidad' => $validatedData['especialidad'],
                'contraseña' => Hash::make($validatedData['password']),
            ]);

            // Login automático con guard 'web' (alumnos)
            Auth::guard('web')->login($usuario);

            return redirect()->route('alumno.dashboard')->with('success', 'Registro exitoso');

        } catch (\Exception $e) {
            return redirect()->back()->withErrors([
                'error' => 'Error en el registro: ' . $e->getMessage()
            ]);
        }
    }

    /**
     * LOGOUT para todos los guards - ESTE ESTÁ BIEN
     */
    public function logout(Request $request)
    {
        $guards = ['web', 'profesor', 'coordinador'];
        
        foreach ($guards as $guard) {
            if (Auth::guard($guard)->check()) {
                Auth::guard($guard)->logout();
            }
        }

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }


    /**
     * DASHBOARD COORDINADOR - ACTUALIZADO para usar Auth en lugar de session
     */
// En AuthController - método coordinadorDashboard
    public function coordinadorDashboard()
    {
        $estadisticas = [
            'total_estudiantes' => 0,
            'preregistrados' => 0,
            'activos' => 0,
            'grupos_activos' => 0,
        ];

        $coordinador = Auth::guard('coordinador')->user();

        return view('coordinador', [ // ← CAMBIAR 'coordinador.dashboard' por 'coordinador'
            'nombre_completo' => $coordinador->nombre_coordinador . ' ' . $coordinador->apellidos_coordinador,
            'rfc_coordinador' => $coordinador->rfc_coordinador,
            'estadisticas' => $estadisticas
        ]);
    }

    // ✅ AGREGAR ESTE MÉTODO QUE FALTA
    public function showRegisterForm()
    {
        return view('register');
    }
}