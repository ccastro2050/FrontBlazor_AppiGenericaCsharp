# Tutorial: Vistas Maestro-Detalle en Blazor Server (.razor)

Guia completa para construir paginas `.razor` con vistas integradas de maestro-detalle en Blazor Server, usando el proyecto `FrontBlazor_AppiGenericaCsharp` como referencia.

---

## Tabla de Contenido

1. [Fundamentos](#1-fundamentos)
2. [Prerequisitos y Estructura del Proyecto](#2-prerequisitos-y-estructura-del-proyecto)
3. [Anatomia de una Pagina CRUD Simple](#3-anatomia-de-una-pagina-crud-simple)
4. [Diseno de la Pagina Maestro-Detalle](#4-diseno-de-la-pagina-maestro-detalle)
5. [Paso a Paso: Construir Factura.razor](#5-paso-a-paso-construir-facturarazor)
6. [Patrones Clave Explicados](#6-patrones-clave-explicados)
7. [Otros Ejemplos de Maestro-Detalle](#7-otros-ejemplos-de-maestro-detalle)
8. [Errores Comunes y Soluciones](#8-errores-comunes-y-soluciones)
9. [Equivalencia Flask vs Blazor](#9-equivalencia-flask-vs-blazor)

---

## 1. Fundamentos

### Relacion Maestro-Detalle

Una relacion maestro-detalle conecta un registro principal (maestro) con multiples registros secundarios (detalle) que dependen de el. El registro maestro existe de forma independiente, pero los registros de detalle solo tienen sentido dentro del contexto del maestro.

**Ejemplo concreto:** Una **Factura** (maestro) contiene uno o mas **Productos por Factura** (detalle). La factura tiene datos generales (cliente, vendedor, fecha, total) y cada linea de producto tiene datos especificos (codigo producto, cantidad, valor unitario, subtotal).

### Ejemplos de Relaciones Maestro-Detalle

| Maestro | Detalle | Descripcion |
|---------|---------|-------------|
| **Factura** | Productos por Factura | Cada factura tiene N productos con cantidad y subtotal |
| Pedido | Items del Pedido | Cada pedido tiene N items solicitados |
| Orden de Compra | Detalle de Compra | Cada orden tiene N materiales/servicios |
| Receta Medica | Medicamentos | Cada receta tiene N medicamentos con dosis |
| Matricula | Materias Inscritas | Cada matricula tiene N materias con horarios |
| Orden de Trabajo | Tareas | Cada orden tiene N tareas asignadas |
| Presupuesto | Partidas | Cada presupuesto tiene N partidas con montos |

### CRUD Simple vs Maestro-Detalle

| Aspecto | CRUD Simple | Maestro-Detalle |
|---------|-------------|-----------------|
| Vistas | Lista + Formulario (toggle) | Lista + Detalle + Formulario (3 vistas) |
| Datos | Una sola tabla | Tabla maestra + tabla detalle (anidada) |
| Formulario | Campos escalares | Campos escalares + filas dinamicas |
| Persistencia | CRUD directo (POST, PUT, DELETE) | Stored procedures (transaccional) |
| JSON | No se usa | JSON como puente para datos anidados |
| Complejidad | Baja | Media-Alta |

---

## 2. Prerequisitos y Estructura del Proyecto

### Que se Necesita

1. **API REST** corriendo (ApiGenericaCsharp en puerto 5035)
2. **Base de datos** con las tablas maestro/detalle y stored procedures
3. **Proyecto Blazor Server** con servicios configurados

### Estructura de Archivos

```
FrontBlazor_AppiGenericaCsharp/
├── Components/
│   ├── Layout/
│   │   └── NavMenu.razor          ← Menu lateral con links
│   └── Pages/
│       ├── Producto.razor          ← CRUD simple (referencia)
│       ├── Cliente.razor           ← CRUD con FKs (referencia)
│       └── Factura.razor           ← Maestro-Detalle (objetivo)
├── Services/
│   ├── ApiService.cs               ← CRUD generico (GET, POST, PUT, DELETE)
│   └── SpService.cs                ← Stored procedures (POST ejecutarsp)
├── Program.cs                      ← Registro de servicios e inyeccion
├── appsettings.json                ← URL de la API
└── appsettings.Development.json    ← URL de la API (desarrollo)
```

### Dos Servicios, Dos Propositos

**`ApiService.cs`** — Para operaciones CRUD simples (una tabla a la vez):
```csharp
// Listar todos los registros de una tabla
List<Dictionary<string, object?>> registros = await Api.ListarAsync("producto");

// Crear un registro
var (exito, mensaje) = await Api.CrearAsync("producto", datos);

// Actualizar un registro
var (exito, mensaje) = await Api.ActualizarAsync("producto", "codigo", "PR001", datos);

// Eliminar un registro
var (exito, mensaje) = await Api.EliminarAsync("producto", "codigo", "PR001");
```

**`SpService.cs`** — Para operaciones complejas via stored procedures:
```csharp
// Ejecutar un SP con parametros
var (exito, resultados, mensaje) = await Sp.EjecutarSpAsync(
    "sp_listar_facturas_y_productosporfactura",
    new Dictionary<string, object?> { ["p_resultado"] = null }
);
```

Los SPs manejan la transaccionalidad: insertar la factura Y sus productos en una sola operacion atomica.

### Registro en Program.cs

```csharp
// Leer URL de la API desde configuracion
var apiBaseUrl = builder.Configuration["ApiBaseUrl"] ?? "http://localhost:5035";

// Registrar HttpClient con la URL base
builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri(apiBaseUrl)
});

// Registrar ambos servicios
builder.Services.AddScoped<FrontBlazor_AppiGenericaCsharp.Services.ApiService>();
builder.Services.AddScoped<FrontBlazor_AppiGenericaCsharp.Services.SpService>();
```

---

## 3. Anatomia de una Pagina CRUD Simple

Antes de construir maestro-detalle, es importante entender el patron CRUD simple. Tomemos `Producto.razor` como referencia.

### Estructura General

```
┌──────────────────────────────────┐
│  @page + @rendermode + @inject   │  ← Directivas
├──────────────────────────────────┤
│  HTML/Razor (vista)              │  ← Interfaz de usuario
│  ├── Alerta de mensaje           │
│  ├── Boton "Nuevo"               │
│  ├── Formulario (crear/editar)   │
│  └── Tabla de registros          │
├──────────────────────────────────┤
│  @code { ... }                   │  ← Logica C#
│  ├── Variables de estado         │
│  ├── OnInitializedAsync()        │
│  ├── CargarRegistros()           │
│  ├── NuevoRegistro()             │
│  ├── EditarRegistro()            │
│  ├── GuardarRegistro()           │
│  └── EliminarRegistro()          │
└──────────────────────────────────┘
```

### Variables de Estado (CRUD Simple)

```csharp
@code {
    // Lista de registros de la tabla
    private List<Dictionary<string, object?>> registros = new();

    // Controla visibilidad del formulario
    private bool mostrarFormulario = false;

    // true = editando, false = creando
    private bool editando = false;

    // Mensaje de alerta y su tipo
    private string mensaje = string.Empty;
    private bool exito = false;

    // Campos del formulario (uno por cada columna de la tabla)
    private string campoCodigo = string.Empty;
    private string campoNombre = string.Empty;
    private int campoStock = 0;
    private double campoValor = 0;
}
```

### Toggle Lista/Formulario

El CRUD simple usa un `bool` para alternar entre lista y formulario:

```html
@* Solo mostrar boton "Nuevo" cuando NO hay formulario visible *@
@if (!mostrarFormulario)
{
    <button class="btn btn-primary mb-3" @onclick="NuevoRegistro">Nuevo Producto</button>
}

@* Formulario: solo visible cuando mostrarFormulario = true *@
@if (mostrarFormulario)
{
    <div class="card mb-3">
        <div class="card-header">
            @(editando ? "Editar Producto" : "Nuevo Producto")
        </div>
        <div class="card-body">
            @* Campos del formulario *@
            <button class="btn btn-success" @onclick="GuardarRegistro">Guardar</button>
            <button class="btn btn-secondary" @onclick="Cancelar">Cancelar</button>
        </div>
    </div>
}

@* Tabla: siempre visible *@
@if (registros.Any())
{
    <table class="table table-striped">
        @* ... filas ... *@
    </table>
}
```

Este patron funciona bien para tablas simples con campos escalares. Pero cuando necesitamos mostrar un **detalle anidado** (como los productos de una factura), necesitamos el patron maestro-detalle.

---

## 4. Diseno de la Pagina Maestro-Detalle

### El Concepto de "Vistas" con String

En lugar de un simple `bool mostrarFormulario`, usamos un `string vista` que puede tener 3 valores:

```csharp
private string vista = "listar";  // Valor inicial
```

| Vista | Contenido | Cuando se muestra |
|-------|-----------|-------------------|
| `"listar"` | Tabla de facturas con resumen | Al cargar la pagina, despues de guardar/eliminar |
| `"ver"` | Detalle completo: cabecera + productos | Al hacer clic en "Ver" |
| `"formulario"` | Formulario con filas dinamicas de productos | Al hacer clic en "Nueva" o "Editar" |

```html
@if (vista == "listar")
{
    @* TABLA DE FACTURAS *@
}

@if (vista == "ver" && facturaActual != null)
{
    @* DETALLE DE UNA FACTURA + PRODUCTOS *@
}

@if (vista == "formulario")
{
    @* FORMULARIO CREAR/EDITAR CON FILAS DINAMICAS *@
}
```

### Diagrama de Navegacion

```
                    ┌─────────────┐
         ┌─────────│   LISTAR    │─────────┐
         │         │  (tabla)    │         │
         │         └──────┬──────┘         │
         │                │                │
    [Nueva]          [Ver]│[Editar]    [Eliminar]
         │                │                │
         ▼                ▼                │
┌─────────────┐    ┌─────────────┐         │
│ FORMULARIO  │    │     VER     │         │
│ (crear)     │    │  (detalle)  │         │
└──────┬──────┘    └──────┬──────┘         │
       │                  │                │
   [Guardar]         [Editar]              │
   [Cancelar]        [Volver]              │
       │                  │                │
       └──────────────────┴────────────────┘
                    (vuelve a LISTAR)
```

---

## 5. Paso a Paso: Construir Factura.razor

### Paso 1: Directivas, Inyecciones y Estructura Base

Todo archivo `.razor` comienza con las directivas en la parte superior:

```html
@page "/factura"
@rendermode InteractiveServer
@inject FrontBlazor_AppiGenericaCsharp.Services.ApiService Api
@inject FrontBlazor_AppiGenericaCsharp.Services.SpService Sp
@inject IJSRuntime JS
@using System.Text.Json

<PageTitle>Facturas</PageTitle>
```

**Cada directiva tiene un proposito:**

| Directiva | Funcion |
|-----------|---------|
| `@page "/factura"` | Define la URL donde se accede a esta pagina |
| `@rendermode InteractiveServer` | Habilita interactividad del lado del servidor (necesario para eventos como `@onclick`) |
| `@inject ... ApiService Api` | Inyecta el servicio CRUD generico (para cargar clientes, vendedores, productos) |
| `@inject ... SpService Sp` | Inyecta el servicio de stored procedures (para las operaciones de facturas) |
| `@inject IJSRuntime JS` | Inyecta el runtime de JavaScript (para `confirm()` antes de eliminar/actualizar) |
| `@using System.Text.Json` | Importa el namespace para serializar/deserializar JSON |

**`ApiService`** se usa para cargar datos de apoyo (tablas simples).
**`SpService`** se usa para las operaciones principales de facturas (que involucran maestro + detalle).
**`IJSRuntime`** permite llamar funciones JavaScript desde C# (como `confirm()`).

### Paso 2: Variables de Estado y Clases Auxiliares

Las variables de estado controlan todo el comportamiento de la pagina:

```csharp
@code {
    // ───────── ESTADO ─────────

    // Controla cual de las 3 vistas se muestra
    private string vista = "listar";

    // Muestra el spinner mientras carga datos
    private bool cargando = true;

    // Mensaje de alerta (exito o error)
    private string mensaje = string.Empty;
    private bool exito = false;

    // Lista de todas las facturas (para la vista "listar")
    private List<Dictionary<string, object?>> facturas = new();

    // Factura seleccionada (para las vistas "ver" y "formulario")
    private Dictionary<string, object?>? facturaActual;

    // ───────── CAMPOS DEL FORMULARIO ─────────

    // true cuando se esta editando una factura existente
    private bool editando = false;

    // Numero de la factura que se esta editando
    private int numeroEditar = 0;

    // Valores seleccionados en los dropdowns
    private int campoCliente = 0;
    private int campoVendedor = 0;

    // Lista de filas de productos (dinamica: se pueden agregar/quitar)
    private List<ProductoFila> filasProductos = new() { new() };

    // ───────── DATOS PARA SELECTS ─────────

    // Datos cargados desde la API para llenar los dropdowns
    private List<ClienteInfo> clientes = new();
    private List<VendedorInfo> vendedores = new();
    private List<Dictionary<string, object?>> productosDisponibles = new();
}
```

**Clases auxiliares** para tipar los datos de los selects y las filas dinamicas:

```csharp
// Representa una fila de producto en el formulario
// Solo necesita codigo y cantidad (el resto viene del select)
private class ProductoFila
{
    public string Codigo { get; set; } = "";
    public int Cantidad { get; set; } = 1;
}

// Datos del cliente para el select (nombre viene de cruzar con persona)
private class ClienteInfo
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public string Credito { get; set; } = "0";
}

// Datos del vendedor para el select (nombre viene de cruzar con persona)
private class VendedorInfo
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public string Carnet { get; set; } = "";
}
```

Estas clases se definen **dentro del bloque `@code`**, al mismo nivel que las variables de estado.

### Paso 3: Vista "listar" — Tabla de Facturas

La primera vista muestra todas las facturas en una tabla:

```html
@if (vista == "listar")
{
    <button class="btn btn-primary mb-3" @onclick="MostrarFormularioNueva">Nueva Factura</button>

    @* Spinner mientras carga *@
    @if (cargando)
    {
        <div class="d-flex justify-content-center my-4">
            <div class="spinner-border text-primary" role="status">
                <span class="visually-hidden">Cargando...</span>
            </div>
        </div>
    }
    else if (facturas.Any())
    {
        <table class="table table-striped table-hover">
            <thead class="table-dark">
                <tr>
                    <th>Numero</th>
                    <th>Cliente</th>
                    <th>Vendedor</th>
                    <th>Fecha</th>
                    <th>Total</th>
                    <th>Productos</th>
                    <th>Acciones</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var f in facturas)
                {
                    <tr>
                        <td>@ObtenerValor(f, "numero")</td>
                        <td>@ObtenerValor(f, "nombre_cliente")</td>
                        <td>@ObtenerValor(f, "nombre_vendedor")</td>
                        <td>@FormatearFecha(ObtenerValor(f, "fecha"))</td>
                        <td>@ObtenerValor(f, "total")</td>
                        <td>@ObtenerProductos(f).Count</td>
                        <td>
                            <button class="btn btn-info btn-sm me-1"
                                    @onclick="() => VerFactura(f)">Ver</button>
                            <button class="btn btn-warning btn-sm me-1"
                                    @onclick="() => EditarFactura(f)">Editar</button>
                            <button class="btn btn-danger btn-sm"
                                    @onclick="() => EliminarFactura(f)">Eliminar</button>
                        </td>
                    </tr>
                }
            </tbody>
        </table>
    }
    else
    {
        <div class="alert alert-warning">No se encontraron facturas.</div>
    }
}
```

**Puntos clave:**
- `ObtenerValor(f, "numero")` extrae un valor del diccionario de forma segura (retorna `""` si no existe)
- `ObtenerProductos(f).Count` parsea el JSON de productos anidado y cuenta cuantos hay
- Los botones de accion usan `@onclick="() => Metodo(f)"` con lambda para pasar el parametro
- El SP ya resuelve los nombres de cliente y vendedor (no hay que cruzar FKs aqui)

### Paso 4: Vista "ver" — Detalle de Factura

Al hacer clic en "Ver", se muestra la factura completa con sus productos:

```html
@if (vista == "ver" && facturaActual != null)
{
    <button class="btn btn-secondary mb-3 me-2" @onclick="Volver">Volver</button>
    <button class="btn btn-warning mb-3" @onclick="() => EditarFactura(facturaActual)">Editar</button>

    @* Cabecera de la factura (datos del maestro) *@
    <div class="card mb-3">
        <div class="card-header">
            <strong>Factura #@ObtenerValor(facturaActual, "numero")</strong>
        </div>
        <div class="card-body">
            <div class="row">
                <div class="col-md-6"><strong>Cliente:</strong> @ObtenerValor(facturaActual, "nombre_cliente")</div>
                <div class="col-md-6"><strong>Vendedor:</strong> @ObtenerValor(facturaActual, "nombre_vendedor")</div>
                <div class="col-md-6"><strong>Fecha:</strong> @FormatearFecha(ObtenerValor(facturaActual, "fecha"))</div>
                <div class="col-md-6"><strong>Total:</strong> @ObtenerValor(facturaActual, "total")</div>
            </div>
        </div>
    </div>

    @* Tabla de productos (datos del detalle) *@
    @if (ObtenerProductos(facturaActual).Any())
    {
        <h5>Productos</h5>
        <table class="table table-striped">
            <thead class="table-dark">
                <tr>
                    <th>Codigo</th>
                    <th>Nombre</th>
                    <th>Cantidad</th>
                    <th>Valor Unitario</th>
                    <th>Subtotal</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var p in ObtenerProductos(facturaActual))
                {
                    <tr>
                        <td>@ObtenerValor(p, "codigo_producto")</td>
                        <td>@ObtenerValor(p, "nombre_producto")</td>
                        <td>@ObtenerValor(p, "cantidad")</td>
                        <td>@ObtenerValor(p, "valorunitario")</td>
                        <td>@ObtenerValor(p, "subtotal")</td>
                    </tr>
                }
            </tbody>
        </table>
    }
}
```

**La estructura visual:**
```
┌─────────────────────────────────────────┐
│  [Volver]  [Editar]                     │
├─────────────────────────────────────────┤
│  Factura #1                             │  ← Card (maestro)
│  Cliente: Ana Torres                    │
│  Vendedor: Carlos Perez                 │
│  Fecha: 2025-12-03 12:57               │
│  Total: 5000000                         │
├─────────────────────────────────────────┤
│  Productos                              │  ← Tabla (detalle)
│  ┌────────┬──────────┬────┬───────┬────┐│
│  │Codigo  │Nombre    │Cant│V.Unit │Subt││
│  ├────────┼──────────┼────┼───────┼────┤│
│  │PR001   │Laptop    │ 2  │2500000│5M  ││
│  └────────┴──────────┴────┴───────┴────┘│
└─────────────────────────────────────────┘
```

### Paso 5: Vista "formulario" — Crear/Editar con Filas Dinamicas

Esta es la parte mas compleja. El formulario tiene dos secciones:
1. **Cabecera** (cliente, vendedor) — campos escalares con selects
2. **Productos** (filas dinamicas) — se pueden agregar y quitar

```html
@if (vista == "formulario")
{
    <div class="card mb-3">
        <div class="card-header">
            @(editando ? $"Editar Factura #{numeroEditar}" : "Nueva Factura")
        </div>
        <div class="card-body">

            @* ─── SECCION 1: CABECERA (selects de cliente y vendedor) ─── *@
            <div class="row">
                <div class="col-md-6 mb-3">
                    <label class="form-label">Cliente</label>
                    <select class="form-select" @bind="campoCliente">
                        <option value="0">-- Seleccione --</option>
                        @foreach (var cli in clientes)
                        {
                            <option value="@cli.Id">@cli.Nombre (Credito: $@cli.Credito)</option>
                        }
                    </select>
                </div>
                <div class="col-md-6 mb-3">
                    <label class="form-label">Vendedor</label>
                    <select class="form-select" @bind="campoVendedor">
                        <option value="0">-- Seleccione --</option>
                        @foreach (var ven in vendedores)
                        {
                            <option value="@ven.Id">@ven.Nombre (Carnet: @ven.Carnet)</option>
                        }
                    </select>
                </div>
            </div>

            @* ─── SECCION 2: PRODUCTOS (filas dinamicas) ─── *@
            <h5 class="mt-3">Productos</h5>
            @for (int i = 0; i < filasProductos.Count; i++)
            {
                var idx = i;  // IMPORTANTE: capturar indice para evitar closure
                <div class="row mb-2 align-items-end">
                    <div class="col-md-6">
                        <label class="form-label">Producto</label>
                        <select class="form-select" @bind="filasProductos[idx].Codigo">
                            <option value="">-- Seleccione --</option>
                            @foreach (var prod in productosDisponibles)
                            {
                                <option value="@prod["codigo"]">
                                    @prod["nombre"] (Stock: @prod["stock"] - $@prod["valorunitario"])
                                </option>
                            }
                        </select>
                    </div>
                    <div class="col-md-3">
                        <label class="form-label">Cantidad</label>
                        <input class="form-control" type="number" min="1"
                               @bind="filasProductos[idx].Cantidad" />
                    </div>
                    <div class="col-md-3">
                        @if (filasProductos.Count > 1)
                        {
                            <button class="btn btn-outline-danger"
                                    @onclick="() => QuitarFila(idx)">Quitar</button>
                        }
                    </div>
                </div>
            }

            <button class="btn btn-outline-primary mb-3"
                    @onclick="AgregarFila">+ Agregar Producto</button>

            @* ─── BOTONES DE ACCION ─── *@
            <div>
                <button class="btn btn-success me-2" @onclick="GuardarFactura">Guardar</button>
                <button class="btn btn-secondary" @onclick="Volver">Cancelar</button>
            </div>
        </div>
    </div>
}
```

**Punto critico — `var idx = i`:**

Dentro de un `@for`, si usas `i` directamente en un `@onclick` o `@bind`, todas las lambdas capturaran la **misma variable** `i`, que al final del loop vale `filasProductos.Count`. Capturar con `var idx = i` crea una copia local por cada iteracion.

```csharp
// MAL - todas las filas apuntan al mismo indice (el ultimo)
@for (int i = 0; i < filasProductos.Count; i++)
{
    <input @bind="filasProductos[i].Cantidad" />  // i cambia en cada iteracion
}

// BIEN - cada fila tiene su propio indice
@for (int i = 0; i < filasProductos.Count; i++)
{
    var idx = i;  // Copia local
    <input @bind="filasProductos[idx].Cantidad" />  // idx es fijo para esta iteracion
}
```

### Paso 6: Logica C# — Metodos del @code

#### 6.1 Ciclo de Vida

```csharp
protected override async Task OnInitializedAsync()
{
    await CargarFacturas();
}
```

`OnInitializedAsync()` se ejecuta una sola vez cuando la pagina se carga por primera vez. Aqui cargamos la lista de facturas.

#### 6.2 Cargar Facturas (listar)

```csharp
private async Task CargarFacturas()
{
    cargando = true;
    facturas = new();

    // Llamar al SP que lista todas las facturas con sus productos
    var parametros = new Dictionary<string, object?> { ["p_resultado"] = null };
    var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
        "sp_listar_facturas_y_productosporfactura", parametros);

    if (ok && resultados.Count > 0)
    {
        // El SP retorna el JSON completo en la primera fila, primer valor
        var jsonStr = resultados[0].Values.FirstOrDefault()?.ToString();
        if (!string.IsNullOrEmpty(jsonStr))
        {
            try
            {
                facturas = ParsearJsonArray(jsonStr);
            }
            catch
            {
                facturas = resultados;
            }
        }
    }

    cargando = false;
}
```

**Flujo de datos del SP:**

```
SP retorna:
  resultados[0]["@p_resultado"] = '[{"numero":1,"nombre_cliente":"Ana","productos":[...]}]'

                    ↓ resultados[0].Values.FirstOrDefault()?.ToString()

jsonStr = '[{"numero":1,"nombre_cliente":"Ana","productos":[{"codigo_producto":"PR001",...}]}]'

                    ↓ ParsearJsonArray(jsonStr)

facturas = [
    { "numero": 1, "nombre_cliente": "Ana", "productos": "[{...}]", ... },
    { "numero": 2, "nombre_cliente": "Maria", "productos": "[{...}]", ... }
]
```

Cada factura es un `Dictionary<string, object?>` donde `"productos"` es un JSON string (se parsea despues cuando se necesita).

#### 6.3 Cargar Datos para Formulario (selects con FK cruzado)

```csharp
private async Task CargarDatosFormulario()
{
    // Cargar tablas necesarias para los selects
    var rawClientes = await Api.ListarAsync("cliente");
    var rawVendedores = await Api.ListarAsync("vendedor");
    var personas = await Api.ListarAsync("persona");
    productosDisponibles = await Api.ListarAsync("producto");

    // Crear mapa de codigo persona -> nombre persona
    var mapaPersonas = personas.ToDictionary(
        p => p["codigo"]?.ToString() ?? "",
        p => p["nombre"]?.ToString() ?? "Sin nombre"
    );

    // Armar lista de clientes con el nombre REAL de la persona
    // (la tabla cliente solo tiene fkcodpersona, no el nombre)
    clientes = rawClientes.Select(c => new ClienteInfo
    {
        Id = int.TryParse(c["id"]?.ToString(), out int id) ? id : 0,
        Nombre = mapaPersonas.GetValueOrDefault(
            c["fkcodpersona"]?.ToString() ?? "", "Sin nombre"),
        Credito = c["credito"]?.ToString() ?? "0"
    }).ToList();

    // Armar lista de vendedores con el nombre REAL de la persona
    vendedores = rawVendedores.Select(v => new VendedorInfo
    {
        Id = int.TryParse(v["id"]?.ToString(), out int id) ? id : 0,
        Nombre = mapaPersonas.GetValueOrDefault(
            v["fkcodpersona"]?.ToString() ?? "", "Sin nombre"),
        Carnet = v["carnet"]?.ToString() ?? ""
    }).ToList();
}
```

**Cross-reference de FKs explicado:**

```
Tabla persona:   { codigo: "P001", nombre: "Ana Torres" }
Tabla cliente:   { id: 1, fkcodpersona: "P001", credito: 5000000 }

                    ↓ mapaPersonas["P001"] = "Ana Torres"

ClienteInfo:     { Id: 1, Nombre: "Ana Torres", Credito: "5000000" }

                    ↓ Se usa en el <select>

<option value="1">Ana Torres (Credito: $5000000)</option>
```

#### 6.4 Ver Factura (detalle)

```csharp
private async Task VerFactura(Dictionary<string, object?> f)
{
    int numero = int.TryParse(ObtenerValor(f, "numero"), out int n) ? n : 0;

    // Llamar al SP que consulta una factura especifica
    var parametros = new Dictionary<string, object?>
    {
        ["p_numero"] = numero,
        ["p_resultado"] = null
    };
    var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
        "sp_consultar_factura_y_productosporfactura", parametros);

    if (ok && resultados.Count > 0)
    {
        var jsonStr = resultados[0].Values.FirstOrDefault()?.ToString();
        if (!string.IsNullOrEmpty(jsonStr))
        {
            try
            {
                // El SP consultar retorna JSON anidado:
                // {"factura":{...},"productos":[...]}
                // AplanarFacturaJson lo convierte a un diccionario plano
                facturaActual = AplanarFacturaJson(jsonStr);
            }
            catch
            {
                facturaActual = f;
            }
        }
    }

    vista = "ver";  // Cambiar a la vista de detalle
}
```

#### 6.5 Editar Factura (precargar formulario)

```csharp
private async Task EditarFactura(Dictionary<string, object?> f)
{
    // Cargar datos de los selects
    await CargarDatosFormulario();

    // Precargar los campos del formulario con los datos actuales
    editando = true;
    numeroEditar = int.TryParse(ObtenerValor(f, "numero"), out int n) ? n : 0;
    campoCliente = int.TryParse(ObtenerValor(f, "fkidcliente"), out int c) ? c : 0;
    campoVendedor = int.TryParse(ObtenerValor(f, "fkidvendedor"), out int v) ? v : 0;

    // Precargar las filas de productos con los productos actuales
    var productos = ObtenerProductos(f);
    if (productos.Any())
    {
        filasProductos = productos.Select(p => new ProductoFila
        {
            Codigo = ObtenerValor(p, "codigo_producto"),
            Cantidad = int.TryParse(ObtenerValor(p, "cantidad"), out int cant) ? cant : 1
        }).ToList();
    }
    else
    {
        filasProductos = new() { new() };
    }

    mensaje = string.Empty;
    vista = "formulario";  // Cambiar a la vista de formulario
}
```

#### 6.6 Eliminar Factura (con confirmacion)

```csharp
private async Task EliminarFactura(Dictionary<string, object?> f)
{
    int numero = int.TryParse(ObtenerValor(f, "numero"), out int n) ? n : 0;

    // Mostrar dialogo de confirmacion del navegador
    var confirmar = await JS.InvokeAsync<bool>(
        "confirm", $"¿Está seguro de eliminar la Factura #{numero}?");
    if (!confirmar) return;  // Si cancela, no hacer nada

    var parametros = new Dictionary<string, object?>
    {
        ["p_numero"] = numero,
        ["p_resultado"] = null
    };
    var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
        "sp_borrar_factura_y_productosporfactura", parametros);

    exito = ok;
    mensaje = ok ? "Factura eliminada exitosamente." : $"Error al eliminar: {msg}";

    if (ok)
        await CargarFacturas();  // Recargar la lista
}
```

#### 6.7 Guardar Factura (crear o actualizar)

```csharp
private async Task GuardarFactura()
{
    // 1. Filtrar filas vacias y convertir a JSON
    var productosValidos = filasProductos
        .Where(p => !string.IsNullOrEmpty(p.Codigo) && p.Cantidad > 0)
        .Select(p => new { codigo = p.Codigo, cantidad = p.Cantidad })
        .ToList();

    // 2. Validar que haya al menos un producto
    if (!productosValidos.Any())
    {
        exito = false;
        mensaje = "Debe agregar al menos un producto.";
        return;
    }

    // 3. Serializar los productos a JSON string
    // Resultado: [{"codigo":"PR001","cantidad":2},{"codigo":"PR003","cantidad":3}]
    string jsonProductos = JsonSerializer.Serialize(productosValidos);

    // 4. Llamar al SP correspondiente
    if (editando)
    {
        var confirmar = await JS.InvokeAsync<bool>(
            "confirm", $"¿Está seguro de actualizar la Factura #{numeroEditar}?");
        if (!confirmar) return;

        var parametros = new Dictionary<string, object?>
        {
            ["p_numero"] = numeroEditar,
            ["p_fkidcliente"] = campoCliente,
            ["p_fkidvendedor"] = campoVendedor,
            ["p_productos"] = jsonProductos,    // JSON string, no objeto
            ["p_resultado"] = null
        };
        var (ok, _, msg) = await Sp.EjecutarSpAsync(
            "sp_actualizar_factura_y_productosporfactura", parametros);
        exito = ok;
        mensaje = ok ? "Factura actualizada exitosamente." : $"Error: {msg}";
    }
    else
    {
        var parametros = new Dictionary<string, object?>
        {
            ["p_fkidcliente"] = campoCliente,
            ["p_fkidvendedor"] = campoVendedor,
            ["p_productos"] = jsonProductos,
            ["p_resultado"] = null
        };
        var (ok, _, msg) = await Sp.EjecutarSpAsync(
            "sp_insertar_factura_y_productosporfactura", parametros);
        exito = ok;
        mensaje = ok ? "Factura creada exitosamente." : $"Error: {msg}";
    }

    // 5. Si exito, volver a la lista
    if (exito)
    {
        await CargarFacturas();
        vista = "listar";
    }
}
```

**Flujo de datos al guardar:**

```
filasProductos = [
    ProductoFila { Codigo = "PR001", Cantidad = 2 },
    ProductoFila { Codigo = "PR003", Cantidad = 3 },
    ProductoFila { Codigo = "",      Cantidad = 1 }    ← Fila vacia (se filtra)
]

        ↓ .Where(p => !string.IsNullOrEmpty(p.Codigo) && p.Cantidad > 0)

productosValidos = [
    { codigo = "PR001", cantidad = 2 },
    { codigo = "PR003", cantidad = 3 }
]

        ↓ JsonSerializer.Serialize()

jsonProductos = '[{"codigo":"PR001","cantidad":2},{"codigo":"PR003","cantidad":3}]'

        ↓ Se envia al SP como parametro "p_productos"

SP recibe el JSON y lo procesa con OPENJSON() (SQL Server) o json_array_elements() (PostgreSQL)
```

#### 6.8 Filas Dinamicas (agregar/quitar)

```csharp
private void AgregarFila()
{
    filasProductos.Add(new ProductoFila());
}

private void QuitarFila(int index)
{
    // Siempre mantener al menos una fila
    if (filasProductos.Count > 1)
        filasProductos.RemoveAt(index);
}
```

#### 6.9 Navegacion

```csharp
private async Task MostrarFormularioNueva()
{
    editando = false;
    numeroEditar = 0;
    campoCliente = 0;
    campoVendedor = 0;
    filasProductos = new() { new() };  // Una fila vacia
    mensaje = string.Empty;
    await CargarDatosFormulario();      // Cargar datos para selects
    vista = "formulario";
}

private async Task Volver()
{
    mensaje = string.Empty;
    await CargarFacturas();
    vista = "listar";
}
```

### Paso 7: Helpers de Parseo JSON

Estos metodos convierten el JSON crudo que retorna el SP en estructuras de datos que Blazor puede renderizar.

#### ParsearJsonArray — Convierte un JSON array a lista de diccionarios

```csharp
private List<Dictionary<string, object?>> ParsearJsonArray(string json)
{
    var lista = new List<Dictionary<string, object?>>();
    var doc = JsonDocument.Parse(json);

    foreach (var elem in doc.RootElement.EnumerateArray())
    {
        lista.Add(ParsearJsonObject(elem));
    }
    return lista;
}
```

**Entrada:** `'[{"numero":1,"total":5000000},{"numero":2,"total":1250000}]'`
**Salida:** `List<Dictionary>` con 2 elementos

#### ParsearJsonObject — Convierte un JSON object a diccionario

```csharp
private Dictionary<string, object?> ParsearJsonObject(JsonElement elem)
{
    var dic = new Dictionary<string, object?>();
    foreach (var prop in elem.EnumerateObject())
    {
        dic[prop.Name] = prop.Value.ValueKind switch
        {
            JsonValueKind.String  => prop.Value.GetString(),
            JsonValueKind.Number  => prop.Value.TryGetInt32(out int i) ? i : prop.Value.GetDouble(),
            JsonValueKind.True    => true,
            JsonValueKind.False   => false,
            JsonValueKind.Null    => null,
            JsonValueKind.Array   => prop.Value.GetRawText(),   // Se guarda como string JSON
            JsonValueKind.Object  => prop.Value.GetRawText(),   // Se guarda como string JSON
            _                     => prop.Value.GetRawText()
        };
    }
    return dic;
}
```

Los arrays y objetos anidados se guardan como **raw JSON string** (no se deserializan). Esto permite parsearlos on-demand con `ObtenerProductos()`.

#### ObtenerValor — Extrae un valor de forma segura

```csharp
private string ObtenerValor(Dictionary<string, object?> dic, string clave)
{
    if (dic.TryGetValue(clave, out var val) && val != null)
        return val.ToString() ?? "";
    return "";
}
```

Nunca lanza excepcion. Si la clave no existe, retorna `""`.

#### ObtenerProductos — Parsea el JSON de productos anidado

```csharp
private List<Dictionary<string, object?>> ObtenerProductos(Dictionary<string, object?> factura)
{
    if (factura.TryGetValue("productos", out var val) && val != null)
    {
        var strVal = val.ToString() ?? "";

        // Si es un JSON string (viene del SP parseado)
        if (strVal.StartsWith("["))
        {
            try { return ParsearJsonArray(strVal); } catch { }
        }

        // Si ya es una lista deserializada
        if (val is List<Dictionary<string, object?>> lista)
            return lista;
    }
    return new();
}
```

#### AplanarFacturaJson — Aplana JSON anidado del SP consultar

El SP `sp_consultar_factura_y_productosporfactura` retorna un JSON con estructura anidada:

```json
{
    "factura": {
        "numero": 1,
        "fecha": "2025-12-03",
        "total": 5000000,
        "fkidcliente": 1,
        "nombre_cliente": "Ana Torres"
    },
    "productos": [
        {"codigo_producto": "PR001", "cantidad": 2, "subtotal": 5000000}
    ]
}
```

Para que `ObtenerValor(facturaActual, "numero")` funcione, hay que "aplanar" las propiedades de `factura` al nivel raiz:

```csharp
private Dictionary<string, object?> AplanarFacturaJson(string json)
{
    var doc = JsonDocument.Parse(json);
    var root = doc.RootElement;
    var resultado = new Dictionary<string, object?>();

    // Si tiene "factura" como sub-objeto, extraer sus propiedades al nivel raiz
    if (root.TryGetProperty("factura", out var factura)
        && factura.ValueKind == JsonValueKind.Object)
    {
        foreach (var prop in factura.EnumerateObject())
        {
            resultado[prop.Name] = prop.Value.ValueKind switch
            {
                JsonValueKind.String => prop.Value.GetString(),
                JsonValueKind.Number => prop.Value.TryGetInt32(out int i)
                    ? i : prop.Value.GetDouble(),
                JsonValueKind.Null   => null,
                _                    => prop.Value.GetRawText()
            };
        }
    }

    // Copiar "productos" como JSON string
    if (root.TryGetProperty("productos", out var productos))
    {
        resultado["productos"] = productos.GetRawText();
    }

    // Fallback: si no tenia estructura anidada, parsear como objeto plano
    if (!resultado.Any())
    {
        resultado = ParsearJsonObject(root);
    }

    return resultado;
}
```

**Resultado despues de aplanar:**
```csharp
{
    "numero": 1,
    "fecha": "2025-12-03",
    "total": 5000000,
    "fkidcliente": 1,
    "nombre_cliente": "Ana Torres",
    "productos": "[{\"codigo_producto\":\"PR001\",...}]"  // JSON string
}
```

Ahora `ObtenerValor(facturaActual, "numero")` retorna `"1"` correctamente.

---

## 6. Patrones Clave Explicados

### Patron 1: Switch de Vistas con String

En lugar de multiples `bool`, un solo `string` controla la vista activa:

```csharp
// Definir
private string vista = "listar";

// Cambiar
vista = "formulario";
vista = "ver";
vista = "listar";

// Renderizar
@if (vista == "listar") { /* tabla */ }
@if (vista == "ver")    { /* detalle */ }
@if (vista == "formulario") { /* form */ }
```

**Ventaja:** Escalable. Si necesitas una 4ta vista (ej: "reporte"), solo agregas otro `@if`.

### Patron 2: Filas Dinamicas con List y @for

```csharp
// Modelo de la fila
private class ProductoFila
{
    public string Codigo { get; set; } = "";
    public int Cantidad { get; set; } = 1;
}

// Lista de filas (inicia con 1 vacia)
private List<ProductoFila> filasProductos = new() { new() };

// Agregar fila
private void AgregarFila() => filasProductos.Add(new ProductoFila());

// Quitar fila (mantener minimo 1)
private void QuitarFila(int index)
{
    if (filasProductos.Count > 1)
        filasProductos.RemoveAt(index);
}
```

**En el HTML — SIEMPRE usar `@for` (no `@foreach`) con `var idx = i`:**

```html
@for (int i = 0; i < filasProductos.Count; i++)
{
    var idx = i;
    <select @bind="filasProductos[idx].Codigo">...</select>
    <input @bind="filasProductos[idx].Cantidad" />
    <button @onclick="() => QuitarFila(idx)">Quitar</button>
}
```

### Patron 3: Cross-Reference de FKs con Mapa Dictionary

Cuando una tabla tiene FK a otra tabla y quieres mostrar el nombre (no el ID):

```csharp
// 1. Cargar ambas tablas
var clientes = await Api.ListarAsync("cliente");      // tiene fkcodpersona
var personas = await Api.ListarAsync("persona");       // tiene codigo + nombre

// 2. Crear mapa codigo → nombre
var mapa = personas.ToDictionary(
    p => p["codigo"]?.ToString() ?? "",
    p => p["nombre"]?.ToString() ?? "Sin nombre"
);

// 3. Cruzar: reemplazar FK por nombre
var resultado = clientes.Select(c => new {
    Id = c["id"],
    Nombre = mapa.GetValueOrDefault(c["fkcodpersona"]?.ToString() ?? "", "Sin nombre")
});
```

### Patron 4: JSON como Puente entre SP y UI

Los SPs retornan datos complejos (maestro + detalle) como un JSON string en un parametro OUTPUT.

```
UI (Blazor)  →  SpService  →  API  →  SP  →  BD
                                         ↓
                              JSON string en @p_resultado
                                         ↓
UI (Blazor)  ←  SpService  ←  API  ←  SP

La UI parsea el JSON con ParsearJsonArray() / AplanarFacturaJson()
```

Los productos se envian al SP tambien como JSON:

```
UI: filasProductos → JsonSerializer.Serialize() → '[{"codigo":"PR001","cantidad":2}]'
                                                              ↓
SP: OPENJSON(@p_productos) → INSERT INTO productosporfactura
```

### Patron 5: Aplanar JSON Anidado

Cuando el SP retorna estructura anidada `{"maestro":{...},"detalle":[...]}`, se aplana para mantener el acceso uniforme con `ObtenerValor()`:

```
Antes:  { "factura": { "numero": 1 }, "productos": [...] }
                    ↓ AplanarFacturaJson()
Despues: { "numero": 1, "productos": "[...]" }

Acceso uniforme: ObtenerValor(facturaActual, "numero")  // funciona en ambos casos
```

---

## 7. Otros Ejemplos de Maestro-Detalle

### Ejemplo: Pedido → Items del Pedido

Misma estructura que Factura, con diferentes campos:

```csharp
// Clases auxiliares
private class ItemFila
{
    public string CodigoProducto { get; set; } = "";
    public int Cantidad { get; set; } = 1;
    public string Observacion { get; set; } = "";
}

// Variables de estado (identico patron)
private string vista = "listar";
private List<Dictionary<string, object?>> pedidos = new();
private List<ItemFila> filasItems = new() { new() };
private int campoCliente = 0;
private string campoDireccionEntrega = "";
```

**SPs equivalentes:**
| Factura | Pedido |
|---------|--------|
| `sp_listar_facturas_y_productosporfactura` | `sp_listar_pedidos_y_items` |
| `sp_consultar_factura_y_productosporfactura` | `sp_consultar_pedido_y_items` |
| `sp_insertar_factura_y_productosporfactura` | `sp_insertar_pedido_y_items` |
| `sp_actualizar_factura_y_productosporfactura` | `sp_actualizar_pedido_y_items` |
| `sp_borrar_factura_y_productosporfactura` | `sp_borrar_pedido_y_items` |

### Ejemplo: Matricula → Materias Inscritas

```csharp
// Clase para la fila de detalle
private class MateriaFila
{
    public string CodigoMateria { get; set; } = "";
    public string Horario { get; set; } = "Manana";
}

// El select de materias carga materias disponibles
private List<Dictionary<string, object?>> materiasDisponibles = new();

// Campos del maestro
private int campoEstudiante = 0;
private string campoPeriodo = "2026-1";
```

### Tabla de Adaptacion

Para adaptar Factura.razor a otro maestro-detalle, cambia estos elementos:

| Elemento | Factura → Productos | Tu caso |
|----------|-------------------|---------|
| `@page` | `"/factura"` | `"/tu-pagina"` |
| Vista "listar" columnas | Numero, Cliente, Vendedor, Total | Tus columnas del maestro |
| Vista "ver" cabecera | Card con datos de factura | Card con datos de tu maestro |
| Vista "ver" detalle | Tabla de productos | Tabla de tu detalle |
| `ProductoFila` | Codigo + Cantidad | Campos de tu detalle |
| Selects del formulario | Cliente, Vendedor | FKs de tu maestro |
| Filas dinamicas | Productos (select + cantidad) | Items de tu detalle |
| SPs | `sp_*_factura_y_productosporfactura` | `sp_*_tu_maestro_y_detalle` |
| Clases auxiliares | `ClienteInfo`, `VendedorInfo` | Las que necesites para tus FKs |

---

## 8. Errores Comunes y Soluciones

### Error 1: Closure en @for

**Sintoma:** Todas las filas modifican la misma fila (la ultima).

```csharp
// MAL
@for (int i = 0; i < filas.Count; i++)
{
    <button @onclick="() => QuitarFila(i)">Quitar</button>
}
```

**Solucion:** Capturar el indice con `var idx = i`:

```csharp
// BIEN
@for (int i = 0; i < filas.Count; i++)
{
    var idx = i;
    <button @onclick="() => QuitarFila(idx)">Quitar</button>
}
```

### Error 2: @bind dentro de @foreach

**Sintoma:** Error de compilacion o comportamiento inesperado con `@bind`.

```csharp
// MAL - @bind no funciona correctamente con @foreach y listas mutables
@foreach (var fila in filas)
{
    <input @bind="fila.Cantidad" />
}
```

**Solucion:** Usar `@for` con indice:

```csharp
// BIEN
@for (int i = 0; i < filas.Count; i++)
{
    var idx = i;
    <input @bind="filas[idx].Cantidad" />
}
```

### Error 3: @p_resultado vs p_resultado

**Sintoma:** Los datos del SP no se parsean correctamente.

SQL Server retorna los parametros OUTPUT con prefijo `@`:
```json
{ "resultados": [{ "@p_resultado": "[...]" }] }
```

PostgreSQL retorna sin prefijo:
```json
{ "resultados": [{ "p_resultado": "[...]" }] }
```

**Solucion:** Acceder por valor (no por clave):

```csharp
// BIEN - funciona con ambos motores
var jsonStr = resultados[0].Values.FirstOrDefault()?.ToString();
```

En el servicio Flask (`api_service.py`), se buscan ambas variantes:

```python
p_resultado = resultados[0].get("p_resultado") or resultados[0].get("@p_resultado")
```

### Error 4: confirm() en Blazor Server

**Sintoma:** Se quiere mostrar un dialogo de confirmacion antes de eliminar/actualizar.

En Blazor Server no se puede usar `window.confirm()` directamente desde C#. Se necesita `IJSRuntime`:

```csharp
// 1. Inyectar en la parte superior del archivo
@inject IJSRuntime JS

// 2. Llamar confirm() via interop
var confirmar = await JS.InvokeAsync<bool>("confirm", "¿Está seguro?");
if (!confirmar) return;
```

### Error 5: RZ1010 — @{ dentro de @if

**Sintoma:** Error de compilacion `Unexpected "{" after "@"`.

```csharp
// MAL - @{ bloque de codigo } dentro de un @if genera error
@if (vista == "ver")
{
    @{ var prods = ObtenerProductos(facturaActual); }
    <p>@prods.Count productos</p>
}
```

**Solucion:** Usar metodos helper o expresiones inline:

```csharp
// BIEN - llamar un metodo directamente
@if (vista == "ver")
{
    <p>@ObtenerProductos(facturaActual).Count productos</p>
}
```

---

## 9. Equivalencia Flask vs Blazor

### Patrones Comparados

| Concepto | Flask (Python) | Blazor Server (C#) |
|----------|---------------|-------------------|
| Llamar SP | `api.ejecutar_sp("sp_name", params)` | `await Sp.EjecutarSpAsync("sp_name", params)` |
| Serializar JSON | `json.dumps(productos)` | `JsonSerializer.Serialize(productos)` |
| Parsear resultado | `datos.get("facturas", [])` | `ParsearJsonArray(jsonStr)` |
| Mostrar alerta | `flash("Exito", "success")` | `exito = true; mensaje = "Exito";` |
| Redirigir | `redirect(url_for('factura.index'))` | `vista = "listar"; await CargarFacturas();` |
| Vista condicional | `render_template(vista='listar')` | `@if (vista == "listar") { ... }` |
| Confirmar eliminar | `onsubmit="return confirm('...')"` | `await JS.InvokeAsync<bool>("confirm", "...")` |
| Confirmar actualizar | `onsubmit="return confirm('...')"` | `await JS.InvokeAsync<bool>("confirm", "...")` |
| Formulario dinamico | JavaScript DOM manipulation | `List<ProductoFila>` + `@for` |
| Agregar fila | `document.createElement()` + JS | `filasProductos.Add(new ProductoFila())` |
| Quitar fila | `element.remove()` + JS | `filasProductos.RemoveAt(index)` |
| FK cross-reference | `mapa_personas = {p['codigo']: p['nombre']}` | `personas.ToDictionary(p => p["codigo"], p => p["nombre"])` |

### Arquitectura Comparada

```
Flask:
  Template (.html) ← Jinja2 → Route (.py) ← requests → API

Blazor:
  Page (.razor)    ← C# →     Service (.cs) ← HttpClient → API
```

En Flask, la logica esta separada en el route (Python) y la vista (HTML con Jinja2).
En Blazor, todo esta en un solo archivo `.razor` que combina HTML + C# en el bloque `@code`.

### Ventaja de Blazor para Maestro-Detalle

Las filas dinamicas en Flask requieren **JavaScript puro** para manipular el DOM (agregar/quitar filas HTML). En Blazor, se manejan con una simple `List<T>` en C# y el framework se encarga de actualizar el DOM automaticamente.

```csharp
// Blazor: agregar una fila es una sola linea de C#
private void AgregarFila() => filasProductos.Add(new ProductoFila());

// El framework re-renderiza automaticamente el @for loop
```

---

## Resumen

Para construir una pagina maestro-detalle en Blazor Server:

1. **Definir 3 vistas** con `string vista` (`"listar"`, `"ver"`, `"formulario"`)
2. **Crear clases auxiliares** para las filas del detalle (`ProductoFila`) y los FKs (`ClienteInfo`)
3. **Usar `@for` con `var idx = i`** para las filas dinamicas (nunca `@foreach`)
4. **Usar SpService** para las operaciones transaccionales (crear/editar/eliminar maestro+detalle)
5. **Usar ApiService** para cargar datos de apoyo (tablas para selects)
6. **Serializar el detalle a JSON** antes de enviar al SP
7. **Parsear el JSON del SP** con helpers reutilizables
8. **Aplanar JSON anidado** cuando el SP retorna estructura `{maestro:{}, detalle:[]}`
9. **Agregar confirmaciones** con `IJSRuntime` para eliminar y actualizar
