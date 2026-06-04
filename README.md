# Taller Práctico: Desarrollo de Aplicaciones con .NET 8 y C#
### De la consola a la nube — un sistema de inventario en cuatro etapas

[![Repositorio](https://img.shields.io/badge/GitHub-practica__dotnet-181717?logo=github)](https://github.com/juangamboaabarca/practica_dotnet)
[![.NET](https://img.shields.io/badge/.NET-8.0_LTS-512BD4?logo=dotnet)](https://dotnet.microsoft.com/download)
[![Licencia](https://img.shields.io/badge/Licencia-MIT-green)](LICENSE)

> **Repositorio oficial:** https://github.com/juangamboaabarca/practica_dotnet

---

## ¿De qué trata este taller?

Este taller guía la construcción de un mismo sistema —un **inventario de tienda**— usando cuatro tipos de aplicaciones distintas con .NET 8 y C#. Cada módulo parte donde el anterior terminó: el código de negocio escrito en el módulo 1 se reutiliza sin modificación en todos los siguientes.

La progresión está diseñada para que cada módulo introduzca exactamente un conjunto nuevo de conceptos, sin reescribir lo que ya funciona. Al finalizar habrás recorrido toda la amplitud del ecosistema .NET moderno: terminal, escritorio Windows, web y móvil multiplataforma.

---

## Público objetivo

Desarrolladores con experiencia previa en otro lenguaje (Java, Python, JavaScript u otros) que quieran aprender .NET y C# desde un enfoque práctico. No se asume conocimiento previo de .NET; sí se asume comprensión de conceptos como clases, funciones, condicionales y colecciones.

---

## Herramientas requeridas

| Herramienta | Versión mínima | Descarga |
|---|---|---|
| .NET SDK | 8.0 LTS | https://dotnet.microsoft.com/download |
| Visual Studio Code | Cualquier reciente | https://code.visualstudio.com |
| C# Dev Kit (extensión) | — | Marketplace de VS Code |
| .NET MAUI (extensión) | — | Marketplace de VS Code (módulo 4) |
| Android SDK / Emulador | API 33+ | Android Studio (módulo 4) |

---

## Estructura del taller

```
taller-dotnet/
├── README.md                   ← Este archivo
├── modulo-01-consola.md        ← Módulo 1: Consola
├── modulo-02-winforms.md       ← Módulo 2: Escritorio
├── modulo-03-web.md            ← Módulo 3: Web
├── modulo-04-maui.md           ← Módulo 4: Multiplataforma
└── src/
    ├── Inventario.Consola/     ← Proyecto Módulo 1 y base compartida
    ├── Inventario.WinForms/    ← Proyecto Módulo 2
    ├── Inventario.Api/         ← Proyecto Módulo 3 (backend)
    ├── Inventario.Web/         ← Proyecto Módulo 3 (frontend)
    └── Inventario.Movil/       ← Proyecto Módulo 4
```

---

## Módulos

### [Módulo 1 — Aplicación de Consola](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-01-consola.md)

El punto de partida. Se construye la **capa de negocio completa** del sistema: modelo de datos, repositorio con persistencia en JSON y servicio con validaciones. La interfaz es un menú de texto en la terminal.

Este módulo establece la arquitectura en tres capas (Models / Services / Repositories) que los módulos siguientes heredan sin cambios.

**Conceptos:** namespaces, clases y propiedades, interfaces, inyección de dependencias manual, LINQ, `System.Text.Json`, manejo de excepciones, top-level statements, funciones locales.

**Duración estimada:** 3–4 horas

**Producto:** Aplicación de consola funcional con CRUD completo y persistencia en `inventario.json`.

---

### [Módulo 2 — Aplicación de Escritorio con WinForms](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-02-winforms.md)

La misma lógica de negocio del módulo 1 se expone ahora en una ventana de Windows. El proyecto WinForms referencia directamente el proyecto de consola —no copia código— lo que demuestra en la práctica qué significa separar capas.

Se construye una ventana principal con tabla de productos y un formulario modal de alta/edición, escritos a mano en código (sin diseñador visual) para entender cada control y su comportamiento.

**Conceptos:** `Form`, controles `DataGridView` / `Panel` / `Button`, `DockStyle`, `BindingSource`, modelo de eventos (`+=`), `ShowDialog` y `DialogResult`, archivos `.Designer.cs`.

**Duración estimada:** 3–4 horas

**Producto:** Aplicación de escritorio Windows con tabla interactiva, formularios modales y persistencia en JSON (reutilizada del módulo 1).

---

### [Módulo 3 — Aplicación Web con ASP.NET Core + Blazor](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-03-web.md)

El sistema se divide en dos: una **API REST** que expone los endpoints del inventario y una **interfaz web** en Blazor que la consume. Los datos migran de JSON a **SQLite** mediante Entity Framework Core, introduciendo migraciones de base de datos.

Es el módulo de mayor densidad conceptual: abarca HTTP, REST, ORM, inyección de dependencias automática, CORS, componentes reactivos y comunicación cliente-servidor.

**Conceptos:** controladores API, verbos HTTP, códigos de estado, DTOs, EF Core, DbContext, migraciones, Blazor components, `@bind`, `@code`, `async/await`, `HttpClient`, CORS.

**Duración estimada:** 5–6 horas

**Producto:** API REST documentable con archivo `.http` de pruebas + interfaz web en Blazor con búsqueda en tiempo real y modal de edición. Base de datos SQLite persistida en disco.

---

### [Módulo 4 — Aplicación Multiplataforma con .NET MAUI](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-04-maui.md)

Un único proyecto de código compila y corre en **Android, iOS y Windows**. La app consume la API REST del módulo 3 y aplica el patrón **MVVM** para separar la lógica de presentación de la interfaz XAML.

Se introducen los mecanismos de reactividad (`INotifyPropertyChanged`, `ObservableCollection`), comandos, navegación basada en rutas y configuración diferenciada por plataforma mediante directivas de compilación.

**Conceptos:** XAML, data binding, MVVM, `ICommand`, `ObservableCollection`, `INotifyPropertyChanged`, Shell navigation, `IQueryAttributable`, converters, `#if ANDROID`, configuración de `HttpClient` por plataforma.

**Duración estimada:** 5–6 horas

**Producto:** App móvil y de escritorio que corre en Android (emulador o físico) y Windows desde el mismo proyecto, con lista de productos, gestos de deslizamiento para editar/eliminar y formulario de alta.

---

## Duración total estimada

| Módulo | Horas estimadas |
|---|---|
| Módulo 1 — Consola | 3–4 h |
| Módulo 2 — WinForms | 3–4 h |
| Módulo 3 — Web | 5–6 h |
| Módulo 4 — MAUI | 5–6 h |
| **Total** | **16–20 horas** |

Las estimaciones asumen trabajo individual con lectura del código y comprensión de los conceptos. En formato de taller grupal con instructor, los módulos 1 y 2 pueden condensarse en una jornada de 8 horas; los módulos 3 y 4 requieren una segunda jornada.

---

## Productos esperados al finalizar

Al completar el taller habrás construido y tendrás funcionando localmente:

1. `Inventario.Consola` — aplicación de terminal con menú interactivo y archivo JSON.
2. `Inventario.WinForms` — aplicación de escritorio Windows con interfaz gráfica.
3. `Inventario.Api` — API REST con 7 endpoints documentados y base de datos SQLite.
4. `Inventario.Web` — interfaz web en Blazor conectada a la API.
5. `Inventario.Movil` — aplicación MAUI ejecutable en Android y Windows.

Todos los proyectos viven en la misma solución (`TallerInventario.sln`) y comparten la capa de dominio del módulo 1.

---

## Hilo conductor arquitectónico

Una de las ideas centrales del taller es mostrar que **cambiar la interfaz no implica reescribir la lógica**. El diagrama siguiente muestra qué se reutiliza y qué es nuevo en cada módulo:

```
Módulo 1          Módulo 2          Módulo 3              Módulo 4
─────────         ─────────         ────────────────      ──────────────
Consola UI   →   WinForms UI   →   Blazor UI         →   MAUI XAML UI
     │                │               │                        │
     └────────────────┘               │                        │
          InventarioService           │                        │
          (sin cambios)               │                        │
               │                     │                        │
   JsonProductoRepository    EfProductoRepository       API REST
   (archivo JSON)            (SQLite + EF Core)      (módulo 3)
```

---

## Retos por módulo

Cada módulo incluye al final tres retos opcionales para reforzar los conceptos antes de avanzar. No son requisito para continuar, pero se recomienda intentar al menos uno antes de pasar al siguiente módulo.

---

## Convenciones de código usadas en el taller

- **Nombres en español** para propiedades, métodos y variables del dominio — facilita la lectura al aprender sin agregar vocabulario técnico en inglés innecesariamente.
- **Comentarios en cada bloque** que explica por qué se escribe así, no solo qué hace el código.
- **Un concepto nuevo por vez** — cada archivo introduce la menor cantidad de elementos nuevos posible.
- **Sin frameworks externos de terceros** en los módulos 1–3 (salvo EF Core, que es oficial de Microsoft). El módulo 4 menciona `CommunityToolkit.Mvvm` como lectura recomendada, pero no lo usa para que el patrón MVVM quede visible.

---

## Orden de trabajo recomendado

1. Instala las herramientas antes de la primera sesión y verifica con `dotnet --version`.
2. Sigue los módulos en orden. El módulo 2 no tiene sentido sin el 1; el 4 depende del 3.
3. Lee el código antes de copiarlo. Los comentarios son parte del material.
4. Ejecuta la app después de cada sección, no solo al final del módulo.
5. Intenta el reto del módulo antes de avanzar; si te trancas, avanza y vuelve.

---

---

## Subir al repositorio

El repositorio oficial del taller es:
**https://github.com/juangamboaabarca/practica_dotnet**

Para clonar y comenzar desde cero:

```bash
git clone https://github.com/juangamboaabarca/practica_dotnet.git
cd practica_dotnet
```

Si ya tienes los archivos localmente y quieres subir todo al repositorio por primera vez:

```bash
# Desde la carpeta raíz del taller (donde está este README.md)
git init
git remote add origin https://github.com/juangamboaabarca/practica_dotnet.git

# Agregar todos los archivos .md y el código fuente
git add README.md modulo-*.md
git add src/

git commit -m "Taller .NET 8: consola, WinForms, Web y MAUI"

git branch -M main
git push -u origin main
```

Para actualizar el repositorio después de cambios:

```bash
git add .
git commit -m "descripción del cambio"
git push
```

---

*Taller desarrollado con .NET 8 LTS — C# 12 — Visual Studio Code*  
*Repositorio: https://github.com/juangamboaabarca/practica_dotnet*
