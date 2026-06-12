# Módulo 3 — Aplicación Web con ASP.NET Core
### Taller: Sistema de Inventario con C# y .NET 8

---

## Objetivo del módulo

Exponer el inventario como una **API REST** y crear una interfaz web interactiva con **Blazor**. Al terminar:d

- Tendrás una API con los endpoints CRUD completos.
- La interfaz web consumirá la API desde el navegador.
- Los datos se guardarán en **SQLite** usando **Entity Framework Core**.
- El módulo 4 (MAUI) consumirá esta misma API.

> **Requisito**: haber completado el Módulo 1. La clase `Producto` se reutiliza como entidad de EF Core.

---

## 1. Conceptos previos

| Término | Definición |
|---|---|
| **API REST** | Interfaz que expone operaciones sobre recursos (Productos) vía HTTP. Cada operación usa un verbo: `GET`, `POST`, `PUT`, `DELETE`. |
| **ASP.NET Core** | Framework de .NET para aplicaciones web y APIs. Es multiplataforma (Windows, Linux, macOS). |
| **Entity Framework Core** | ORM (Object-Relational Mapper): traduce objetos C# a SQL y viceversa. Elimina la mayoría del SQL manual. |
| **SQLite** | Base de datos embebida en un archivo. Ideal para desarrollo y apps pequeñas. |
| **Blazor Server** | Framework UI web donde el código C# corre en el servidor y solo se envían diferencias de DOM al navegador. No requiere JavaScript. |
| **Inyección de dependencias (DI)** | En ASP.NET Core el contenedor de DI crea y administra los servicios automáticamente. No necesitas hacer `new ServicioX()` manualmente. |

---

## 2. Crear los proyectos

Desde la raíz del taller:

```bash
# Proyecto API REST (.NET 8)
dotnet new webapi --name Inventario.Api --output src/Inventario.Api --framework net8.0

# Proyecto Blazor (UI web, .NET 8)
dotnet new blazor --name Inventario.Web --output src/Inventario.Web --framework net8.0 --interactivity Server

# Agregar a la solución
dotnet sln add src/Inventario.Api/Inventario.Api.csproj
dotnet sln add src/Inventario.Web/Inventario.Web.csproj

# La Web consume la API y también usa los modelos del módulo 1
cd src/Inventario.Web
dotnet add reference ../Inventario.Consola/Inventario.Consola.csproj

en la raiz del proyecto

dotnet add src\Inventario.Api\Inventario.Api.csproj package Microsoft.EntityFrameworkCore --version 8.*
dotnet add src\Inventario.Api\Inventario.Api.csproj package Microsoft.EntityFrameworkCore.Design --version 8.*
dotnet add src\Inventario.Api\Inventario.Api.csproj package Microsoft.EntityFrameworkCore.Sqlite --version 8.*


dotnet add src\Inventario.Api\Inventario.Api.csproj reference src\Inventario.Consola\Inventario.Consola.csproj
```

> **Nota sobre versiones:** Si `--framework net8.0` no funciona en tu CLI, la plantilla por defecto usa .NET 8 de todas formas. Verifica tu versión con `dotnet --version` (debe ser 8.0 o superior).

---

## 3. Configurar Entity Framework Core en la API

### 3.1 Instalar paquetes

```bash
cd src/Inventario.Api

# Para .NET 8, usa versión 8.x de Entity Framework Core
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 8.0.0
```

> **Compatibilidad:** Asegúrate de que `dotnet --version` retorna `8.0.x` o superior. Si usas .NET 7 o inferior, ajusta la versión de los paquetes en consecuencia (`7.0.x` para .NET 7).

### 3.2 `DbContext` — `InventarioDbContext.cs`

El `DbContext` es el puente entre EF Core y la base de datos.

```csharp
// src/Inventario.Api/Data/InventarioDbContext.cs

using Inventario.Consola.Models;
using Microsoft.EntityFrameworkCore;

namespace Inventario.Api.Data;

/// <summary>
/// Representa la sesión con la base de datos.
/// DbSet<T> corresponde a una tabla en SQLite.
/// </summary>
public class InventarioDbContext : DbContext
{
    // El constructor recibe las opciones de configuración (cadena de conexión, proveedor, etc.)
    // desde el contenedor de DI. Nunca se configura aquí de forma fija.
    public InventarioDbContext(DbContextOptions<InventarioDbContext> options)
        : base(options) { }

    // DbSet<Producto> representa la tabla "Productos" en la base de datos.
    // Permite escribir consultas LINQ que EF traduce a SQL.
    public DbSet<Producto> Productos => Set<Producto>();

    /// <summary>
    /// OnModelCreating permite personalizar el mapeo objeto-relacional.
    /// </summary>
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Producto>(entity =>
        {
            // Clave primaria con autoincremento (EF la detecta por convención al llamarse "Id")
            entity.HasKey(p => p.Id);

            // Columnas con restricciones
            entity.Property(p => p.Nombre)
                  .IsRequired()           // NOT NULL en SQLite
                  .HasMaxLength(200);

            entity.Property(p => p.Categoria)
                  .HasMaxLength(100)
                  .HasDefaultValue("General");

            // 'decimal' requiere especificar precisión en SQL; (18,2) = 18 dígitos totales, 2 decimales
            entity.Property(p => p.Precio)
                  .HasColumnType("decimal(18,2)");

            entity.Property(p => p.FechaRegistro)
                  .HasDefaultValueSql("CURRENT_TIMESTAMP");  // SQLite: fecha actual del servidor
        });
    }
}
```

### 3.3 Repositorio con EF Core — `EfProductoRepository.cs`

Este repositorio implementa la misma interfaz del módulo 1, pero ahora persiste en SQLite:

```csharp
// src/Inventario.Api/Repositories/EfProductoRepository.cs

using Inventario.Api.Data;
using Inventario.Consola.Models;
using Inventario.Consola.Repositories;
using Microsoft.EntityFrameworkCore;

namespace Inventario.Api.Repositories;

/// <summary>
/// Implementación del repositorio usando Entity Framework Core + SQLite.
/// Reemplaza a JsonProductoRepository sin cambiar el servicio de negocio.
/// </summary>
public class EfProductoRepository : IProductoRepository
{
    private readonly InventarioDbContext _context;

    public EfProductoRepository(InventarioDbContext context)
    {
        _context = context;
    }

    // ToList() ejecuta la consulta SQL: SELECT * FROM Productos ORDER BY Nombre
    public List<Producto> ObtenerTodos() =>
        _context.Productos
                .OrderBy(p => p.Nombre)
                .ToList();

    // FirstOrDefault genera: SELECT TOP 1 * FROM Productos WHERE Id = @id
    public Producto? ObtenerPorId(int id) =>
        _context.Productos.FirstOrDefault(p => p.Id == id);

    public void Agregar(Producto producto)
    {
        // Add rastrea el objeto como "nuevo" (estado Added)
        _context.Productos.Add(producto);

        // SaveChanges envía el INSERT a la base de datos
        _context.SaveChanges();

        // Después de SaveChanges, EF actualiza 'producto.Id' con el valor generado por SQLite
    }

    public void Actualizar(Producto producto)
    {
        // Update rastrea el objeto como "modificado" (estado Modified)
        _context.Productos.Update(producto);
        _context.SaveChanges();  // Genera UPDATE SET ... WHERE Id = @id
    }

    public void Eliminar(int id)
    {
        var producto = _context.Productos.Find(id);

        if (producto is null)
            throw new KeyNotFoundException($"Producto con Id {id} no encontrado.");

        _context.Productos.Remove(producto);  // Estado: Deleted
        _context.SaveChanges();               // Genera DELETE WHERE Id = @id
    }
}
```

---

## 4. Configurar la API — `Program.cs`

```csharp
// src/Inventario.Api/Program.cs

using Inventario.Api.Data;
using Inventario.Api.Repositories;
using Inventario.Consola.Repositories;
using Inventario.Consola.Services;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// ── Registro de servicios en el contenedor de DI ──────────────────────────
// AddDbContext registra el DbContext y configura SQLite como proveedor.
// La cadena de conexión lee desde appsettings.json.
builder.Services.AddDbContext<InventarioDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

// Registrar el repositorio: cada vez que se pida IProductoRepository, se entrega EfProductoRepository.
// AddScoped = una instancia por solicitud HTTP (el ciclo de vida más común para repositorios)
builder.Services.AddScoped<IProductoRepository, EfProductoRepository>();

// Registrar el servicio de negocio.
// AddScoped también para el servicio porque depende del repositorio Scoped.
builder.Services.AddScoped<IInventarioService, InventarioService>();

// Habilitar controladores de API
builder.Services.AddControllers();

// CORS: permite que la app Blazor (en otro puerto) consuma la API
builder.Services.AddCors(options =>
{
    options.AddPolicy("PermitirBlazor", policy =>
    {
        // En desarrollo: permite HTTPS y HTTP desde distintos puertos
        policy.WithOrigins("https://localhost:7001", "http://localhost:5001")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

var app = builder.Build();

// ── Inicializar la base de datos ──────────────────────────────────────────
// Crear y aplicar migraciones automáticamente al iniciar la app.
// En producción esto se haría con dotnet ef database update en el pipeline CI/CD.
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<InventarioDbContext>();
    db.Database.Migrate();  // Crea el archivo SQLite y aplica las migraciones pendientes
}

// ── Pipeline de middleware ────────────────────────────────────────────────
// El middleware es una cadena de procesamiento: cada solicitud HTTP pasa por él en orden.
app.UseHttpsRedirection();  // Redirige HTTP → HTTPS
app.UseCors("PermitirBlazor");
app.UseAuthorization();
app.MapControllers();       // Registra las rutas definidas en los controladores

app.Run();
```

### Archivo `appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=inventario.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

---

## 5. Migraciones de EF Core

Las migraciones son la historia versionada del esquema de base de datos:

```bash
# Instalar la herramienta global de EF (solo una vez por máquina)
# Asegúrate de usar la versión que coincida con tu .NET SDK
dotnet tool install --global dotnet-ef --version 8.0.0

# O si ya está instalada, actualízala
dotnet tool update --global dotnet-ef --version 8.0.0

# Cambiar al directorio del proyecto API
cd src/Inventario.Api

# Crear la primera migración: genera el código SQL para crear la tabla Productos
# Si `dotnet ef` no funciona, intenta: dotnet ef --version (para verificar la instalación)
dotnet ef migrations add CrearTablaProductos --output-dir Data/Migrations

# Aplicar la migración (crea inventario.db en el directorio de trabajo)
dotnet ef database update
```

**Verificación:** Tras ejecutar `database update`, deberías ver `inventario.db` en el directorio `src/Inventario.Api/`.

EF Core genera dos archivos en la carpeta `Data/Migrations/`:
- `<timestamp>_CrearTablaProductos.cs` — código C# que crea/elimina la tabla.
- `InventarioDbContextModelSnapshot.cs` — snapshot del modelo actual.

**Nunca edites estos archivos manualmente.** Si cometes un error, usa `dotnet ef migrations remove` para deshacer la última migración.

---

## 6. Controlador de la API — `ProductosController.cs`

```csharp
// src/Inventario.Api/Controllers/ProductosController.cs

using Inventario.Consola.Models;
using Inventario.Consola.Services;
using Microsoft.AspNetCore.Mvc;

namespace Inventario.Api.Controllers;

/// <summary>
/// Expone los endpoints REST del inventario.
/// Ruta base: /api/productos
/// </summary>
[ApiController]
[Route("api/[controller]")]  // [controller] = "productos" (nombre de clase sin "Controller")
public class ProductosController : ControllerBase
{
    private readonly IInventarioService _servicio;

    // ASP.NET Core inyecta el servicio automáticamente gracias al registro en Program.cs
    public ProductosController(IInventarioService servicio)
    {
        _servicio = servicio;
    }

    // ── GET /api/productos ────────────────────────────────────────────────
    /// <summary>Devuelve todos los productos ordenados por nombre.</summary>
    [HttpGet]
    public ActionResult<IEnumerable<Producto>> ObtenerTodos()
    {
        // Ok() genera una respuesta HTTP 200 con el cuerpo serializado a JSON
        return Ok(_servicio.ListarProductos());
    }

    // ── GET /api/productos/{id} ───────────────────────────────────────────
    /// <summary>Devuelve un producto por su Id.</summary>
    [HttpGet("{id:int}")]  // :int valida que el parámetro de ruta sea un entero
    public ActionResult<Producto> ObtenerPorId(int id)
    {
        var producto = _servicio.BuscarProducto(id);

        // NotFound() genera HTTP 404 con un mensaje descriptivo
        if (producto is null)
            return NotFound(new { mensaje = $"Producto con Id {id} no encontrado." });

        return Ok(producto);
    }

    // ── POST /api/productos ───────────────────────────────────────────────
    /// <summary>Crea un nuevo producto.</summary>
    [HttpPost]
    public ActionResult<Producto> Crear([FromBody] CrearProductoRequest request)
    {
        // [FromBody] indica que el JSON del cuerpo de la solicitud se deserializa al parámetro
        try
        {
            var nuevo = _servicio.AgregarProducto(
                request.Nombre,
                request.Categoria,
                request.Precio,
                request.Stock);

            // CreatedAtAction genera HTTP 201 con el header Location apuntando al nuevo recurso
            return CreatedAtAction(nameof(ObtenerPorId), new { id = nuevo.Id }, nuevo);
        }
        catch (ArgumentException ex)
        {
            // BadRequest() genera HTTP 400 con el mensaje de error
            return BadRequest(new { mensaje = ex.Message });
        }
    }

    // ── PUT /api/productos/{id} ───────────────────────────────────────────
    /// <summary>Actualiza un producto existente.</summary>
    [HttpPut("{id:int}")]
    public IActionResult Actualizar(int id, [FromBody] CrearProductoRequest request)
    {
        try
        {
            _servicio.ActualizarProducto(id, request.Nombre, request.Categoria,
                                         request.Precio, request.Stock);
            // NoContent() genera HTTP 204: operación exitosa sin cuerpo de respuesta
            return NoContent();
        }
        catch (KeyNotFoundException ex)
        {
            return NotFound(new { mensaje = ex.Message });
        }
        catch (ArgumentException ex)
        {
            return BadRequest(new { mensaje = ex.Message });
        }
    }

    // ── DELETE /api/productos/{id} ────────────────────────────────────────
    /// <summary>Elimina un producto.</summary>
    [HttpDelete("{id:int}")]
    public IActionResult Eliminar(int id)
    {
        try
        {
            _servicio.EliminarProducto(id);
            return NoContent();
        }
        catch (KeyNotFoundException ex)
        {
            return NotFound(new { mensaje = ex.Message });
        }
    }

    // ── GET /api/productos/categoria/{categoria} ──────────────────────────
    [HttpGet("categoria/{categoria}")]
    public ActionResult<IEnumerable<Producto>> PorCategoria(string categoria)
    {
        return Ok(_servicio.BuscarPorCategoria(categoria));
    }

    // ── GET /api/productos/stock-bajo?umbral=5 ────────────────────────────
    [HttpGet("stock-bajo")]
    public ActionResult<IEnumerable<Producto>> StockBajo([FromQuery] int umbral = 5)
    {
        // [FromQuery] lee el parámetro de la query string: ?umbral=10
        return Ok(_servicio.ObtenerStockBajo(umbral));
    }
}

/// <summary>
/// DTO (Data Transfer Object): define exactamente qué campos acepta la API.
/// Usar DTOs evita exponer directamente el modelo de dominio y permite
/// evolucionar ambos de forma independiente.
/// </summary>
public record CrearProductoRequest(
    string Nombre,
    string Categoria,
    decimal Precio,
    int Stock);
```

**Puntos clave:**

- `[ApiController]` — habilita validación automática del modelo y binding desde el cuerpo HTTP.
- `ActionResult<T>` — permite devolver tanto el objeto tipado como códigos de error HTTP desde el mismo método.
- `record` (C# 9+) — tipo inmutable con igualdad por valor. Perfecto para DTOs: conciso y seguro.
- Los verbos HTTP (`GET`, `POST`, `PUT`, `DELETE`) reflejan la operación semántica, no solo la URL.

---

## 7. Interfaz web con Blazor Server

### 7.1 Servicio HTTP — `ProductoApiService.cs`

El proyecto Blazor consumirá la API mediante `HttpClient`:

```csharp
// src/Inventario.Web/Services/ProductoApiService.cs

using System.Net.Http.Json;
using Inventario.Consola.Models;

namespace Inventario.Web.Services;

/// <summary>
/// Servicio que encapsula las llamadas HTTP a la API REST de productos.
/// HttpClient es inyectado por el contenedor de DI de Blazor.
/// </summary>
public class ProductoApiService
{
    private readonly HttpClient _httpClient;
    private const string BaseUrl = "api/productos";

    public ProductoApiService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    // GetFromJsonAsync: hace GET y deserializa el JSON a List<Producto> automáticamente
    public async Task<List<Producto>> ObtenerTodosAsync() =>
        await _httpClient.GetFromJsonAsync<List<Producto>>(BaseUrl)
        ?? new List<Producto>();

    public async Task<Producto?> ObtenerPorIdAsync(int id) =>
        await _httpClient.GetFromJsonAsync<Producto>($"{BaseUrl}/{id}");

    // PostAsJsonAsync: hace POST serializando el objeto a JSON automáticamente
    public async Task<bool> AgregarAsync(ProductoDto dto)
    {
        var response = await _httpClient.PostAsJsonAsync(BaseUrl, dto);
        return response.IsSuccessStatusCode;  // true si HTTP 2xx
    }

    public async Task<bool> ActualizarAsync(int id, ProductoDto dto)
    {
        var response = await _httpClient.PutAsJsonAsync($"{BaseUrl}/{id}", dto);
        return response.IsSuccessStatusCode;
    }

    public async Task<bool> EliminarAsync(int id)
    {
        var response = await _httpClient.DeleteAsync($"{BaseUrl}/{id}");
        return response.IsSuccessStatusCode;
    }
}

// DTO para enviar datos al servidor (sin Id, sin FechaRegistro)
public record ProductoDto(string Nombre, string Categoria, decimal Precio, int Stock);
```

### 7.2 Registrar servicios en Blazor — `Program.cs`

```csharp
// src/Inventario.Web/Program.cs

using Inventario.Web.Components;
using Inventario.Web.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();

builder.Services.AddHttpClient<ProductoApiService>(client =>
{
    client.BaseAddress = new Uri("http://localhost:5103/");
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
}

app.UseStaticFiles();
app.UseAntiforgery();

app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();

app.Run();
```

### 7.3 Página de inventario — `Pages/Productos.razor`

Los componentes Blazor (`.razor`) mezclan HTML con C# en el mismo archivo:

```razor
// src/Inventario.Web/Components/Pages/Productos.razor

@page "/inventario"
@rendermode InteractiveServer
@using Microsoft.AspNetCore.Components.Web
@using Inventario.Web.Services
@using ProductoModel = Inventario.Consola.Models.Producto
@inject ProductoApiService ApiService

<PageTitle>Inventario de Tienda</PageTitle>

<h1 class="titulo">Inventario de Tienda</h1>

@* ── Barra de herramientas ──────────────────────────────────────────────── *@
<div class="barra-tools">
    <button class="btn btn-success" @onclick="AbrirFormularioNuevo">➕ Agregar</button>

    <input class="form-control buscador" placeholder="Buscar por nombre..."
           @bind="textoBusqueda" @bind:event="oninput" @bind:after="FiltrarProductos" />
</div>

@* ── Tabla de productos ──────────────────────────────────────────────────── *@
@if (cargando)
{
    <p class="estado">Cargando productos...</p>
}
else if (!productosFiltrados.Any())
{
    <p class="estado">No hay productos que mostrar.</p>
}
else
{
    <table class="tabla-inventario">
        <thead>
            <tr>
                <th>ID</th>
                <th>Nombre</th>
                <th>Categoría</th>
                <th>Precio</th>
                <th>Stock</th>
                <th>Acciones</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var producto in productosFiltrados)
            {
                <tr class="@(producto.Stock <= 5 ? "stock-bajo" : "")">
                    <td>@producto.Id</td>
                    <td>@producto.Nombre</td>
                    <td>@producto.Categoria</td>
                    <td>@producto.Precio.ToString("C2")</td>
                    <td>@producto.Stock</td>
                    <td>
                        <button class="btn btn-sm btn-primary"
                                @onclick="() => AbrirFormularioEditar(producto)">
                            ✏️ Editar
                        </button>
                        <button class="btn btn-sm btn-danger"
                                @onclick="() => ConfirmarEliminar(producto)">
                            🗑️ Eliminar
                        </button>
                    </td>
                </tr>
            }
        </tbody>
    </table>
}

@* ── Modal de formulario ─────────────────────────────────────────────────── *@
@if (mostrarFormulario)
{
    <div class="modal-fondo">
        <div class="modal-contenido">
            <h3>@(productoEditar?.Id > 0 ? "Editar" : "Agregar") Producto</h3>

            <div class="campo">
                <label>Nombre</label>
                <input @bind="formNombre" class="form-control" />
            </div>
            <div class="campo">
                <label>Categoría</label>
                <input @bind="formCategoria" class="form-control" />
            </div>
            <div class="campo">
                <label>Precio</label>
                <input type="number" step="0.01" @bind="formPrecio" class="form-control" />
            </div>
            <div class="campo">
                <label>Stock</label>
                <input type="number" @bind="formStock" class="form-control" />
            </div>

            @if (!string.IsNullOrEmpty(mensajeError))
            {
                <p class="error">@mensajeError</p>
            }

            <div class="modal-botones">
                <button class="btn btn-secondary" @onclick="CerrarFormulario">Cancelar</button>
                <button class="btn btn-success" @onclick="GuardarProducto">Guardar</button>
            </div>
        </div>
    </div>
}

@code {
    // ── Estado del componente ─────────────────────────────────────────────
    // En Blazor, las variables del bloque @code son el "estado" del componente.
    // Cuando cambian, Blazor re-renderiza la UI automáticamente.

    private List<ProductoModel> productos = new();
    private List<ProductoModel> productosFiltrados = new();
    private bool   cargando = true;
    private string textoBusqueda = "";

    // Estado del formulario modal
    private bool   mostrarFormulario = false;
    private ProductoModel? productoEditar;
    private string formNombre    = "";
    private string formCategoria = "";
    private decimal formPrecio   = 0;
    private int    formStock     = 0;
    private string mensajeError  = "";

    // OnInitializedAsync: equivalente al constructor, pero async. Se llama al montar el componente.
    protected override async Task OnInitializedAsync()
    {
        await CargarProductos();
    }

    private async Task CargarProductos()
    {
        cargando  = true;
        productos = await ApiService.ObtenerTodosAsync();
        FiltrarProductos();
        cargando  = false;
    }

    // Se llama en tiempo real mientras el usuario escribe en el buscador
    private void FiltrarProductos()
    {
        productosFiltrados = string.IsNullOrWhiteSpace(textoBusqueda)
            ? productos
            : productos
                .Where(p => p.Nombre.Contains(textoBusqueda, StringComparison.OrdinalIgnoreCase)
                         || p.Categoria.Contains(textoBusqueda, StringComparison.OrdinalIgnoreCase))
                .ToList();
    }

    private void AbrirFormularioNuevo()
    {
        productoEditar = null;
        formNombre     = "";
        formCategoria  = "";
        formPrecio     = 0;
        formStock      = 0;
        mensajeError   = "";
        mostrarFormulario = true;
    }

    private void AbrirFormularioEditar(ProductoModel producto)
    {
        productoEditar = producto;
        formNombre     = producto.Nombre;
        formCategoria  = producto.Categoria;
        formPrecio     = producto.Precio;
        formStock      = producto.Stock;
        mensajeError   = "";
        mostrarFormulario = true;
    }

    private void CerrarFormulario() => mostrarFormulario = false;

    private async Task GuardarProducto()
    {
        if (string.IsNullOrWhiteSpace(formNombre))
        {
            mensajeError = "El nombre es obligatorio.";
            return;
        }

        var dto = new ProductoDto(formNombre, formCategoria, formPrecio, formStock);
        bool exito;

        if (productoEditar?.Id > 0)
            exito = await ApiService.ActualizarAsync(productoEditar.Id, dto);
        else
            exito = await ApiService.AgregarAsync(dto);

        if (exito)
        {
            CerrarFormulario();
            await CargarProductos();  // Refresca la tabla
        }
        else
        {
            mensajeError = "Ocurrió un error al guardar. Verifica los datos.";
        }
    }

    private async Task ConfirmarEliminar(ProductoModel producto)
    {
        // En producción usarías un diálogo de confirmación personalizado.
        // Por simplicidad aquí usamos confirm de JavaScript a través de JS Interop.
        await ApiService.EliminarAsync(producto.Id);
        await CargarProductos();
    }
}

<style>
    .titulo { color: #0078d4; margin-bottom: 1rem; }
    .barra-tools { display: flex; gap: 1rem; margin-bottom: 1rem; align-items: center; }
    .buscador { max-width: 300px; }
    .tabla-inventario { width: 100%; border-collapse: collapse; }
    .tabla-inventario th { background: #0078d4; color: white; padding: 10px; }
    .tabla-inventario td { padding: 8px; border-bottom: 1px solid #ddd; }
    .tabla-inventario tr:hover { background: #f0f0f0; }
    .stock-bajo { background: #fff3cd !important; }
    .estado { color: #666; text-align: center; padding: 2rem; }
    .modal-fondo { position: fixed; top:0; left:0; width:100%; height:100%;
                   background: rgba(0,0,0,0.5); display:flex;
                   align-items:center; justify-content:center; z-index:1000; }
    .modal-contenido { background: white; padding: 2rem; border-radius: 8px;
                       min-width: 380px; box-shadow: 0 4px 20px rgba(0,0,0,0.3); }
    .campo { margin-bottom: 1rem; }
    .campo label { display:block; font-weight: bold; margin-bottom: 4px; }
    .modal-botones { display:flex; justify-content:flex-end; gap:1rem; margin-top:1.5rem; }
    .error { color: red; font-size: 0.9rem; }
    .btn { padding: 6px 14px; border: none; border-radius: 4px; cursor: pointer; }
    .btn-success { background: #198754; color: white; }
    .btn-primary { background: #0d6efd; color: white; }
    .btn-danger  { background: #dc3545; color: white; }
    .btn-secondary { background: #6c757d; color: white; }
    .btn-sm { padding: 3px 8px; font-size: 0.85rem; }
    .form-control { width: 100%; padding: 6px; border: 1px solid #ccc; border-radius: 4px; }
</style>
```

**Puntos clave de Blazor:**

- `@page "/inventario"` — define la ruta de la página.
- `@inject` — inyecta un servicio registrado en el DI de Blazor.
- `@bind` — enlace bidireccional: cuando la variable cambia, la UI se actualiza; cuando el usuario edita, la variable cambia.
- `@onclick="() => Metodo(param)"` — *lambda* que captura el parámetro del foreach actual.
- `@code { }` — bloque C# dentro del componente. Las variables aquí son el estado reactivo.
- `async Task` — en Blazor, los manejadores de eventos y el ciclo de vida son *async* para no bloquear el UI thread.

---

## 8. Ejecutar ambos proyectos simultáneamente

### 8.1 Configurar puertos (opcional)

Por defecto, los proyectos usan puertos generados aleatoriamente. Para fijarlos:

1. **API** — edita `src/Inventario.Api/Properties/launchSettings.json`:
```json
{
  "profiles": {
    "https": {
      "commandName": "Project",
      "launchBrowser": false,
      "applicationUrl": "https://localhost:7000;http://localhost:5000"
    }
  }
}
```

2. **Web** — edita `src/Inventario.Web/Properties/launchSettings.json`:
```json
{
  "profiles": {
    "https": {
      "commandName": "Project",
      "launchBrowser": true,
      "applicationUrl": "https://localhost:7001;http://localhost:5001"
    }
  }
}
```

Luego actualiza `Program.cs` en ambos proyectos si es necesario.

### 8.2 Ejecutar en dos terminales

Abre dos terminales en VS Code o tu terminal preferida:

**Terminal 1 — API:**
```bash
cd src/Inventario.Api
dotnet run
# Escucha en https://localhost:7000 (o el puerto que hayas configurado)
```

**Terminal 2 — Web (desde otra terminal):**
```bash
cd src/Inventario.Web
dotnet run
# Escucha en https://localhost:7001 (o el puerto que hayas configurado)
```

**En el navegador:** abre `https://localhost:7001/inventario` (ajusta el puerto si es distinto).

Para ejecutar ambos con un solo comando, puedes usar el archivo `.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "run-api",
      "type": "shell",
      "command": "dotnet run --project src/Inventario.Api"
    },
    {
      "label": "run-web",
      "type": "shell",
      "command": "dotnet run --project src/Inventario.Web"
    },
    {
      "label": "run-all",
      "dependsOn": ["run-api", "run-web"],
      "dependsOrder": "parallel"
    }
  ]
}
```

---

## 9. Probar la API con archivos `.http`

VS Code con la extensión **REST Client** permite probar endpoints directamente:

```http
### Crear archivo: src/Inventario.Api/api-tests.http

@base = https://localhost:7100/api/productos

### Obtener todos los productos
GET {{base}}
Accept: application/json

### Obtener producto por Id
GET {{base}}/1

### Crear producto
POST {{base}}
Content-Type: application/json

{
  "nombre": "Laptop Pro 15",
  "categoria": "Electrónica",
  "precio": 1299.99,
  "stock": 10
}

### Actualizar producto
PUT {{base}}/1
Content-Type: application/json

{
  "nombre": "Laptop Pro 15 (Actualizada)",
  "categoria": "Electrónica",
  "precio": 1199.99,
  "stock": 8
}

### Eliminar producto
DELETE {{base}}/1

### Productos por categoría
GET {{base}}/categoria/Electrónica

### Stock bajo
GET {{base}}/stock-bajo?umbral=3
```

---

## 10. Diagrama de arquitectura del módulo

```
 Navegador
    │
    ▼
┌───────────────────────────────────┐
│        Inventario.Web (Blazor)    │
│   Inventario.razor                │
│   ProductoApiService              │
└──────────────┬────────────────────┘
               │ HTTP (JSON)
               ▼
┌───────────────────────────────────┐
│       Inventario.Api (REST)        │
│   ProductosController             │
│   InventarioService               │
│   EfProductoRepository            │
└──────────────┬────────────────────┘
               │ EF Core
               ▼
        🗄️ inventario.db (SQLite)
```

---

## 11. Reto del módulo

1. **Paginación**: modifica `GET /api/productos` para aceptar `?pagina=1&tamaño=10` y devolver solo una página.
2. **Autenticación básica**: protege los endpoints `POST`, `PUT` y `DELETE` con una API Key enviada en el header `X-Api-Key`.
3. **Gráfico de stock**: en la página Blazor agrega una visualización simple (barras HTML/CSS) que muestre el stock de los 10 productos con menor inventario.

---

**Inicio del taller →** [README](https://github.com/juangamboaabarca/practica_dotnet/blob/main/README.md)

**← Módulo anterior**: [Módulo 2: WinForms](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-02-winforms.md)  
**Siguiente módulo →**: [Módulo 4: Aplicación Multiplataforma con MAUI](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-04-maui.md)
