# Módulo 2 — Aplicación de Escritorio con WinForms
### Taller: Sistema de Inventario con C# y .NET 8

---

## Objetivo del módulo

Migrar la lógica de negocio del módulo 1 a una interfaz gráfica de escritorio usando **Windows Forms (WinForms)**. Al terminar:

- Tendrás una ventana principal con una tabla de productos.
- Podrás agregar, editar y eliminar desde formularios visuales.
- La lógica de negocio (`InventarioService`) y el repositorio JSON serán **exactamente los mismos** del módulo 1, sin modificar.
- Entenderás la separación entre capa de presentación y capa de negocio.

> **Requisito**: haber completado el Módulo 1. Las clases `Producto`, `InventarioService` y `JsonProductoRepository` se usan directamente.

---

## 1. Entender WinForms en .NET 8

Windows Forms es el framework de UI de escritorio más antiguo de .NET. Aunque solo corre en **Windows**, sigue siendo ampliamente usado en aplicaciones empresariales internas. Sus ventajas:

- Curva de aprendizaje muy baja.
- Controles ricos listos para usar (`DataGridView`, `TextBox`, `ComboBox`, etc.).
- Soporte completo en .NET 8.

Para aplicaciones multiplataforma se usará MAUI en el módulo 4.

---

## 2. Crear el proyecto WinForms

Desde la raíz del taller:

```bash
# Crear el proyecto WinForms
dotnet new winforms --name Inventario.WinForms --output src/Inventario.WinForms

# Agregarlo a la solución
dotnet sln add src/Inventario.WinForms/Inventario.WinForms.csproj
```

### 2.1 Agregar referencia al proyecto de consola

En lugar de copiar el código, **referenciamos** el proyecto del módulo 1 directamente. Así comparten el mismo `InventarioService` y `JsonProductoRepository`.

```bash
cd src/Inventario.WinForms

dotnet add reference ../Inventario.Consola/Inventario.Consola.csproj
```

Verifica el archivo `.csproj`:

```xml
<!-- src/Inventario.WinForms/Inventario.WinForms.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <!-- OutputType WindowsExe = aplicación con ventana (sin consola visible) -->
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <!-- UseWindowsForms habilita el framework de UI -->
    <UseWindowsForms>true</UseWindowsForms>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- Referencia al proyecto base con los modelos y servicios -->
    <ProjectReference Include="..\Inventario.Consola\Inventario.Consola.csproj" />
  </ItemGroup>
</Project>
```

**Puntos clave:**
- `net8.0-windows` — el sufijo `-windows` habilita APIs exclusivas de Windows (WinForms, WPF, registros del sistema, etc.).
- `UseWindowsForms` — activa las clases de `System.Windows.Forms`.
- La referencia al proyecto de consola evita duplicar código: es **reutilización real entre capas**.

---

## 3. Estructura del proyecto WinForms

```
Inventario.WinForms/
├── Forms/
│   ├── FormPrincipal.cs       ← Ventana principal con tabla
│   ├── FormPrincipal.Designer.cs  ← Diseño generado (no editar manualmente)
│   ├── FormProducto.cs        ← Formulario agregar/editar
│   └── FormProducto.Designer.cs
├── Program.cs                 ← Punto de entrada WinForms
└── Inventario.WinForms.csproj
```

> En WinForms cada ventana (`Form`) tiene dos archivos: el `.cs` con la lógica y el `.Designer.cs` con el diseño de controles. En VS Code el diseño se escribe a mano (no hay diseñador visual como en Visual Studio); esto es más explícito y educativo.

---

## 4. Punto de entrada — `Program.cs`

```csharp
// src/Inventario.WinForms/Program.cs

using Inventario.Consola.Repositories;
using Inventario.Consola.Services;
using Inventario.WinForms.Forms;

// ApplicationConfiguration.Initialize() configura DPI y estilos visuales modernos.
// Es el reemplazo moderno de Application.EnableVisualStyles() + SetCompatibleTextRenderingDefault()
ApplicationConfiguration.Initialize();

// Composición de dependencias: igual que en el módulo 1
var repositorio = new JsonProductoRepository();
var servicio    = new InventarioService(repositorio);

// Application.Run lanza el bucle de mensajes de Windows y muestra la ventana principal.
// Cuando el usuario cierra FormPrincipal, el bucle termina y el proceso finaliza.
Application.Run(new FormPrincipal(servicio));
```

**Puntos clave:**
- El ciclo de vida de una aplicación WinForms se basa en un **bucle de mensajes**: Windows envía eventos (clics, teclado, redraws) que el runtime procesa continuamente.
- `Application.Run` no retorna hasta que el formulario principal se cierre.
- El servicio se crea una sola vez y se pasa al formulario: patrón de inyección de dependencias.

---

## 5. Formulario principal — `FormPrincipal`

### 5.1 Diseño — `FormPrincipal.Designer.cs`

Este archivo define los controles y su posición. En VS Code lo escribimos manualmente:

```csharp
// src/Inventario.WinForms/Forms/FormPrincipal.Designer.cs

namespace Inventario.WinForms.Forms;

partial class FormPrincipal
{
    // IContainer es requerido por el patrón Designer para manejar el ciclo de vida de componentes.
    private System.ComponentModel.IContainer? components = null;

    // Declaración de controles: cada campo corresponde a un elemento visual.
    private DataGridView dgvProductos = null!;
    private Button       btnAgregar   = null!;
    private Button       btnEditar    = null!;
    private Button       btnEliminar  = null!;
    private Button       btnRefrescar = null!;
    private Label        lblTitulo    = null!;
    private Panel        panelBotones = null!;
    private StatusStrip  statusStrip  = null!;
    private ToolStripStatusLabel lblEstado = null!;

    /// <summary>
    /// Inicializa y configura todos los controles del formulario.
    /// Este método es llamado por el constructor de FormPrincipal.
    /// </summary>
    private void InitializeComponent()
    {
        // Instanciar controles
        lblTitulo    = new Label();
        panelBotones = new Panel();
        btnAgregar   = new Button();
        btnEditar    = new Button();
        btnEliminar  = new Button();
        btnRefrescar = new Button();
        dgvProductos = new DataGridView();
        statusStrip  = new StatusStrip();
        lblEstado    = new ToolStripStatusLabel();

        // ── Configuración del formulario principal ────────────────────────
        Text            = "Inventario de Tienda";
        Size            = new Size(900, 600);
        MinimumSize     = new Size(700, 450);
        StartPosition   = FormStartPosition.CenterScreen;  // Centra al abrirse
        BackColor       = Color.WhiteSmoke;

        // ── Label de título ───────────────────────────────────────────────
        lblTitulo.Text      = "Gestión de Inventario";
        lblTitulo.Font      = new Font("Segoe UI", 16, FontStyle.Bold);
        lblTitulo.ForeColor = Color.FromArgb(0, 120, 215);  // Azul Windows
        lblTitulo.Dock      = DockStyle.Top;                 // Se extiende al ancho
        lblTitulo.Height    = 50;
        lblTitulo.TextAlign = ContentAlignment.MiddleCenter;

        // ── Panel lateral de botones ──────────────────────────────────────
        // DockStyle.Right ancla el panel al borde derecho; se redimensiona con la ventana.
        panelBotones.Dock      = DockStyle.Right;
        panelBotones.Width     = 130;
        panelBotones.Padding   = new Padding(10);
        panelBotones.BackColor = Color.FromArgb(240, 240, 240);

        // Función auxiliar local para estandarizar el estilo de botones
        void ConfigurarBoton(Button btn, string texto, Color color, int top)
        {
            btn.Text      = texto;
            btn.BackColor = color;
            btn.ForeColor = Color.White;
            btn.FlatStyle = FlatStyle.Flat;  // Sin borde 3D clásico
            btn.FlatAppearance.BorderSize = 0;
            btn.Font      = new Font("Segoe UI", 10);
            btn.Size      = new Size(110, 40);
            btn.Location  = new Point(10, top);
            btn.Cursor    = Cursors.Hand;
        }

        ConfigurarBoton(btnAgregar,   "➕ Agregar",   Color.FromArgb(0, 153, 0),   10);
        ConfigurarBoton(btnEditar,    "✏️ Editar",    Color.FromArgb(0, 102, 204),  60);
        ConfigurarBoton(btnEliminar,  "🗑️ Eliminar",  Color.FromArgb(204, 0, 0),   110);
        ConfigurarBoton(btnRefrescar, "🔄 Refrescar", Color.FromArgb(100, 100, 100),170);

        panelBotones.Controls.AddRange(
            new Control[] { btnAgregar, btnEditar, btnEliminar, btnRefrescar });

        // ── DataGridView (tabla de productos) ─────────────────────────────
        // Dock.Fill: ocupa todo el espacio restante del formulario
        dgvProductos.Dock                  = DockStyle.Fill;
        dgvProductos.ReadOnly              = true;   // Edición solo desde el formulario dedicado
        dgvProductos.SelectionMode         = DataGridViewSelectionMode.FullRowSelect;
        dgvProductos.MultiSelect           = false;
        dgvProductos.AllowUserToAddRows    = false;
        dgvProductos.AllowUserToDeleteRows = false;
        dgvProductos.AutoSizeColumnsMode   = DataGridViewAutoSizeColumnsMode.Fill;
        dgvProductos.BackgroundColor       = Color.White;
        dgvProductos.BorderStyle           = BorderStyle.None;
        dgvProductos.ColumnHeadersHeightSizeMode =
            DataGridViewColumnHeadersHeightSizeMode.AutoSize;

        // Estilo del encabezado de columnas
        dgvProductos.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(0, 120, 215);
        dgvProductos.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
        dgvProductos.ColumnHeadersDefaultCellStyle.Font      = new Font("Segoe UI", 10, FontStyle.Bold);
        dgvProductos.EnableHeadersVisualStyles = false;  // Necesario para aplicar el estilo personalizado

        // Estilo de filas alternas para mejor legibilidad
        dgvProductos.AlternatingRowsDefaultCellStyle.BackColor = Color.FromArgb(245, 245, 255);

        // ── Barra de estado ───────────────────────────────────────────────
        lblEstado.Text = "Listo";
        statusStrip.Items.Add(lblEstado);

        // ── Agregar controles al formulario ───────────────────────────────
        // El orden importa cuando se usan Dock: primero los que "recortan" espacio (Right, Top, Bottom)
        // y al final el Fill que ocupa lo que quede.
        Controls.Add(dgvProductos);  // Fill → va primero en Controls pero se procesa al final
        Controls.Add(panelBotones);
        Controls.Add(lblTitulo);
        Controls.Add(statusStrip);
    }

    // Libera los recursos administrados cuando el formulario se destruye.
    protected override void Dispose(bool disposing)
    {
        if (disposing && (components != null))
            components.Dispose();
        base.Dispose(disposing);
    }
}
```

### 5.2 Lógica — `FormPrincipal.cs`

```csharp
// src/Inventario.WinForms/Forms/FormPrincipal.cs

using Inventario.Consola.Models;
using Inventario.Consola.Services;

namespace Inventario.WinForms.Forms;

/// <summary>
/// Ventana principal de la aplicación. Muestra el listado de productos
/// y permite navegar a las acciones de alta, edición y baja.
/// </summary>
public partial class FormPrincipal : Form
{
    private readonly IInventarioService _servicio;

    // Constructor: recibe el servicio inyectado desde Program.cs
    public FormPrincipal(IInventarioService servicio)
    {
        _servicio = servicio;
        InitializeComponent();  // Construye la UI definida en el .Designer.cs

        // Suscribir los manejadores de eventos de los botones
        // El operador += conecta el evento con el método que lo maneja
        btnAgregar.Click   += BtnAgregar_Click;
        btnEditar.Click    += BtnEditar_Click;
        btnEliminar.Click  += BtnEliminar_Click;
        btnRefrescar.Click += (_, _) => CargarProductos();

        // Doble clic en la tabla abre el editor
        dgvProductos.CellDoubleClick += DgvProductos_CellDoubleClick;
    }

    // Load se dispara cuando el formulario termina de construirse y está listo para mostrarse
    protected override void OnLoad(EventArgs e)
    {
        base.OnLoad(e);
        ConfigurarColumnas();
        CargarProductos();
    }

    /// <summary>
    /// Define las columnas del DataGridView y las enlaza a las propiedades del modelo.
    /// </summary>
    private void ConfigurarColumnas()
    {
        dgvProductos.Columns.Clear();

        // DataPropertyName vincula la columna con una propiedad del objeto enlazado
        dgvProductos.Columns.AddRange(new DataGridViewColumn[]
        {
            new DataGridViewTextBoxColumn { DataPropertyName = "Id",           HeaderText = "ID",        Width = 50  },
            new DataGridViewTextBoxColumn { DataPropertyName = "Nombre",       HeaderText = "Nombre",    FillWeight = 30 },
            new DataGridViewTextBoxColumn { DataPropertyName = "Categoria",    HeaderText = "Categoría", FillWeight = 20 },
            new DataGridViewTextBoxColumn { DataPropertyName = "Precio",       HeaderText = "Precio",    FillWeight = 15,
                DefaultCellStyle = new DataGridViewCellStyle { Format = "C2", Alignment = DataGridViewContentAlignment.MiddleRight } },
            new DataGridViewTextBoxColumn { DataPropertyName = "Stock",        HeaderText = "Stock",     FillWeight = 10,
                DefaultCellStyle = new DataGridViewCellStyle { Alignment = DataGridViewContentAlignment.MiddleCenter } },
            new DataGridViewTextBoxColumn { DataPropertyName = "FechaRegistro",HeaderText = "Registrado",FillWeight = 20,
                DefaultCellStyle = new DataGridViewCellStyle { Format = "dd/MM/yyyy" } },
        });
    }

    /// <summary>
    /// Obtiene los productos del servicio y los enlaza al DataGridView.
    /// DataSource con BindingList permite que los cambios en la lista
    /// se reflejen automáticamente en la tabla.
    /// </summary>
    private void CargarProductos()
    {
        var productos = _servicio.ListarProductos();

        // BindingSource es el puente entre la fuente de datos y el control visual.
        // Permite filtrar, navegar y actualizar sin acceder al control directamente.
        var bindingSource = new BindingSource { DataSource = productos };
        dgvProductos.DataSource = bindingSource;

        ActualizarEstado(productos.Count);
    }

    private void ActualizarEstado(int total)
    {
        lblEstado.Text = $"Total: {total} producto(s)";
    }

    /// <summary>
    /// Devuelve el Id del producto seleccionado en la tabla, o null si no hay selección.
    /// </summary>
    private int? ObtenerIdSeleccionado()
    {
        if (dgvProductos.SelectedRows.Count == 0) return null;

        // Cells["Id"] accede a la celda por el nombre de la propiedad vinculada
        var celda = dgvProductos.SelectedRows[0].Cells["Id"].Value;
        return celda is int id ? id : null;
    }

    // ── Manejadores de eventos ──────────────────────────────────────────────
    // Los nombres siguen la convención NombreControl_NombreEvento

    private void BtnAgregar_Click(object? sender, EventArgs e)
    {
        // Se crea el formulario de producto sin producto (modo Alta)
        using var form = new FormProducto(_servicio, null);

        // ShowDialog muestra el formulario como modal (bloquea la ventana padre)
        // y retorna el resultado cuando se cierra.
        if (form.ShowDialog() == DialogResult.OK)
            CargarProductos();  // Refresca la tabla solo si se guardó
    }

    private void BtnEditar_Click(object? sender, EventArgs e)
    {
        int? id = ObtenerIdSeleccionado();
        if (id is null)
        {
            MessageBox.Show("Selecciona un producto para editar.",
                "Atención", MessageBoxButtons.OK, MessageBoxIcon.Warning);
            return;
        }

        var producto = _servicio.BuscarProducto(id.Value);
        if (producto is null) return;

        using var form = new FormProducto(_servicio, producto);
        if (form.ShowDialog() == DialogResult.OK)
            CargarProductos();
    }

    private void BtnEliminar_Click(object? sender, EventArgs e)
    {
        int? id = ObtenerIdSeleccionado();
        if (id is null)
        {
            MessageBox.Show("Selecciona un producto para eliminar.",
                "Atención", MessageBoxButtons.OK, MessageBoxIcon.Warning);
            return;
        }

        var producto = _servicio.BuscarProducto(id.Value);
        if (producto is null) return;

        var confirmacion = MessageBox.Show(
            $"¿Deseas eliminar '{producto.Nombre}'?",
            "Confirmar eliminación",
            MessageBoxButtons.YesNo,
            MessageBoxIcon.Question);  // Ícono de pregunta

        if (confirmacion == DialogResult.Yes)
        {
            try
            {
                _servicio.EliminarProducto(id.Value);
                CargarProductos();
                lblEstado.Text = $"Producto '{producto.Nombre}' eliminado.";
            }
            catch (Exception ex)
            {
                MessageBox.Show(ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }

    private void DgvProductos_CellDoubleClick(object? sender, DataGridViewCellEventArgs e)
    {
        // e.RowIndex == -1 indica que se hizo clic en el encabezado de columna
        if (e.RowIndex >= 0)
            BtnEditar_Click(sender, EventArgs.Empty);
    }
}
```

---

## 6. Formulario de producto — `FormProducto`

Este formulario sirve tanto para **agregar** (cuando `producto` es `null`) como para **editar** (cuando se pasa un `Producto` existente).

### 6.1 Diseño — `FormProducto.Designer.cs`

```csharp
// src/Inventario.WinForms/Forms/FormProducto.Designer.cs

namespace Inventario.WinForms.Forms;

partial class FormProducto
{
    private System.ComponentModel.IContainer? components = null;

    private Label   lblNombre    = null!;
    private Label   lblCategoria = null!;
    private Label   lblPrecio    = null!;
    private Label   lblStock     = null!;
    private TextBox txtNombre    = null!;
    private TextBox txtCategoria = null!;
    private TextBox txtPrecio    = null!;
    private NumericUpDown nudStock = null!;
    private Button  btnGuardar   = null!;
    private Button  btnCancelar  = null!;
    private TableLayoutPanel tableLayout = null!;

    private void InitializeComponent()
    {
        lblNombre    = new Label();
        lblCategoria = new Label();
        lblPrecio    = new Label();
        lblStock     = new Label();
        txtNombre    = new TextBox();
        txtCategoria = new TextBox();
        txtPrecio    = new TextBox();
        nudStock     = new NumericUpDown();
        btnGuardar   = new Button();
        btnCancelar  = new Button();
        tableLayout  = new TableLayoutPanel();

        // ── Formulario ────────────────────────────────────────────────────
        Text            = "Producto";
        Size            = new Size(400, 310);
        FormBorderStyle = FormBorderStyle.FixedDialog;  // Sin redimensionar
        MaximizeBox     = false;
        MinimizeBox     = false;
        StartPosition   = FormStartPosition.CenterParent;  // Centrado sobre la ventana padre
        Padding         = new Padding(15);

        // ── TableLayoutPanel: diseño de cuadrícula ─────────────────────────
        // Organiza los controles en filas y columnas, evitando posicionamiento manual.
        tableLayout.Dock        = DockStyle.Fill;
        tableLayout.ColumnCount = 2;
        tableLayout.RowCount    = 6;
        tableLayout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 100));
        tableLayout.ColumnStyles.Add(new ColumnStyle(SizeType.Percent,  100));
        for (int i = 0; i < 5; i++)
            tableLayout.RowStyles.Add(new RowStyle(SizeType.Absolute, 45));
        tableLayout.RowStyles.Add(new RowStyle(SizeType.Absolute, 50)); // fila botones

        void ConfigLabel(Label lbl, string texto)
        {
            lbl.Text      = texto;
            lbl.Dock      = DockStyle.Fill;
            lbl.TextAlign = ContentAlignment.MiddleRight;
            lbl.Font      = new Font("Segoe UI", 10);
        }

        void ConfigInput(Control ctrl)
        {
            ctrl.Dock = DockStyle.Fill;
            ctrl.Font = new Font("Segoe UI", 10);
            if (ctrl is TextBox tb) tb.BorderStyle = BorderStyle.FixedSingle;
        }

        ConfigLabel(lblNombre,    "Nombre:");
        ConfigLabel(lblCategoria, "Categoría:");
        ConfigLabel(lblPrecio,    "Precio:");
        ConfigLabel(lblStock,     "Stock:");

        ConfigInput(txtNombre);
        ConfigInput(txtCategoria);
        ConfigInput(txtPrecio);

        // NumericUpDown: control numérico con flechas de incremento
        nudStock.Dock     = DockStyle.Fill;
        nudStock.Minimum  = 0;
        nudStock.Maximum  = 99999;
        nudStock.Font     = new Font("Segoe UI", 10);

        // Botones de acción
        btnGuardar.Text      = "Guardar";
        btnGuardar.BackColor = Color.FromArgb(0, 153, 0);
        btnGuardar.ForeColor = Color.White;
        btnGuardar.FlatStyle = FlatStyle.Flat;
        btnGuardar.FlatAppearance.BorderSize = 0;
        btnGuardar.Font      = new Font("Segoe UI", 10);
        btnGuardar.Dock      = DockStyle.Right;
        btnGuardar.Width     = 100;

        btnCancelar.Text      = "Cancelar";
        btnCancelar.BackColor = Color.FromArgb(180, 180, 180);
        btnCancelar.ForeColor = Color.White;
        btnCancelar.FlatStyle = FlatStyle.Flat;
        btnCancelar.FlatAppearance.BorderSize = 0;
        btnCancelar.Font      = new Font("Segoe UI", 10);
        btnCancelar.Dock      = DockStyle.Left;
        btnCancelar.Width     = 100;

        // AcceptButton = se activa con Enter | CancelButton = se activa con Escape
        AcceptButton = btnGuardar;
        CancelButton = btnCancelar;

        // Agregar controles a la cuadrícula: (control, columna, fila)
        tableLayout.Controls.Add(lblNombre,    0, 0);
        tableLayout.Controls.Add(txtNombre,    1, 0);
        tableLayout.Controls.Add(lblCategoria, 0, 1);
        tableLayout.Controls.Add(txtCategoria, 1, 1);
        tableLayout.Controls.Add(lblPrecio,    0, 2);
        tableLayout.Controls.Add(txtPrecio,    1, 2);
        tableLayout.Controls.Add(lblStock,     0, 3);
        tableLayout.Controls.Add(nudStock,     1, 3);
        tableLayout.Controls.Add(btnCancelar,  0, 5);
        tableLayout.Controls.Add(btnGuardar,   1, 5);

        Controls.Add(tableLayout);
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing && (components != null))
            components.Dispose();
        base.Dispose(disposing);
    }
}
```

### 6.2 Lógica — `FormProducto.cs`

```csharp
// src/Inventario.WinForms/Forms/FormProducto.cs

using Inventario.Consola.Models;
using Inventario.Consola.Services;

namespace Inventario.WinForms.Forms;

/// <summary>
/// Formulario para agregar o editar un producto.
/// Si 'productoExistente' es null → modo Alta.
/// Si 'productoExistente' tiene valor → modo Edición.
/// </summary>
public partial class FormProducto : Form
{
    private readonly IInventarioService _servicio;
    private readonly Producto? _productoExistente;

    public FormProducto(IInventarioService servicio, Producto? productoExistente)
    {
        _servicio           = servicio;
        _productoExistente  = productoExistente;

        InitializeComponent();

        // Suscribir eventos
        btnGuardar.Click  += BtnGuardar_Click;
        btnCancelar.Click += (_, _) =>
        {
            DialogResult = DialogResult.Cancel;  // Señala al padre que se canceló
            Close();
        };
    }

    protected override void OnLoad(EventArgs e)
    {
        base.OnLoad(e);

        if (_productoExistente is not null)
        {
            // Modo edición: pre-llenar los campos con los datos actuales
            Text             = "Editar Producto";
            txtNombre.Text   = _productoExistente.Nombre;
            txtCategoria.Text = _productoExistente.Categoria;
            txtPrecio.Text   = _productoExistente.Precio.ToString("F2");
            nudStock.Value   = _productoExistente.Stock;
        }
        else
        {
            Text = "Agregar Producto";
        }

        // Colocar el foco en el primer campo al abrir el formulario
        txtNombre.Focus();
    }

    private void BtnGuardar_Click(object? sender, EventArgs e)
    {
        // Validación simple en el formulario antes de llamar al servicio
        if (string.IsNullOrWhiteSpace(txtNombre.Text))
        {
            MessageBox.Show("El nombre es obligatorio.", "Validación",
                MessageBoxButtons.OK, MessageBoxIcon.Warning);
            txtNombre.Focus();
            return;
        }

        if (!decimal.TryParse(txtPrecio.Text, out decimal precio) || precio < 0)
        {
            MessageBox.Show("Precio inválido. Ingresa un número mayor o igual a 0.",
                "Validación", MessageBoxButtons.OK, MessageBoxIcon.Warning);
            txtPrecio.Focus();
            return;
        }

        try
        {
            if (_productoExistente is null)
            {
                // Alta: llamar al servicio con los datos del formulario
                _servicio.AgregarProducto(
                    txtNombre.Text,
                    txtCategoria.Text,
                    precio,
                    (int)nudStock.Value);
            }
            else
            {
                // Edición: actualizar el producto existente
                _servicio.ActualizarProducto(
                    _productoExistente.Id,
                    txtNombre.Text,
                    txtCategoria.Text,
                    precio,
                    (int)nudStock.Value);
            }

            // DialogResult.OK le indica al formulario padre que la operación fue exitosa
            DialogResult = DialogResult.OK;
            Close();
        }
        catch (Exception ex)
        {
            MessageBox.Show(ex.Message, "Error al guardar",
                MessageBoxButtons.OK, MessageBoxIcon.Error);
        }
    }
}
```

---

## 7. Ejecutar el proyecto

```bash
cd src/Inventario.WinForms
dotnet run
```

Para depurar desde VS Code, abre el archivo `launch.json` (`.vscode/launch.json`) y agrega una configuración para el proyecto WinForms:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "WinForms: Inventario",
      "type": "coreclr",
      "request": "launch",
      "program": "${workspaceFolder}/src/Inventario.WinForms/bin/Debug/net8.0-windows/Inventario.WinForms.exe",
      "args": [],
      "cwd": "${workspaceFolder}/src/Inventario.WinForms",
      "stopAtEntry": false,
      "console": "internalConsole",
      "preLaunchTask": "build-winforms"
    }
  ]
}
```

Crea `.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build-winforms",
      "command": "dotnet",
      "type": "process",
      "args": ["build", "${workspaceFolder}/src/Inventario.WinForms/Inventario.WinForms.csproj"],
      "problemMatcher": "$msCompile"
    }
  ]
}
```

---

## 8. Diagrama de arquitectura del módulo

```
┌─────────────────────────────────────────────────┐
│               FormPrincipal                      │
│   DataGridView  │  Botones Agregar/Editar/Eliminar│
└───────┬─────────────────────┬───────────────────┘
        │ abre (ShowDialog)   │ usa
        ▼                     ▼
┌───────────────┐   ┌─────────────────────────────┐
│  FormProducto │   │      IInventarioService      │
│  (Alta/Edición│   │   (misma del módulo 1)       │
└───────────────┘   └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   JsonProductoRepository     │
                    │   (misma del módulo 1)       │
                    └──────────────┬──────────────┘
                                   │
                            📄 inventario.json
```

**Dato clave**: los dos bloques inferiores son exactamente los mismos archivos del módulo 1. Solo cambia la capa de presentación.

---

## 9. Conceptos WinForms aprendidos

| Concepto | Descripción |
|---|---|
| `Form` | Ventana base; todo el UI vive dentro de un Form |
| `Control.Dock` | Ancla un control a un borde o rellena el espacio disponible |
| `DataGridView` | Tabla de datos con enlace a colecciones |
| `BindingSource` | Puente entre datos y controles visuales |
| `ShowDialog()` | Muestra un Form modal y retorna un `DialogResult` |
| `Events (+= / handler)` | Modelo de eventos para responder a interacciones del usuario |
| `.Designer.cs` | Convención para separar el diseño de la lógica |
| `AcceptButton` / `CancelButton` | Botones activados con Enter/Escape |

---

## 10. Reto del módulo

1. **Panel de estadísticas**: agrega encima del `DataGridView` un panel pequeño que muestre el total de productos, el valor total del inventario y cuántos tienen stock bajo.
2. **Filtro en tiempo real**: coloca un `TextBox` de búsqueda que filtre la tabla mientras el usuario escribe (usa el evento `TextChanged` y un `BindingSource` con `Filter`).
3. **Exportar a CSV**: agrega un botón "Exportar" que guarde la lista actual en un `.csv`.

---

**Inicio del taller →** [README](https://github.com/juangamboaabarca/practica_dotnet/blob/main/README.md)

**← Módulo anterior**: [Módulo 1: Consola](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-01-consola.md)  
**Siguiente módulo →**: [Módulo 3: Aplicación Web con ASP.NET Core](https://github.com/juangamboaabarca/practica_dotnet/blob/main/modulo-03-web.md)
