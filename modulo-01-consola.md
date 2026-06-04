# Módulo 1 — Aplicación de Consola en .NET
### Taller: Sistema de Inventario con C# y .NET 8

---

## Objetivo del módulo

Construir una aplicación de consola funcional que gestione un inventario de productos. Al terminar este módulo tendrás:

- El entorno de desarrollo completamente configurado.
- Una aplicación con menú interactivo (CRUD completo).
- Persistencia de datos en un archivo JSON local.
- Las clases base que **reutilizarás en todos los módulos siguientes**.

---

## 1. Configuración del entorno

### 1.1 Herramientas a instalar

| Herramienta | Descarga | Notas |
|---|---|---|
| .NET 8 SDK | https://dotnet.microsoft.com/download | Elige la versión LTS más reciente |
| Visual Studio Code | https://code.visualstudio.com | Editor principal del taller |
| Extensión C# Dev Kit | Marketplace de VS Code | Busca `ms-dotnettools.csdevkit` |

Verifica la instalación en la terminal:

```bash
dotnet --version
# Debe mostrar algo como: 8.0.xxx
```

### 1.2 Extensiones recomendadas en VS Code

Abre el panel de extensiones (`Ctrl+Shift+X`) e instala:

- **C# Dev Kit** — IntelliSense, depuración y explorador de soluciones.
- **NuGet Gallery** — administrar paquetes NuGet desde VS Code.
- **.NET Install Tool** — gestión automática del SDK desde el editor.

---

## 2. Crear el proyecto

Abre la terminal integrada de VS Code (`Ctrl+`` `) y ejecuta:

```bash
# Crear carpeta raíz del taller
mkdir taller-dotnet
cd taller-dotnet

# Crear la solución (contenedor de todos los módulos)
dotnet new sln --name TallerInventario

# Crear el proyecto de consola dentro de una subcarpeta
dotnet new console --name Inventario.Consola --output src/Inventario.Consola

# Agregar el proyecto a la solución
dotnet sln add src/Inventario.Consola/Inventario.Consola.csproj
```

La solución es importante porque **los módulos siguientes compartirán código** de este proyecto base.

Abre la carpeta en VS Code:

```bash
code .
```

---

## 3. Estructura del proyecto

Dentro de `src/Inventario.Consola/` crearemos la siguiente estructura:

```
Inventario.Consola/
├── Models/
│   └── Producto.cs          ← Entidad principal
├── Services/
│   ├── IInventarioService.cs  ← Contrato (interfaz)
│   └── InventarioService.cs   ← Lógica de negocio
├── Repositories/
│   ├── IProductoRepository.cs ← Contrato de acceso a datos
│   └── JsonProductoRepository.cs ← Persistencia en JSON
├── Program.cs               ← Punto de entrada y menú
└── Inventario.Consola.csproj
```

> **¿Por qué esta estructura?**  
> Separar el código en capas (*Models / Services / Repositories*) no es burocracia: te permite cambiar el motor de base de datos, el tipo de interfaz, o la lógica de negocio de forma independiente. En los módulos 2, 3 y 4 **reutilizaremos exactamente estas capas** sin modificarlas.

---

## 4. Modelo de datos — `Producto.cs`

Crea la carpeta `Models` y el archivo `Producto.cs`:

```csharp
// src/Inventario.Consola/Models/Producto.cs

namespace Inventario.Consola.Models;

/// <summary>
/// Representa un artículo dentro del inventario de la tienda.
/// </summary>
public class Producto
{
    // Identificador único del producto. Se genera automáticamente al crear.
    public int Id { get; set; }

    // Nombre descriptivo del producto. Requerido.
    public string Nombre { get; set; } = string.Empty;

    // Categoría a la que pertenece (ej: "Electrónica", "Ropa").
    public string Categoria { get; set; } = string.Empty;

    // Precio de venta al público.
    public decimal Precio { get; set; }

    // Unidades disponibles en bodega.
    public int Stock { get; set; }

    // Fecha en que se registró el producto en el sistema.
    public DateTime FechaRegistro { get; set; } = DateTime.Now;
}
```

**Puntos clave del código:**

- `namespace Inventario.Consola.Models` — organiza la clase en un espacio de nombres que refleja su ubicación en el proyecto. En C# moderno (archivo-scoped namespaces) se declara en una sola línea sin llaves.
- `{ get; set; }` — propiedad auto-implementada: el compilador genera los campos privados por ti.
- `= string.Empty` — inicialización defensiva para evitar valores `null` en cadenas.
- `= DateTime.Now` — valor por defecto que se asigna cuando se instancia el objeto.

---

## 5. Repositorio — acceso a datos

### 5.1 Interfaz `IProductoRepository.cs`

```csharp
// src/Inventario.Consola/Repositories/IProductoRepository.cs

using Inventario.Consola.Models;

namespace Inventario.Consola.Repositories;

/// <summary>
/// Define el contrato de persistencia. Cualquier fuente de datos
/// (JSON, SQLite, API remota) debe implementar estos métodos.
/// </summary>
public interface IProductoRepository
{
    List<Producto> ObtenerTodos();
    Producto? ObtenerPorId(int id);
    void Agregar(Producto producto);
    void Actualizar(Producto producto);
    void Eliminar(int id);
}
```

**Puntos clave:**

- `interface` — define un contrato sin implementación. Obliga a que cualquier repositorio concreto exponga exactamente estos métodos.
- `Producto?` — el `?` indica que el método puede devolver `null` cuando no encuentra el producto (tipo de referencia anulable, habilitado por defecto en .NET 8).
- El módulo 3 tendrá un repositorio que guardará en SQLite, pero el servicio de negocio no necesitará cambiar gracias a esta interfaz.

### 5.2 Implementación `JsonProductoRepository.cs`

Primero agrega el paquete para manejar JSON:

```bash
cd src/Inventario.Consola
dotnet add package System.Text.Json
# Nota: en .NET 8 ya está incluido; este paso sería para versiones anteriores.
```

Crea el archivo:

```csharp
// src/Inventario.Consola/Repositories/JsonProductoRepository.cs

using System.Text.Json;
using Inventario.Consola.Models;

namespace Inventario.Consola.Repositories;

/// <summary>
/// Persiste los productos en un archivo JSON local llamado "inventario.json".
/// </summary>
public class JsonProductoRepository : IProductoRepository
{
    // Ruta donde se almacenará el archivo JSON.
    // AppDomain.CurrentDomain.BaseDirectory apunta a la carpeta de compilación.
    private readonly string _rutaArchivo =
        Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "inventario.json");

    // Opciones de serialización: nombres en camelCase y JSON con formato legible.
    private readonly JsonSerializerOptions _opciones = new()
    {
        WriteIndented = true,
        PropertyNameCaseInsensitive = true
    };

    // ── Métodos de soporte ──────────────────────────────────────────────────

    /// <summary>
    /// Lee el archivo JSON y deserializa la lista de productos.
    /// Si el archivo no existe, devuelve una lista vacía.
    /// </summary>
    private List<Producto> LeerArchivo()
    {
        if (!File.Exists(_rutaArchivo))
            return new List<Producto>();

        string json = File.ReadAllText(_rutaArchivo);
        return JsonSerializer.Deserialize<List<Producto>>(json, _opciones)
               ?? new List<Producto>();
    }

    /// <summary>
    /// Serializa la lista completa y sobreescribe el archivo JSON.
    /// </summary>
    private void GuardarArchivo(List<Producto> productos)
    {
        string json = JsonSerializer.Serialize(productos, _opciones);
        File.WriteAllText(_rutaArchivo, json);
    }

    // ── Implementación de la interfaz ───────────────────────────────────────

    public List<Producto> ObtenerTodos() => LeerArchivo();

    public Producto? ObtenerPorId(int id) =>
        LeerArchivo().FirstOrDefault(p => p.Id == id);

    public void Agregar(Producto producto)
    {
        var lista = LeerArchivo();

        // Genera el próximo ID autoincremental (máximo actual + 1).
        // Si la lista está vacía, el primer ID será 1.
        producto.Id = lista.Count > 0 ? lista.Max(p => p.Id) + 1 : 1;

        lista.Add(producto);
        GuardarArchivo(lista);
    }

    public void Actualizar(Producto producto)
    {
        var lista = LeerArchivo();

        // Busca el índice del producto a reemplazar.
        int indice = lista.FindIndex(p => p.Id == producto.Id);

        if (indice == -1)
            throw new KeyNotFoundException($"Producto con Id {producto.Id} no encontrado.");

        lista[indice] = producto;   // Reemplaza el objeto completo en esa posición.
        GuardarArchivo(lista);
    }

    public void Eliminar(int id)
    {
        var lista = LeerArchivo();

        // RemoveAll elimina todos los elementos que cumplan la condición.
        // En este caso, como los IDs son únicos, solo borra uno.
        int eliminados = lista.RemoveAll(p => p.Id == id);

        if (eliminados == 0)
            throw new KeyNotFoundException($"Producto con Id {id} no encontrado.");

        GuardarArchivo(lista);
    }
}
```

**Puntos clave:**

- `readonly` — el campo solo puede asignarse en la declaración o en el constructor. Protege contra reasignaciones accidentales.
- El patrón **leer → modificar → guardar** es simple y válido para volúmenes pequeños. En el módulo 3 lo reemplazaremos con EF Core para mayor escalabilidad.
- `FirstOrDefault` — método LINQ que devuelve el primer elemento que cumple la condición, o `null` si ninguno la cumple.
- `=>` en métodos de una línea — *expression body*, equivalente a escribir `{ return ...; }`.

---

## 6. Servicio de negocio

### 6.1 Interfaz `IInventarioService.cs`

```csharp
// src/Inventario.Consola/Services/IInventarioService.cs

using Inventario.Consola.Models;

namespace Inventario.Consola.Services;

public interface IInventarioService
{
    List<Producto> ListarProductos();
    Producto? BuscarProducto(int id);
    Producto AgregarProducto(string nombre, string categoria, decimal precio, int stock);
    void ActualizarProducto(int id, string nombre, string categoria, decimal precio, int stock);
    void EliminarProducto(int id);
    List<Producto> BuscarPorCategoria(string categoria);
    List<Producto> ObtenerStockBajo(int umbral = 5);
}
```

### 6.2 Implementación `InventarioService.cs`

```csharp
// src/Inventario.Consola/Services/InventarioService.cs

using Inventario.Consola.Models;
using Inventario.Consola.Repositories;

namespace Inventario.Consola.Services;

/// <summary>
/// Contiene la lógica de negocio del inventario.
/// Depende de IProductoRepository, no de una implementación concreta.
/// Esto se conoce como inyección de dependencias manual.
/// </summary>
public class InventarioService : IInventarioService
{
    // El servicio conoce solo el contrato (interfaz), no el detalle de almacenamiento.
    private readonly IProductoRepository _repositorio;

    // Constructor: recibe el repositorio desde fuera (inyección de dependencias).
    public InventarioService(IProductoRepository repositorio)
    {
        _repositorio = repositorio;
    }

    public List<Producto> ListarProductos() =>
        _repositorio.ObtenerTodos()
                    .OrderBy(p => p.Nombre)   // LINQ: ordena alfabéticamente
                    .ToList();

    public Producto? BuscarProducto(int id) =>
        _repositorio.ObtenerPorId(id);

    public Producto AgregarProducto(string nombre, string categoria, decimal precio, int stock)
    {
        // Validaciones de negocio: lanzan excepción si los datos son inválidos.
        if (string.IsNullOrWhiteSpace(nombre))
            throw new ArgumentException("El nombre no puede estar vacío.");
        if (precio < 0)
            throw new ArgumentException("El precio no puede ser negativo.");
        if (stock < 0)
            throw new ArgumentException("El stock no puede ser negativo.");

        var producto = new Producto
        {
            Nombre    = nombre.Trim(),
            Categoria = categoria.Trim(),
            Precio    = precio,
            Stock     = stock
        };

        _repositorio.Agregar(producto);
        return producto;  // Devuelve el producto ya con su Id asignado.
    }

    public void ActualizarProducto(int id, string nombre, string categoria,
                                   decimal precio, int stock)
    {
        var existente = _repositorio.ObtenerPorId(id)
            ?? throw new KeyNotFoundException($"No existe un producto con Id {id}.");

        existente.Nombre    = nombre.Trim();
        existente.Categoria = categoria.Trim();
        existente.Precio    = precio;
        existente.Stock     = stock;

        _repositorio.Actualizar(existente);
    }

    public void EliminarProducto(int id) => _repositorio.Eliminar(id);

    public List<Producto> BuscarPorCategoria(string categoria) =>
        _repositorio.ObtenerTodos()
                    // StringComparison.OrdinalIgnoreCase: comparación sin importar mayúsculas
                    .Where(p => p.Categoria.Equals(categoria, StringComparison.OrdinalIgnoreCase))
                    .OrderBy(p => p.Nombre)
                    .ToList();

    public List<Producto> ObtenerStockBajo(int umbral = 5) =>
        _repositorio.ObtenerTodos()
                    .Where(p => p.Stock <= umbral)
                    .OrderBy(p => p.Stock)
                    .ToList();
}
```

**Puntos clave:**

- `?? throw` — operador *null-coalescing throw*: si el resultado es `null`, lanza la excepción directamente. Es más compacto que un `if` separado.
- Validaciones en el servicio, no en la UI — así la misma validación aplica tanto en consola como en WinForms o la API web.
- LINQ (`Where`, `OrderBy`, `ToList`) — consultas funcionales sobre colecciones. Son el equivalente de SQL en memoria.

---

## 7. Punto de entrada — `Program.cs`

```csharp
// src/Inventario.Consola/Program.cs

using Inventario.Consola.Repositories;
using Inventario.Consola.Services;

// ── Composición manual de dependencias ─────────────────────────────────────
// Creamos el repositorio concreto (JSON) y lo inyectamos al servicio.
// En ASP.NET Core (módulo 3) esto lo hará el contenedor de DI automáticamente.
var repositorio = new JsonProductoRepository();
var servicio    = new InventarioService(repositorio);

// ── Bucle principal ─────────────────────────────────────────────────────────
bool salir = false;

while (!salir)
{
    Console.Clear();
    MostrarEncabezado();

    Console.WriteLine("  [1] Listar productos");
    Console.WriteLine("  [2] Agregar producto");
    Console.WriteLine("  [3] Editar producto");
    Console.WriteLine("  [4] Eliminar producto");
    Console.WriteLine("  [5] Buscar por categoría");
    Console.WriteLine("  [6] Productos con stock bajo");
    Console.WriteLine("  [0] Salir");
    Console.Write("\n  Selecciona una opción: ");

    string opcion = Console.ReadLine() ?? "";

    // switch con pattern matching de C# 8+
    switch (opcion)
    {
        case "1": ListarProductos();       break;
        case "2": AgregarProducto();       break;
        case "3": EditarProducto();        break;
        case "4": EliminarProducto();      break;
        case "5": BuscarPorCategoria();    break;
        case "6": MostrarStockBajo();      break;
        case "0": salir = true;            break;
        default:
            MostrarError("Opción no válida."); break;
    }
}

Console.WriteLine("\n  ¡Hasta luego!\n");

// ── Funciones locales ───────────────────────────────────────────────────────
// C# permite definir funciones dentro de Program.cs (top-level statements).
// Son visibles solo dentro de este archivo.

void MostrarEncabezado()
{
    Console.ForegroundColor = ConsoleColor.Cyan;
    Console.WriteLine("╔══════════════════════════════╗");
    Console.WriteLine("║    INVENTARIO DE TIENDA      ║");
    Console.WriteLine("╚══════════════════════════════╝");
    Console.ResetColor();
    Console.WriteLine();
}

void MostrarError(string mensaje)
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine($"\n  ✗ {mensaje}");
    Console.ResetColor();
    Console.Write("  Presiona Enter para continuar...");
    Console.ReadLine();
}

void MostrarExito(string mensaje)
{
    Console.ForegroundColor = ConsoleColor.Green;
    Console.WriteLine($"\n  ✔ {mensaje}");
    Console.ResetColor();
    Console.Write("  Presiona Enter para continuar...");
    Console.ReadLine();
}

void ListarProductos()
{
    Console.Clear();
    MostrarEncabezado();

    var productos = servicio.ListarProductos();

    if (productos.Count == 0)
    {
        Console.WriteLine("  No hay productos registrados.");
    }
    else
    {
        // Formato de tabla simple con cadenas alineadas
        Console.WriteLine($"  {"ID",-5} {"Nombre",-25} {"Categoría",-15} {"Precio",10} {"Stock",6}");
        Console.WriteLine($"  {new string('─', 65)}");

        foreach (var p in productos)
        {
            // Formatos numéricos: C2 = moneda con 2 decimales
            Console.WriteLine($"  {p.Id,-5} {p.Nombre,-25} {p.Categoria,-15} {p.Precio,10:C2} {p.Stock,6}");
        }
    }

    Console.Write("\n  Presiona Enter para continuar...");
    Console.ReadLine();
}

void AgregarProducto()
{
    Console.Clear();
    MostrarEncabezado();
    Console.WriteLine("  ── AGREGAR PRODUCTO ──\n");

    try
    {
        Console.Write("  Nombre    : ");
        string nombre = Console.ReadLine() ?? "";

        Console.Write("  Categoría : ");
        string categoria = Console.ReadLine() ?? "";

        Console.Write("  Precio    : ");
        // decimal.Parse lanzará FormatException si el usuario escribe texto
        decimal precio = decimal.Parse(Console.ReadLine() ?? "0");

        Console.Write("  Stock     : ");
        int stock = int.Parse(Console.ReadLine() ?? "0");

        var nuevo = servicio.AgregarProducto(nombre, categoria, precio, stock);
        MostrarExito($"Producto '{nuevo.Nombre}' agregado con Id {nuevo.Id}.");
    }
    catch (FormatException)
    {
        MostrarError("Precio o stock inválido. Usa solo números.");
    }
    catch (ArgumentException ex)
    {
        MostrarError(ex.Message);
    }
}

void EditarProducto()
{
    Console.Clear();
    MostrarEncabezado();
    Console.WriteLine("  ── EDITAR PRODUCTO ──\n");

    try
    {
        Console.Write("  Id del producto a editar: ");
        int id = int.Parse(Console.ReadLine() ?? "0");

        var existente = servicio.BuscarProducto(id);
        if (existente is null)
        {
            MostrarError($"No se encontró un producto con Id {id}.");
            return;  // Termina la función local anticipadamente
        }

        Console.WriteLine($"\n  Producto actual: {existente.Nombre} | {existente.Categoria} | {existente.Precio:C2} | Stock: {existente.Stock}");
        Console.WriteLine("  (Presiona Enter para mantener el valor actual)\n");

        Console.Write($"  Nombre [{existente.Nombre}]: ");
        string nombre = Console.ReadLine() ?? "";
        if (string.IsNullOrWhiteSpace(nombre)) nombre = existente.Nombre;

        Console.Write($"  Categoría [{existente.Categoria}]: ");
        string categoria = Console.ReadLine() ?? "";
        if (string.IsNullOrWhiteSpace(categoria)) categoria = existente.Categoria;

        Console.Write($"  Precio [{existente.Precio}]: ");
        string precioInput = Console.ReadLine() ?? "";
        decimal precio = string.IsNullOrWhiteSpace(precioInput)
            ? existente.Precio
            : decimal.Parse(precioInput);

        Console.Write($"  Stock [{existente.Stock}]: ");
        string stockInput = Console.ReadLine() ?? "";
        int stock = string.IsNullOrWhiteSpace(stockInput)
            ? existente.Stock
            : int.Parse(stockInput);

        servicio.ActualizarProducto(id, nombre, categoria, precio, stock);
        MostrarExito("Producto actualizado correctamente.");
    }
    catch (FormatException)
    {
        MostrarError("Valor numérico inválido.");
    }
    catch (KeyNotFoundException ex)
    {
        MostrarError(ex.Message);
    }
}

void EliminarProducto()
{
    Console.Clear();
    MostrarEncabezado();
    Console.WriteLine("  ── ELIMINAR PRODUCTO ──\n");

    try
    {
        Console.Write("  Id del producto a eliminar: ");
        int id = int.Parse(Console.ReadLine() ?? "0");

        var existente = servicio.BuscarProducto(id);
        if (existente is null)
        {
            MostrarError($"No se encontró un producto con Id {id}.");
            return;
        }

        Console.Write($"\n  ¿Eliminar '{existente.Nombre}'? (s/n): ");
        string confirmacion = Console.ReadLine() ?? "";

        if (confirmacion.ToLower() == "s")
        {
            servicio.EliminarProducto(id);
            MostrarExito("Producto eliminado.");
        }
        else
        {
            Console.WriteLine("\n  Operación cancelada.");
            Console.ReadLine();
        }
    }
    catch (FormatException)
    {
        MostrarError("Id inválido.");
    }
}

void BuscarPorCategoria()
{
    Console.Clear();
    MostrarEncabezado();
    Console.WriteLine("  ── BUSCAR POR CATEGORÍA ──\n");

    Console.Write("  Categoría: ");
    string categoria = Console.ReadLine() ?? "";

    var resultados = servicio.BuscarPorCategoria(categoria);

    if (resultados.Count == 0)
    {
        MostrarError($"No se encontraron productos en la categoría '{categoria}'.");
        return;
    }

    Console.WriteLine($"\n  Encontrados: {resultados.Count} producto(s)\n");
    Console.WriteLine($"  {"ID",-5} {"Nombre",-25} {"Precio",10} {"Stock",6}");
    Console.WriteLine($"  {new string('─', 50)}");

    foreach (var p in resultados)
        Console.WriteLine($"  {p.Id,-5} {p.Nombre,-25} {p.Precio,10:C2} {p.Stock,6}");

    Console.Write("\n  Presiona Enter para continuar...");
    Console.ReadLine();
}

void MostrarStockBajo()
{
    Console.Clear();
    MostrarEncabezado();
    Console.WriteLine("  ── PRODUCTOS CON STOCK BAJO ──\n");

    Console.Write("  Umbral de stock (Enter = 5): ");
    string umbralInput = Console.ReadLine() ?? "";
    int umbral = string.IsNullOrWhiteSpace(umbralInput) ? 5 : int.Parse(umbralInput);

    var productos = servicio.ObtenerStockBajo(umbral);

    if (productos.Count == 0)
    {
        Console.WriteLine($"  Todos los productos tienen stock mayor a {umbral}.");
    }
    else
    {
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine($"  ⚠ {productos.Count} producto(s) con stock ≤ {umbral}:\n");
        Console.ResetColor();

        foreach (var p in productos)
            Console.WriteLine($"  • {p.Nombre,-25} Stock: {p.Stock}");
    }

    Console.Write("\n  Presiona Enter para continuar...");
    Console.ReadLine();
}
```

**Puntos clave de `Program.cs`:**

- **Top-level statements** (C# 9+) — no se necesita una clase `Program` ni un método `Main` explícito. El compilador los genera automáticamente.
- **Funciones locales** — las funciones definidas después de las instrucciones de nivel superior son privadas a este archivo. Son el mecanismo correcto para organizar el código en `Program.cs`.
- **Manejo de excepciones por capas** — cada excepción tiene su tipo: `FormatException` para entradas numéricas inválidas, `ArgumentException` para reglas de negocio, `KeyNotFoundException` para IDs inexistentes.

---

## 8. Ejecutar el proyecto

```bash
# Desde la raíz del taller
cd src/Inventario.Consola
dotnet run
```

Deberías ver el menú principal. Prueba agregar algunos productos, listarlos y luego revisar el archivo `inventario.json` que se genera automáticamente junto al ejecutable.

Para depurar desde VS Code presiona `F5` — el C# Dev Kit detecta el proyecto automáticamente.

---

## 9. Diagrama de arquitectura del módulo

```
┌─────────────────────────────────────────────────┐
│                   Program.cs                     │
│            (UI de consola / menú)                │
└───────────────────┬─────────────────────────────┘
                    │ usa
                    ▼
┌─────────────────────────────────────────────────┐
│            InventarioService                     │
│          (lógica de negocio)                     │
│  • validaciones  • transformaciones  • consultas │
└───────────────────┬─────────────────────────────┘
                    │ usa interfaz IProductoRepository
                    ▼
┌─────────────────────────────────────────────────┐
│         JsonProductoRepository                   │
│        (persistencia en JSON)                    │
│  leer archivo → modificar lista → guardar        │
└─────────────────────────────────────────────────┘
                    │
                    ▼
           📄 inventario.json
```

---

## 10. Reto del módulo

Extiende la aplicación con las siguientes funcionalidades antes de pasar al módulo 2:

1. **Reporte de valor total**: calcula `Precio × Stock` de cada producto y muestra el valor total del inventario.
2. **Exportar a CSV**: agrega una opción que guarde la lista de productos en un archivo `.csv` compatible con Excel.
3. **Búsqueda por nombre**: implementa una búsqueda parcial (`Contains`) que encuentre productos cuyo nombre contenga el texto ingresado.

---

## Resumen

| Concepto | Dónde se usó |
|---|---|
| Clases y propiedades | `Producto.cs` |
| Interfaces | `IProductoRepository`, `IInventarioService` |
| Inyección de dependencias manual | Constructor de `InventarioService` |
| LINQ | `OrderBy`, `Where`, `FirstOrDefault` |
| Serialización JSON | `JsonProductoRepository` |
| Manejo de excepciones | `Program.cs`, `InventarioService` |
| Top-level statements | `Program.cs` |
| Funciones locales | `Program.cs` |

---

**Inicio del taller →** [README](https://github.com/juangamboaabarca/practica_dotnet/blob/main/README.md)

**Siguiente módulo →** [Módulo 2: Aplicación de Escritorio con WinForms](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-02-winforms.md)
