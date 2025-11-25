# 🔐 Sistema de Autenticación - AuthController

## 📋 Descripción general
Controlador principal que maneja la autenticación unificada para los tres tipos de usuario del sistema GLOTTY: **Alumnos**, **Profesores** y **Coordinadores**.

## 🏗️ Estructura del Controlador

### Namespace e Imports
```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use App\Models\Usuario;
use App\Models\Profesor;
use App\Models\Coordinador;
use App\Models\Preregistro;
```
## Flujo de Autenticación por Capas
**El sistema intenta autenticar en el siguiente orden: Coordinador → Profesor → Alumno**
```php
public function login(Request $request)
{
    $request->validate([
        'email' => 'required|string|email',
        'password' => 'required|string',
    ]);

    $email = $request->email;
    $password = $request->password;

    // Capa 1: Autenticación como COORDINADOR
    $coordinador = Coordinador::where('correo_coordinador', $email)->first();
    if ($coordinador && Hash::check($password, $coordinador->contraseña)) {
        Auth::guard('coordinador')->login($coordinador);
        return $this->redirectByGuard('coordinador');
    }

    // Capa 2: Autenticación como PROFESOR
    $profesor = Profesor::where('correo_profesor', $email)->first();
    if ($profesor && Hash::check($password, $profesor->contraseña)) {
        Auth::guard('profesor')->login($profesor);
        return $this->redirectByGuard('profesor');
    }

    // Capa 3: Autenticación como ALUMNO
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
```


