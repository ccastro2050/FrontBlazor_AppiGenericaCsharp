# Guia de Uso - EntidadesController desde Blazor Server

Frontend Blazor Server que consume la API generica `ApiGenericaCsharp` para operaciones CRUD sobre cualquier tabla.

- **API**: `http://localhost:5035` (configurado en `appsettings.Development.json` → `ApiBaseUrl`)
- **Blazor**: `http://localhost:5200`
- **Servicio**: `ApiService.cs` inyectado via DI

---

## 1. Arquitectura

```
Blazor Server (puerto 5200)
    └── ApiService.cs (HttpClient)
            └── GET/POST/PUT/DELETE → http://localhost:5035/api/{tabla}
                    └── EntidadesController (API)
                            └── Base de datos (PostgreSQL / SQL Server)
```

### Configuracion en Program.cs

```csharp
// Lee la URL de appsettings.json / appsettings.Development.json
var apiBaseUrl = builder.Configuration["ApiBaseUrl"] ?? "http://localhost:5035";
builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri(apiBaseUrl)
});
builder.Services.AddScoped<ApiService>();
```

### appsettings.Development.json

```json
{
  "ApiBaseUrl": "http://localhost:5035"
}
```

---

## 2. Servicio ApiService

Ubicacion: `Services/ApiService.cs`

### Metodos disponibles

| Metodo | Endpoint API | Descripcion |
|--------|-------------|-------------|
| `ListarAsync(tabla, limite?)` | `GET /api/{tabla}?limite=N` | Lista registros |
| `CrearAsync(tabla, datos, camposEncriptar?)` | `POST /api/{tabla}` | Crea registro |
| `ActualizarAsync(tabla, nombreClave, valorClave, datos, camposEncriptar?)` | `PUT /api/{tabla}/{clave}/{valor}` | Actualiza registro |
| `EliminarAsync(tabla, nombreClave, valorClave)` | `DELETE /api/{tabla}/{clave}/{valor}` | Elimina registro |
| `ObtenerDiagnosticoAsync()` | `GET /api/diagnostico/conexion` | Info de conexion |

### Tipos de retorno

- `ListarAsync` → `List<Dictionary<string, object?>>`
- `CrearAsync`, `ActualizarAsync`, `EliminarAsync` → `(bool exito, string mensaje)`
- `ObtenerDiagnosticoAsync` → `Dictionary<string, string>?`

---

## 3. Tablas disponibles

| Tabla | Clave Primaria | Tipo PK | Campos |
|-------|---------------|---------|--------|
| empresa | codigo | string | nombre |
| persona | codigo | string | nombre, email, telefono |
| producto | codigo | string | nombre, stock (int), valorunitario (decimal) |
| rol | id | int (auto) | nombre |
| ruta | ruta | string | descripcion |
| usuario | email | string | contrasena |
| cliente | id | int (auto) | credito, fkcodpersona, fkcodempresa |
| vendedor | id | int (auto) | carnet, direccion, fkcodpersona |

---

## 4. Ejemplo completo: Producto.razor

### Inyeccion del servicio

```razor
@page "/producto"
@rendermode InteractiveServer
@inject FrontBlazor_AppiGenericaCsharp.Services.ApiService Api
```

### Variables de estado

```csharp
@code {
    private List<Dictionary<string, object?>> registros = new();
    private bool cargando = true;
    private bool mostrarFormulario = false;
    private bool editando = false;
    private string mensaje = string.Empty;
    private bool exito = false;

    // Campos del formulario
    private string campoCodigo = string.Empty;
    private string campoNombre = string.Empty;
    private int campoStock = 0;
    private double campoValor = 0;
    private int? limite = null;
}
```

### Cargar registros

```csharp
protected override async Task OnInitializedAsync()
{
    await CargarRegistros();
}

private async Task CargarRegistros()
{
    cargando = true;
    registros = await Api.ListarAsync("producto", limite);
    cargando = false;
}
```

### Crear registro

```csharp
var datos = new Dictionary<string, object?>
{
    ["codigo"] = campoCodigo,
    ["nombre"] = campoNombre,
    ["stock"] = campoStock,
    ["valorunitario"] = campoValor
};
var resultado = await Api.CrearAsync("producto", datos);
exito = resultado.exito;
mensaje = resultado.mensaje;
```

### Actualizar registro

```csharp
// No incluir la PK en los datos, va en la URL
datos.Remove("codigo");
var resultado = await Api.ActualizarAsync("producto", "codigo", campoCodigo, datos);
```

### Eliminar registro

```csharp
var resultado = await Api.EliminarAsync("producto", "codigo", codigo);
```

---

## 5. Patron: Selects con foreign keys (Cliente.razor)

Cuando una tabla tiene FK a otra, se cargan ambas tablas para mostrar nombres en vez de codigos.

```csharp
// En OnInitializedAsync: cargar tablas relacionadas
personas = await Api.ListarAsync("persona");
empresas = await Api.ListarAsync("empresa");
await CargarRegistros(); // clientes

// Helper para buscar nombre
private string ObtenerNombrePersona(string? codigo)
{
    if (string.IsNullOrEmpty(codigo)) return "Sin persona";
    var persona = personas.FirstOrDefault(p => p["codigo"]?.ToString() == codigo);
    return persona?["nombre"]?.ToString() ?? codigo;
}
```

```razor
@* Select con datos de otra tabla *@
<select class="form-select" @bind="campoFkcodpersona">
    <option value="">-- Seleccione --</option>
    @foreach (var p in personas)
    {
        <option value="@p["codigo"]">@p["nombre"] (@p["codigo"])</option>
    }
</select>

@* En la tabla, mostrar nombre en vez de codigo FK *@
<td>@ObtenerNombrePersona(reg["fkcodpersona"]?.ToString())</td>
```

---

## 6. Patron: Encriptacion de campos (Usuario.razor)

Para encriptar campos como contrasena con bcrypt:

```csharp
// Al crear
var resultado = await Api.CrearAsync("usuario", datos, "contrasena");

// Al actualizar (solo si se quiere re-encriptar)
var resultado = await Api.ActualizarAsync("usuario", "email", email, datos, "contrasena");
```

El parametro `camposEncriptar` se envia como query string: `?camposEncriptar=contrasena`

---

## 7. Patron: Alertas

```razor
@if (!string.IsNullOrEmpty(mensaje))
{
    <div class="alert @(exito ? "alert-success" : "alert-danger") alert-dismissible fade show">
        @mensaje
        <button type="button" class="btn-close" @onclick="() => mensaje = string.Empty"></button>
    </div>
}
```

---

## 8. Patron: Spinner de carga

```razor
@if (cargando)
{
    <div class="d-flex justify-content-center my-4">
        <div class="spinner-border text-primary" role="status">
            <span class="visually-hidden">Cargando...</span>
        </div>
    </div>
}
```

---

## 9. Como agregar una nueva pagina CRUD

1. Crear `Components/Pages/NuevaTabla.razor`
2. Agregar `@page "/nuevatabla"` y `@rendermode InteractiveServer`
3. Inyectar `@inject ... ApiService Api`
4. Copiar el patron de `Empresa.razor` (mas simple) o `Producto.razor`
5. Cambiar nombre de tabla, campos y clave primaria
6. Agregar link en `Components/Layout/NavMenu.razor`:
   ```razor
   <div class="nav-item px-3">
       <NavLink class="nav-link" href="nuevatabla">
           <span class="bi bi-list-nested-nav-menu" aria-hidden="true"></span> NuevaTabla
       </NavLink>
   </div>
   ```

---

## 10. Equivalencia con Flask

| Flask (Python) | Blazor Server (C#) |
|----------------|-------------------|
| `api.listar('tabla')` | `await Api.ListarAsync("tabla")` |
| `api.crear('tabla', datos)` | `await Api.CrearAsync("tabla", datos)` |
| `api.actualizar('tabla', 'pk', val, datos)` | `await Api.ActualizarAsync("tabla", "pk", val, datos)` |
| `api.eliminar('tabla', 'pk', val)` | `await Api.EliminarAsync("tabla", "pk", val)` |
| `flash(msg, 'success')` | `exito = true; mensaje = msg;` |
| `redirect(url_for(...))` | `await CargarRegistros();` (SPA, no redirect) |
| `request.form.get('campo')` | `@bind="campoCampo"` (two-way binding) |
| `{% if editando %}` | `@if (editando)` |

---

## 11. Pruebas con Swagger / Postman / curl

### Listar productos
```bash
curl http://localhost:5035/api/producto
```

### Crear producto
```bash
curl -X POST http://localhost:5035/api/producto \
  -H "Content-Type: application/json" \
  -d '{"codigo":"PR099","nombre":"Test","stock":10,"valorunitario":500}'
```

### Actualizar producto
```bash
curl -X PUT http://localhost:5035/api/producto/codigo/PR099 \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Test Actualizado","stock":20,"valorunitario":600}'
```

### Eliminar producto
```bash
curl -X DELETE http://localhost:5035/api/producto/codigo/PR099
```

### Con limite
```bash
curl http://localhost:5035/api/producto?limite=5
```

### Swagger UI
Abrir en navegador: `http://localhost:5035/swagger`
