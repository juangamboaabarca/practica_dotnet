# Módulo 4 — Aplicación Multiplataforma con .NET MAUI
### Taller: Sistema de Inventario con C# y .NET 8

---

## Objetivo del módulo

Crear una aplicación móvil y de escritorio multiplataforma que consuma la API del módulo 3, usando **.NET MAUI** y el patrón **MVVM**. Al terminar:

- La misma app correrá en **Android**, **iOS** y **Windows** desde un único proyecto.
- Usarás el patrón MVVM para separar la lógica de la UI.
- La app consumirá la API REST del módulo 3 (`Inventario.Api`).
- Entenderás cómo se gestiona el estado en aplicaciones móviles.

> **Requisito**: tener la API del módulo 3 ejecutándose. El módulo 4 no persiste datos propios; los delega a la API.

---

## 1. Conceptos previos

| Término | Definición |
|---|---|
| **.NET MAUI** | Multi-platform App UI. Sucesor de Xamarin.Forms. Un proyecto → Android, iOS, Windows, macOS. |
| **XAML** | Lenguaje declarativo XML para definir interfaces de usuario. Separa el diseño del código. |
| **MVVM** | Model–View–ViewModel. La Vista (XAML) observa al ViewModel; el ViewModel llama a los Servicios. |
| **INotifyPropertyChanged** | Interfaz que notifica a la UI cuando una propiedad cambia (reactividad). |
| **ICommand** | Abstracción de una acción ejecutable desde la UI (equivale a un evento de botón). |
| **Data Binding** | Enlace declarativo entre propiedades del ViewModel y controles XAML. Sin código en el code-behind. |
| **HttpClient** | Cliente HTTP para consumir la API. En MAUI requiere configuración especial por plataforma. |

---

## 2. Preparar el entorno para MAUI

MAUI requiere cargas de trabajo adicionales en el SDK:

```bash
# Instalar las cargas de trabajo necesarias (solo una vez)
dotnet workload install maui

# Para desarrollo en Android específicamente
dotnet workload install android

# Verificar que todo esté instalado
dotnet workload list
```

En VS Code, instala la extensión **.NET MAUI** (busca `ms-dotnettools.dotnet-maui`).

---

## 3. Crear el proyecto MAUI

Desde la raíz del taller:

```bash
dotnet new maui --name Inventario.Movil --output src/Inventario.Movil
dotnet sln add src/Inventario.Movil/Inventario.Movil.csproj
```

Revisa el `csproj` generado — es único en .NET:

```xml
<!-- src/Inventario.Movil/Inventario.Movil.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>
      net8.0-android;net8.0-ios;net8.0-windows10.0.19041.0
    </TargetFrameworks>
    <!-- La misma base de código compila para 3 plataformas distintas -->

    <RootNamespace>Inventario.Movil</RootNamespace>
    <UseMaui>true</UseMaui>
    <SingleProject>true</SingleProject>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <!-- Metadatos de la app por plataforma -->
    <ApplicationTitle>Inventario</ApplicationTitle>
    <ApplicationId>com.taller.inventario</ApplicationId>
    <ApplicationVersion>1</ApplicationVersion>
  </PropertyGroup>
</Project>
```

---

## 4. Estructura del proyecto

```
Inventario.Movil/
├── Models/
│   └── Producto.cs              ← Copia ligera del modelo (sin dependencia de EF)
├── Services/
│   ├── IProductoApiService.cs   ← Contrato del cliente HTTP
│   └── ProductoApiService.cs    ← Implementación con HttpClient
├── ViewModels/
│   ├── BaseViewModel.cs         ← Base con INotifyPropertyChanged
│   ├── InventarioViewModel.cs   ← Lógica de la lista principal
│   └── ProductoViewModel.cs     ← Lógica del formulario
├── Views/
│   ├── InventarioPage.xaml      ← Lista de productos
│   ├── InventarioPage.xaml.cs
│   ├── ProductoPage.xaml        ← Formulario agregar/editar
│   └── ProductoPage.xaml.cs
├── MauiProgram.cs               ← Punto de entrada y configuración DI
└── AppShell.xaml                ← Navegación de la app
```

---

## 5. Modelo local — `Models/Producto.cs`

En MAUI definimos una copia ligera del modelo sin dependencias externas:

```csharp
// src/Inventario.Movil/Models/Producto.cs

namespace Inventario.Movil.Models;

/// <summary>
/// Versión ligera del modelo Producto para el cliente móvil.
/// No depende de EF Core ni de System.Text.Json directamente.
/// </summary>
public class Producto
{
    public int     Id           { get; set; }
    public string  Nombre       { get; set; } = string.Empty;
    public string  Categoria    { get; set; } = string.Empty;
    public decimal Precio       { get; set; }
    public int     Stock        { get; set; }
    public DateTime FechaRegistro { get; set; }
}
```

---

## 6. Servicio HTTP — `Services/ProductoApiService.cs`

### 6.1 Interfaz

```csharp
// src/Inventario.Movil/Services/IProductoApiService.cs

using Inventario.Movil.Models;

namespace Inventario.Movil.Services;

public interface IProductoApiService
{
    Task<List<Producto>> ObtenerTodosAsync();
    Task<Producto?>      ObtenerPorIdAsync(int id);
    Task<bool>           AgregarAsync(ProductoDto dto);
    Task<bool>           ActualizarAsync(int id, ProductoDto dto);
    Task<bool>           EliminarAsync(int id);
}

// DTO para enviar datos a la API
public record ProductoDto(string Nombre, string Categoria, decimal Precio, int Stock);
```

### 6.2 Implementación

```csharp
// src/Inventario.Movil/Services/ProductoApiService.cs

using System.Net.Http.Json;
using Inventario.Movil.Models;

namespace Inventario.Movil.Services;

public class ProductoApiService : IProductoApiService
{
    private readonly HttpClient _httpClient;
    private const string EndpointBase = "api/productos";

    public ProductoApiService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<List<Producto>> ObtenerTodosAsync()
    {
        try
        {
            // GetFromJsonAsync deserializa automáticamente el JSON de la respuesta
            return await _httpClient.GetFromJsonAsync<List<Producto>>(EndpointBase)
                   ?? new List<Producto>();
        }
        catch (Exception ex)
        {
            // En una app real registrarías el error en un servicio de logs (ej: AppCenter)
            Console.WriteLine($"Error al obtener productos: {ex.Message}");
            return new List<Producto>();
        }
    }

    public async Task<Producto?> ObtenerPorIdAsync(int id) =>
        await _httpClient.GetFromJsonAsync<Producto>($"{EndpointBase}/{id}");

    public async Task<bool> AgregarAsync(ProductoDto dto)
    {
        var response = await _httpClient.PostAsJsonAsync(EndpointBase, dto);
        return response.IsSuccessStatusCode;
    }

    public async Task<bool> ActualizarAsync(int id, ProductoDto dto)
    {
        var response = await _httpClient.PutAsJsonAsync($"{EndpointBase}/{id}", dto);
        return response.IsSuccessStatusCode;
    }

    public async Task<bool> EliminarAsync(int id)
    {
        var response = await _httpClient.DeleteAsync($"{EndpointBase}/{id}");
        return response.IsSuccessStatusCode;
    }
}
```

---

## 7. ViewModel base — `ViewModels/BaseViewModel.cs`

```csharp
// src/Inventario.Movil/ViewModels/BaseViewModel.cs

using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace Inventario.Movil.ViewModels;

/// <summary>
/// Clase base para todos los ViewModels.
/// Implementa INotifyPropertyChanged para que la UI reaccione
/// automáticamente cuando las propiedades cambian.
/// </summary>
public class BaseViewModel : INotifyPropertyChanged
{
    // El evento que la UI (XAML) escucha para saber cuándo re-renderizar
    public event PropertyChangedEventHandler? PropertyChanged;

    /// <summary>
    /// Notifica a la UI que la propiedad con el nombre dado cambió.
    /// [CallerMemberName] rellena automáticamente el nombre del método que llama.
    /// </summary>
    protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

    /// <summary>
    /// Asigna el valor y notifica el cambio solo si el valor realmente cambió.
    /// Evita re-renders innecesarios.
    /// </summary>
    protected bool SetProperty<T>(ref T field, T value,
        [CallerMemberName] string? propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }

    // ── Propiedades comunes a todos los ViewModels ─────────────────────────

    private bool _ocupado;
    /// <summary>
    /// Indica si hay una operación async en curso. La UI puede enlazarse a esto
    /// para mostrar un indicador de carga y deshabilitar controles.
    /// </summary>
    public bool Ocupado
    {
        get => _ocupado;
        set => SetProperty(ref _ocupado, value);
    }

    private string _titulo = "";
    public string Titulo
    {
        get => _titulo;
        set => SetProperty(ref _titulo, value);
    }
}
```

**Puntos clave:**

- `INotifyPropertyChanged` es el mecanismo de reactividad en .NET. Sin él, la UI no sabe cuándo actualizar sus controles.
- `SetProperty` es el patrón estándar: compara, asigna y notifica en un solo método genérico.
- `[CallerMemberName]` es un atributo mágico: el compilador reemplaza el parámetro con el nombre de la propiedad que llamó al método, evitando escribir el nombre como string (frágil ante refactorizaciones).

---

## 8. ViewModel de inventario — `ViewModels/InventarioViewModel.cs`

```csharp
// src/Inventario.Movil/ViewModels/InventarioViewModel.cs

using System.Collections.ObjectModel;
using System.Windows.Input;
using Inventario.Movil.Models;
using Inventario.Movil.Services;

namespace Inventario.Movil.ViewModels;

/// <summary>
/// ViewModel de la pantalla principal de inventario.
/// Expone:
///   - La colección de productos (observable)
///   - Comandos para cargar, navegar y eliminar
///   - Estado de carga y mensajes de error
/// </summary>
public class InventarioViewModel : BaseViewModel
{
    private readonly IProductoApiService _apiService;

    // ObservableCollection notifica a la UI cuando se agregan, quitan o reordenan elementos.
    // Es el reemplazo de List<T> para colecciones enlazadas a UI en MAUI/WPF.
    public ObservableCollection<Producto> Productos { get; } = new();

    private string _mensajeError = "";
    public string MensajeError
    {
        get => _mensajeError;
        set => SetProperty(ref _mensajeError, value);
    }

    private bool _hayError;
    public bool HayError
    {
        get => _hayError;
        set => SetProperty(ref _hayError, value);
    }

    // ICommand es la abstracción de un botón/acción en MVVM.
    // La Vista enlaza Command="{Binding CargarCommand}" sin necesitar event handlers.
    public ICommand CargarCommand    { get; }
    public ICommand AgregarCommand   { get; }
    public ICommand EditarCommand    { get; }
    public ICommand EliminarCommand  { get; }

    public InventarioViewModel(IProductoApiService apiService)
    {
        _apiService = apiService;
        Titulo = "Inventario de Tienda";

        // Command con async: el patrón estándar es lambda async que llama al método privado
        CargarCommand   = new Command(async () => await CargarProductosAsync());
        AgregarCommand  = new Command(async () => await NavegarAgregarAsync());
        EditarCommand   = new Command<Producto>(async p => await NavegarEditarAsync(p));
        EliminarCommand = new Command<Producto>(async p => await EliminarProductoAsync(p));
    }

    /// <summary>
    /// Carga los productos desde la API y actualiza la colección observable.
    /// </summary>
    public async Task CargarProductosAsync()
    {
        if (Ocupado) return;  // Evita llamadas concurrentes

        Ocupado   = true;
        HayError  = false;
        MensajeError = "";

        try
        {
            var lista = await _apiService.ObtenerTodosAsync();

            // Limpiar y repoblar la ObservableCollection
            // (no reemplazamos la referencia para mantener el binding activo)
            Productos.Clear();
            foreach (var p in lista)
                Productos.Add(p);
        }
        catch (Exception ex)
        {
            MensajeError = $"No se pudo conectar a la API: {ex.Message}";
            HayError = true;
        }
        finally
        {
            // finally garantiza que Ocupado vuelva a false aunque haya excepción
            Ocupado = false;
        }
    }

    private async Task NavegarAgregarAsync()
    {
        // Shell.Current.GoToAsync navega usando rutas declaradas en AppShell.xaml
        // Pasamos el parámetro "id=0" para indicar que es alta (nuevo producto)
        await Shell.Current.GoToAsync($"producto?id=0");
    }

    private async Task NavegarEditarAsync(Producto producto)
    {
        await Shell.Current.GoToAsync($"producto?id={producto.Id}");
    }

    private async Task EliminarProductoAsync(Producto producto)
    {
        // DisplayAlert es el equivalente de MessageBox en MAUI
        bool confirmar = await Shell.Current.DisplayAlert(
            "Confirmar",
            $"¿Eliminar '{producto.Nombre}'?",
            "Sí", "No");

        if (!confirmar) return;

        bool exito = await _apiService.EliminarAsync(producto.Id);

        if (exito)
        {
            // Eliminar del ObservableCollection actualiza la UI inmediatamente
            Productos.Remove(producto);
        }
        else
        {
            await Shell.Current.DisplayAlert("Error", "No se pudo eliminar el producto.", "OK");
        }
    }
}
```

---

## 9. ViewModel de producto — `ViewModels/ProductoViewModel.cs`

```csharp
// src/Inventario.Movil/ViewModels/ProductoViewModel.cs

using System.Windows.Input;
using Inventario.Movil.Models;
using Inventario.Movil.Services;

namespace Inventario.Movil.ViewModels;

/// <summary>
/// ViewModel para la pantalla de alta/edición de producto.
/// Implementa IQueryAttributable para recibir el parámetro de navegación "id".
/// </summary>
public class ProductoViewModel : BaseViewModel, IQueryAttributable
{
    private readonly IProductoApiService _apiService;

    // Propiedades del formulario: cada una notifica al binding cuando cambia
    private int _id;
    private string _nombre    = "";
    private string _categoria = "";
    private decimal _precio;
    private int _stock;
    private string _mensajeError = "";
    private bool _hayError;

    public int     Id        { get => _id;       set => SetProperty(ref _id, value);       }
    public string  Nombre    { get => _nombre;   set => SetProperty(ref _nombre, value);   }
    public string  Categoria { get => _categoria;set => SetProperty(ref _categoria, value);}
    public decimal Precio    { get => _precio;   set => SetProperty(ref _precio, value);   }
    public int     Stock     { get => _stock;    set => SetProperty(ref _stock, value);    }
    public string  MensajeError { get => _mensajeError; set => SetProperty(ref _mensajeError, value); }
    public bool    HayError     { get => _hayError;     set => SetProperty(ref _hayError, value);     }

    public ICommand GuardarCommand  { get; }
    public ICommand CancelarCommand { get; }

    public ProductoViewModel(IProductoApiService apiService)
    {
        _apiService    = apiService;
        GuardarCommand  = new Command(async () => await GuardarAsync(), PuedeGuardar);
        CancelarCommand = new Command(async () => await Shell.Current.GoToAsync(".."));
        // ".." = ruta relativa: volver a la página anterior en el stack de navegación
    }

    /// <summary>
    /// IQueryAttributable.ApplyQueryAttributes: MAUI llama este método al navegar
    /// hacia esta página pasando parámetros en la URL (ej: producto?id=5).
    /// </summary>
    public async void ApplyQueryAttributes(IDictionary<string, object> query)
    {
        if (query.TryGetValue("id", out var idObj) &&
            int.TryParse(idObj?.ToString(), out int id) && id > 0)
        {
            // Modo edición: cargar datos del producto desde la API
            Titulo = "Editar Producto";
            var producto = await _apiService.ObtenerPorIdAsync(id);

            if (producto is not null)
            {
                Id        = producto.Id;
                Nombre    = producto.Nombre;
                Categoria = producto.Categoria;
                Precio    = producto.Precio;
                Stock     = producto.Stock;
            }
        }
        else
        {
            // Modo alta: formulario en blanco
            Titulo = "Agregar Producto";
        }
    }

    private bool PuedeGuardar() =>
        !string.IsNullOrWhiteSpace(Nombre) && Precio >= 0 && Stock >= 0;

    private async Task GuardarAsync()
    {
        if (Ocupado) return;

        Ocupado  = true;
        HayError = false;

        try
        {
            var dto = new ProductoDto(Nombre, Categoria, Precio, Stock);
            bool exito;

            if (Id > 0)
                exito = await _apiService.ActualizarAsync(Id, dto);
            else
                exito = await _apiService.AgregarAsync(dto);

            if (exito)
                await Shell.Current.GoToAsync("..");  // Volver y la lista se recargará
            else
            {
                MensajeError = "No se pudo guardar. Verifica la conexión con la API.";
                HayError = true;
            }
        }
        catch (Exception ex)
        {
            MensajeError = ex.Message;
            HayError = true;
        }
        finally
        {
            Ocupado = false;
        }
    }
}
```

---

## 10. Pantalla de inventario — `Views/InventarioPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- src/Inventario.Movil/Views/InventarioPage.xaml -->

<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:Inventario.Movil.ViewModels"
             x:Class="Inventario.Movil.Views.InventarioPage"
             Title="{Binding Titulo}">

    <!--
        BindingContext conecta la página con su ViewModel.
        Cuando el ViewModel llama OnPropertyChanged, la UI se actualiza.
    -->

    <!-- ActivityIndicator: spinner visible cuando Ocupado=true -->
    <ContentPage.ToolbarItems>
        <ToolbarItem Text="➕ Agregar"
                     Command="{Binding AgregarCommand}" />
    </ContentPage.ToolbarItems>

    <Grid RowDefinitions="Auto,*,Auto">

        <!-- Fila 0: Indicador de carga -->
        <ActivityIndicator Grid.Row="0"
                           IsRunning="{Binding Ocupado}"
                           IsVisible="{Binding Ocupado}"
                           Color="#0078d4"
                           Margin="0,10" />

        <!-- Fila 0 también: Mensaje de error (si lo hay) -->
        <Label Grid.Row="0"
               Text="{Binding MensajeError}"
               TextColor="Red"
               IsVisible="{Binding HayError}"
               HorizontalOptions="Center"
               Margin="10,10" />

        <!-- Fila 1: Lista de productos con CollectionView -->
        <CollectionView Grid.Row="1"
                        ItemsSource="{Binding Productos}"
                        SelectionMode="None">

            <CollectionView.EmptyView>
                <Label Text="No hay productos. Toca ➕ para agregar uno."
                       HorizontalOptions="Center"
                       VerticalOptions="Center"
                       TextColor="Gray" />
            </CollectionView.EmptyView>

            <CollectionView.ItemTemplate>
                <DataTemplate>
                    <!--
                        Cada elemento de la colección se renderiza con este template.
                        El contexto de binding aquí es un Producto individual.
                    -->
                    <SwipeView>
                        <!-- Acciones al deslizar hacia la izquierda -->
                        <SwipeView.RightItems>
                            <SwipeItems>
                                <SwipeItem Text="Editar"
                                           BackgroundColor="#0078d4"
                                           Command="{Binding Source={RelativeSource AncestorType={x:Type vm:InventarioViewModel}},
                                                             Path=EditarCommand}"
                                           CommandParameter="{Binding .}" />
                                <SwipeItem Text="Eliminar"
                                           BackgroundColor="Red"
                                           Command="{Binding Source={RelativeSource AncestorType={x:Type vm:InventarioViewModel}},
                                                             Path=EliminarCommand}"
                                           CommandParameter="{Binding .}" />
                            </SwipeItems>
                        </SwipeView.RightItems>

                        <!-- Contenido principal del item -->
                        <Grid Padding="15,10"
                              ColumnDefinitions="*,Auto"
                              BackgroundColor="White">

                            <!-- Columna izquierda: nombre y categoría -->
                            <StackLayout Grid.Column="0" Spacing="3">
                                <Label Text="{Binding Nombre}"
                                       FontSize="16"
                                       FontAttributes="Bold" />
                                <Label Text="{Binding Categoria}"
                                       FontSize="13"
                                       TextColor="Gray" />
                            </StackLayout>

                            <!-- Columna derecha: precio y stock -->
                            <StackLayout Grid.Column="1"
                                         HorizontalOptions="End"
                                         Spacing="3">
                                <Label Text="{Binding Precio, StringFormat='{0:C2}'}"
                                       FontSize="15"
                                       FontAttributes="Bold"
                                       TextColor="#198754"
                                       HorizontalOptions="End" />
                                <Label FontSize="13"
                                       HorizontalOptions="End">
                                    <Label.FormattedText>
                                        <FormattedString>
                                            <Span Text="Stock: " TextColor="Gray" />
                                            <Span Text="{Binding Stock}"
                                                  TextColor="{Binding Stock,
                                                    Converter={StaticResource StockColorConverter}}" />
                                        </FormattedString>
                                    </Label.FormattedText>
                                </Label>
                            </StackLayout>
                        </Grid>
                    </SwipeView>
                </DataTemplate>
            </CollectionView.ItemTemplate>
        </CollectionView>

    </Grid>
</ContentPage>
```

### Code-behind — `Views/InventarioPage.xaml.cs`

```csharp
// src/Inventario.Movil/Views/InventarioPage.xaml.cs

using Inventario.Movil.ViewModels;

namespace Inventario.Movil.Views;

public partial class InventarioPage : ContentPage
{
    private readonly InventarioViewModel _viewModel;

    public InventarioPage(InventarioViewModel viewModel)
    {
        InitializeComponent();

        // BindingContext conecta la página XAML con el ViewModel
        BindingContext = _viewModel = viewModel;
    }

    // OnAppearing se llama cada vez que la página se muestra
    // (también al volver desde ProductoPage después de guardar)
    protected override async void OnAppearing()
    {
        base.OnAppearing();
        await _viewModel.CargarProductosAsync();
    }
}
```

---

## 11. Pantalla de producto — `Views/ProductoPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- src/Inventario.Movil/Views/ProductoPage.xaml -->

<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="Inventario.Movil.Views.ProductoPage"
             Title="{Binding Titulo}">

    <ScrollView>
        <StackLayout Padding="20" Spacing="15">

            <!-- Campo: Nombre -->
            <Label Text="Nombre" FontAttributes="Bold" />
            <Entry Text="{Binding Nombre}"
                   Placeholder="Ej: Laptop Pro 15"
                   ReturnType="Next" />

            <!-- Campo: Categoría -->
            <Label Text="Categoría" FontAttributes="Bold" />
            <Entry Text="{Binding Categoria}"
                   Placeholder="Ej: Electrónica"
                   ReturnType="Next" />

            <!-- Campo: Precio -->
            <Label Text="Precio" FontAttributes="Bold" />
            <Entry Text="{Binding Precio}"
                   Keyboard="Numeric"
                   Placeholder="0.00"
                   ReturnType="Next" />

            <!-- Campo: Stock -->
            <Label Text="Stock" FontAttributes="Bold" />
            <Entry Text="{Binding Stock}"
                   Keyboard="Numeric"
                   Placeholder="0"
                   ReturnType="Done" />

            <!-- Mensaje de error -->
            <Label Text="{Binding MensajeError}"
                   TextColor="Red"
                   IsVisible="{Binding HayError}"
                   FontSize="13" />

            <!-- Botones de acción -->
            <Grid ColumnDefinitions="*,*" ColumnSpacing="10">
                <Button Grid.Column="0"
                        Text="Cancelar"
                        Command="{Binding CancelarCommand}"
                        BackgroundColor="#6c757d"
                        TextColor="White" />
                <Button Grid.Column="1"
                        Text="Guardar"
                        Command="{Binding GuardarCommand}"
                        BackgroundColor="#198754"
                        TextColor="White"
                        IsEnabled="{Binding Ocupado, Converter={StaticResource InvertedBoolConverter}}" />
            </Grid>

            <!-- Indicador de guardado en proceso -->
            <ActivityIndicator IsRunning="{Binding Ocupado}"
                               IsVisible="{Binding Ocupado}"
                               Color="#0078d4" />

        </StackLayout>
    </ScrollView>
</ContentPage>
```

### Code-behind — `Views/ProductoPage.xaml.cs`

```csharp
// src/Inventario.Movil/Views/ProductoPage.xaml.cs

using Inventario.Movil.ViewModels;

namespace Inventario.Movil.Views;

public partial class ProductoPage : ContentPage
{
    public ProductoPage(ProductoViewModel viewModel)
    {
        InitializeComponent();
        BindingContext = viewModel;
    }
}
```

---

## 12. Navegación — `AppShell.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- src/Inventario.Movil/AppShell.xaml -->

<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:Inventario.Movil.Views"
       x:Class="Inventario.Movil.AppShell"
       Title="Inventario">

    <!--
        ShellContent define la página raíz de la aplicación.
        ContentTemplate usa DataTemplate para crear la página bajo demanda (lazy loading).
    -->
    <ShellContent
        Title="Inventario"
        ContentTemplate="{DataTemplate views:InventarioPage}"
        Route="inventario" />

</Shell>
```

### Code-behind — `AppShell.xaml.cs`

```csharp
// src/Inventario.Movil/AppShell.xaml.cs

using Inventario.Movil.Views;

namespace Inventario.Movil;

public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();

        // Registrar la ruta de ProductoPage para poder navegar con GoToAsync("producto?id=5")
        // Las rutas no declaradas en XAML deben registrarse aquí
        Routing.RegisterRoute("producto", typeof(ProductoPage));
    }
}
```

---

## 13. Punto de entrada — `MauiProgram.cs`

```csharp
// src/Inventario.Movil/MauiProgram.cs

using Inventario.Movil.Services;
using Inventario.Movil.ViewModels;
using Inventario.Movil.Views;
using Microsoft.Extensions.Logging;

namespace Inventario.Movil;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();

        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
                fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
            });

        // ── Registrar servicios ───────────────────────────────────────────

        // HttpClient con la URL base de la API
        // En Android el emulador usa 10.0.2.2 como alias del localhost del host
        builder.Services.AddHttpClient<IProductoApiService, ProductoApiService>(client =>
        {
#if ANDROID
            // Android emulator: 10.0.2.2 = localhost del PC donde corre la API
            client.BaseAddress = new Uri("http://10.0.2.2:5100/");
#else
            // Windows y iOS (dispositivo físico en la misma red)
            client.BaseAddress = new Uri("https://localhost:7100/");
#endif
        });

        // Registrar ViewModels como Transient (nueva instancia cada vez que se navega a la página)
        builder.Services.AddTransient<InventarioViewModel>();
        builder.Services.AddTransient<ProductoViewModel>();

        // Registrar las páginas (Views) también en DI para que MAUI inyecte los ViewModels
        builder.Services.AddTransient<InventarioPage>();
        builder.Services.AddTransient<ProductoPage>();

        // Shell se registra como Singleton (una sola instancia durante la vida de la app)
        builder.Services.AddSingleton<AppShell>();

#if DEBUG
        builder.Logging.AddDebug();
#endif

        return builder.Build();
    }
}
```

**Puntos clave:**

- `#if ANDROID` — directivas de compilación condicional: el código entre estas directivas solo se compila para esa plataforma. La misma base de código, comportamiento diferente por plataforma.
- `AddTransient` vs `AddSingleton` — los ViewModels son Transient porque cada página necesita una instancia fresca con estado limpio. El Shell es Singleton porque es el contenedor de toda la navegación.
- MAUI usa el mismo contenedor de DI de ASP.NET Core, aplicado al mundo móvil.

---

## 14. Converters — `Converters/StockColorConverter.cs`

Los converters transforman valores para la presentación en XAML sin lógica en el code-behind:

```csharp
// src/Inventario.Movil/Converters/StockColorConverter.cs

using System.Globalization;

namespace Inventario.Movil.Converters;

/// <summary>
/// Convierte un valor de stock en un color:
/// rojo si es bajo (≤ 5), verde si es suficiente.
/// Se usa en el binding de Color del Label de stock.
/// </summary>
public class StockColorConverter : IValueConverter
{
    // Convert: llamado al leer el valor (ViewModel → Vista)
    public object Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (value is int stock)
            return stock <= 5 ? Colors.Red : Colors.Green;
        return Colors.Black;
    }

    // ConvertBack: llamado en bindings bidireccionales (raramente necesario en converters de color)
    public object ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotImplementedException();
}
```

Registra el converter en `App.xaml` o en el `ResourceDictionary` de cada página:

```xml
<!-- En App.xaml o en el ResourceDictionary de la página -->
<Application.Resources>
    <ResourceDictionary>
        <converters:StockColorConverter x:Key="StockColorConverter" />
        <converters:InvertedBoolConverter x:Key="InvertedBoolConverter" />
    </ResourceDictionary>
</Application.Resources>
```

---

## 15. Ejecutar en diferentes plataformas

```bash
cd src/Inventario.Movil

# Ejecutar en Windows (inmediato, sin emulador)
dotnet run -f net8.0-windows10.0.19041.0

# Ejecutar en Android (requiere Android SDK y emulador o dispositivo)
# Primero lista los emuladores disponibles
dotnet build -t:Run -f net8.0-android

# Ver dispositivos conectados
dotnet build -t:_GetAndroidDevices -f net8.0-android
```

Para depurar en VS Code con el emulador de Android:

1. Asegúrate de que el emulador esté corriendo (Android Studio → AVD Manager).
2. Abre la paleta de comandos (`Ctrl+Shift+P`) y escribe `.NET MAUI: Pick Android device`.
3. Presiona `F5`.

---

## 16. Diagrama completo del taller

```
┌──────────────────────────────────────────────────────────────┐
│              Inventario.Movil (.NET MAUI)                     │
│   InventarioPage  ←→  InventarioViewModel                    │
│   ProductoPage    ←→  ProductoViewModel                      │
│              ↓  HttpClient                                    │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTP/REST (JSON)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│              Inventario.Api (ASP.NET Core)                    │
│   ProductosController → InventarioService                    │
│                       → EfProductoRepository                 │
└──────────────────────┬───────────────────────────────────────┘
                       │ EF Core
                       ▼
                 🗄️ inventario.db (SQLite)

────────── Módulos independientes ──────────────────────────────

 Inventario.Consola   Inventario.WinForms
 (Módulo 1)           (Módulo 2)
 ↕ comparten          ↕ reutiliza
 JsonProductoRepository  InventarioService
```

---

## 17. Reto del módulo

1. **Búsqueda local**: agrega un `SearchBar` en `InventarioPage` que filtre la `ObservableCollection` en tiempo real sin llamar a la API.
2. **Caché offline**: guarda la lista en `Preferences` (almacenamiento clave-valor de MAUI) para mostrar datos aunque no haya conexión.
3. **Notificación de stock bajo**: al cargar la lista, si hay productos con stock ≤ 5, muestra un banner amarillo con la cuenta.

---

## Resumen comparativo de los 4 módulos

| Módulo | Proyecto | UI | Datos | Patrón |
|---|---|---|---|---|
| 1 | `Inventario.Consola` | Consola | JSON en archivo | — |
| 2 | `Inventario.WinForms` | WinForms | JSON en archivo (reutilizado) | Event-driven |
| 3 | `Inventario.Api` + `Inventario.Web` | Blazor + REST | SQLite (EF Core) | MVC / Component |
| 4 | `Inventario.Movil` | MAUI / XAML | API REST (módulo 3) | MVVM |

---

**Inicio del taller →** [README](https://github.com/juangamboaabarca/practica_dotnet/blob/main/README.md)

**← Módulo anterior**: [Módulo 3: Web con ASP.NET Core](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-03-web.md)

---

## Lectura recomendada

- [Documentación oficial .NET MAUI](https://learn.microsoft.com/dotnet/maui/)
- [Patrones MVVM en MAUI](https://learn.microsoft.com/dotnet/maui/xaml/fundamentals/mvvm)
- [Community Toolkit MAUI](https://learn.microsoft.com/dotnet/communitytoolkit/maui/) — simplifica MVVM con `[ObservableProperty]` y `[RelayCommand]`
- [CommunityToolkit.Mvvm](https://learn.microsoft.com/dotnet/communitytoolkit/mvvm/) — generadores de código para reducir el boilerplate de `INotifyPropertyChanged`
