# 🔐 Sistema de Autenticación - AuthController

## 📋 Overview
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