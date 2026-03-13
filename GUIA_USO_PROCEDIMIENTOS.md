# Guia de Uso - ProcedimientosController desde Blazor Server

Frontend Blazor Server que consume stored procedures de la API generica `ApiGenericaCsharp`.

- **API**: `http://localhost:5035` (configurado en `appsettings.Development.json`)
- **Blazor**: `http://localhost:5200`
- **Servicio**: `SpService.cs` inyectado via DI
- **Endpoint API**: `POST /api/procedimientos/ejecutarsp`

---

## 1. Arquitectura

```
Blazor Server (puerto 5200)
    └── SpService.cs (HttpClient)
            └── POST → http://localhost:5035/api/procedimientos/ejecutarsp
                    └── ProcedimientosController (API)
                            └── Stored Procedure (PostgreSQL / SQL Server)
```

---

## 2. Servicio SpService

Ubicacion: `Services/SpService.cs`

### Metodo principal

```csharp
public async Task<(bool exito, List<Dictionary<string, object?>> resultados, string mensaje)>
    EjecutarSpAsync(string nombreSP, Dictionary<string, object?>? parametros = null)
```

**Payload que envia al API:**
```json
{
  "nombreSP": "nombre_del_sp",
  "param1": "valor1",
  "param2": 123,
  "p_resultado": null
}
```

**Retorna:**
- `exito`: true/false
- `resultados`: lista de diccionarios (filas del DataTable)
- `mensaje`: mensaje de exito o error

---

## 3. Stored Procedures disponibles

### Tabla resumen

| SP | Accion | Parametros |
|----|--------|-----------|
| `sp_listar_facturas_y_productosporfactura` | Listar todas | `p_resultado` |
| `sp_consultar_factura_y_productosporfactura` | Ver una | `p_numero`, `p_resultado` |
| `sp_insertar_factura_y_productosporfactura` | Crear | `p_fkidcliente`, `p_fkidvendedor`, `p_productos`, `p_minimo_detalle`, `p_resultado` |
| `sp_actualizar_factura_y_productosporfactura` | Actualizar | `p_numero`, `p_fkidcliente`, `p_fkidvendedor`, `p_productos`, `p_minimo_detalle`, `p_resultado` |
| `sp_borrar_factura_y_productosporfactura` | Eliminar | `p_numero`, `p_resultado` |

---

## 4. Ejemplos de uso desde Blazor

### 4.1 Listar facturas

```csharp
var parametros = new Dictionary<string, object?> { ["p_resultado"] = null };
var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
    "sp_listar_facturas_y_productosporfactura", parametros);

if (ok && resultados.Count > 0)
{
    // El SP retorna JSON en p_resultado (primera fila, primer valor)
    var jsonStr = resultados[0].Values.FirstOrDefault()?.ToString();
    var facturas = ParsearJsonArray(jsonStr);
    // facturas es List<Dictionary<string, object?>>
    // Cada factura tiene: numero, fecha, total, nombre_cliente, nombre_vendedor, productos
}
```

**JSON retornado por el SP:**
```json
[
  {
    "numero": 1,
    "fecha": "2025-12-03T12:57:19",
    "total": 5000000.00,
    "fkidcliente": 1,
    "nombre_cliente": "Ana Torres",
    "fkidvendedor": 1,
    "nombre_vendedor": "Carlos Perez",
    "productos": [
      {"codigo_producto": "PR001", "nombre_producto": "Laptop", "cantidad": 2, "valorunitario": 2500000, "subtotal": 5000000}
    ]
  }
]
```

### 4.2 Consultar una factura

```csharp
var parametros = new Dictionary<string, object?>
{
    ["p_numero"] = 1,
    ["p_resultado"] = null
};
var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
    "sp_consultar_factura_y_productosporfactura", parametros);
```

### 4.3 Crear factura

```csharp
var productos = new[] {
    new { codigo = "PR001", cantidad = 2 },
    new { codigo = "PR003", cantidad = 3 }
};
string jsonProductos = JsonSerializer.Serialize(productos);

var parametros = new Dictionary<string, object?>
{
    ["p_fkidcliente"] = 1,
    ["p_fkidvendedor"] = 1,
    ["p_productos"] = jsonProductos,  // JSON string, no objeto
    ["p_resultado"] = null
};
var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
    "sp_insertar_factura_y_productosporfactura", parametros);
```

### 4.4 Actualizar factura

```csharp
var parametros = new Dictionary<string, object?>
{
    ["p_numero"] = 1,
    ["p_fkidcliente"] = 2,
    ["p_fkidvendedor"] = 1,
    ["p_productos"] = "[{\"codigo\":\"PR002\",\"cantidad\":1}]",
    ["p_resultado"] = null
};
var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
    "sp_actualizar_factura_y_productosporfactura", parametros);
```

### 4.5 Eliminar factura

```csharp
var parametros = new Dictionary<string, object?>
{
    ["p_numero"] = 1,
    ["p_resultado"] = null
};
var (ok, resultados, msg) = await Sp.EjecutarSpAsync(
    "sp_borrar_factura_y_productosporfactura", parametros);
```

---

## 5. Parametro p_minimo_detalle

Controla cuantos productos minimo debe tener una factura.

- **Default:** 1 (al menos un producto)
- **Si no se envia:** La API envia 0 al SP, el SP lo convierte a 1 con `COALESCE(NULLIF(p_minimo_detalle, 0), 1)`
- **Uso:** Enviar como entero en el payload

```csharp
// Exigir minimo 3 productos
var parametros = new Dictionary<string, object?>
{
    ["p_fkidcliente"] = 1,
    ["p_fkidvendedor"] = 1,
    ["p_productos"] = jsonProductos,
    ["p_minimo_detalle"] = 3,   // minimo 3 productos
    ["p_resultado"] = null
};
```

Si se envian menos productos del minimo, el SP lanza excepcion:
```
La factura requiere minimo 3 producto(s).
```

---

## 6. Validacion de stock

El trigger en la base de datos valida automaticamente que haya stock suficiente antes de insertar/actualizar productos en una factura.

**Si no hay stock suficiente:**
```
Stock insuficiente para producto PR001. Stock disponible: 5, cantidad solicitada: 10
```

Este error llega como HTTP 500 desde la API.

**Comportamiento del trigger:**
- **INSERT:** Valida stock, calcula subtotal, descuenta stock, recalcula total factura
- **UPDATE:** Devuelve stock anterior, valida nuevo stock, recalcula
- **DELETE:** Restaura stock, recalcula total factura

---

## 7. Parseo de resultados JSON

Los SPs retornan JSON en el parametro `p_resultado`. La API lo convierte a una fila de DataTable con una columna `p_resultado` que contiene el JSON como string.

### Patron de parseo en Factura.razor

```csharp
var (ok, resultados, msg) = await Sp.EjecutarSpAsync("sp_listar...", parametros);
if (ok && resultados.Count > 0)
{
    // Primera fila, primer valor = JSON string
    var jsonStr = resultados[0].Values.FirstOrDefault()?.ToString();
    if (!string.IsNullOrEmpty(jsonStr))
    {
        var facturas = ParsearJsonArray(jsonStr);
    }
}
```

### Helper para parsear JSON array

```csharp
private List<Dictionary<string, object?>> ParsearJsonArray(string json)
{
    var lista = new List<Dictionary<string, object?>>();
    var doc = JsonDocument.Parse(json);
    foreach (var elem in doc.RootElement.EnumerateArray())
    {
        var dic = new Dictionary<string, object?>();
        foreach (var prop in elem.EnumerateObject())
        {
            dic[prop.Name] = prop.Value.ValueKind switch
            {
                JsonValueKind.String => prop.Value.GetString(),
                JsonValueKind.Number => prop.Value.TryGetInt32(out int i) ? i : prop.Value.GetDouble(),
                JsonValueKind.Null => null,
                JsonValueKind.Array => prop.Value.GetRawText(),
                _ => prop.Value.GetRawText()
            };
        }
        lista.Add(dic);
    }
    return lista;
}
```

---

## 8. Manejo de errores

| Codigo HTTP | Causa | Ejemplo |
|-------------|-------|---------|
| 400 | SP no existe | `"El procedimiento 'sp_inexistente' no existe en la base de datos"` |
| 400 | Parametros invalidos | Falta nombreSP |
| 500 | Stock insuficiente | `"Stock insuficiente para producto PR001..."` |
| 500 | Minimo productos | `"La factura requiere minimo 3 producto(s)."` |
| 500 | Factura no existe | `"Factura 999 no existe"` |

En Blazor, `SpService` captura el error:
```csharp
var (ok, resultados, msg) = await Sp.EjecutarSpAsync("sp_inexistente", params);
// ok = false
// msg = "El procedimiento 'sp_inexistente' no existe en la base de datos"
```

---

## 9. Formulario dinamico de productos (Factura.razor)

### Clase para filas

```csharp
private class ProductoFila
{
    public string Codigo { get; set; } = "";
    public int Cantidad { get; set; } = 1;
}
private List<ProductoFila> filasProductos = new() { new() };
```

### Agregar/quitar filas

```csharp
private void AgregarFila() => filasProductos.Add(new ProductoFila());

private void QuitarFila(int index)
{
    if (filasProductos.Count > 1)
        filasProductos.RemoveAt(index);
}
```

### Convertir a JSON para el SP

```csharp
var productosValidos = filasProductos
    .Where(p => !string.IsNullOrEmpty(p.Codigo) && p.Cantidad > 0)
    .Select(p => new { codigo = p.Codigo, cantidad = p.Cantidad })
    .ToList();
string jsonProductos = JsonSerializer.Serialize(productosValidos);
// Resultado: [{"codigo":"PR001","cantidad":2},{"codigo":"PR003","cantidad":3}]
```

---

## 10. Selects con nombres cruzados (cliente/vendedor)

Para mostrar nombres reales en los selects (no solo IDs), se carga la tabla persona y se cruza:

```csharp
var rawClientes = await Api.ListarAsync("cliente");
var personas = await Api.ListarAsync("persona");
var mapaPersonas = personas.ToDictionary(
    p => p["codigo"]?.ToString() ?? "",
    p => p["nombre"]?.ToString() ?? "Sin nombre"
);

clientes = rawClientes.Select(c => new ClienteInfo
{
    Id = int.TryParse(c["id"]?.ToString(), out int id) ? id : 0,
    Nombre = mapaPersonas.GetValueOrDefault(c["fkcodpersona"]?.ToString() ?? "", "Sin nombre"),
    Credito = c["credito"]?.ToString() ?? "0"
}).ToList();
```

---

## 11. Equivalencia con Flask

| Flask (Python) | Blazor Server (C#) |
|----------------|-------------------|
| `api.ejecutar_sp("sp_name", params)` | `await Sp.EjecutarSpAsync("sp_name", params)` |
| `json.dumps(productos)` | `JsonSerializer.Serialize(productos)` |
| `datos.get("facturas", [])` | `ParsearJsonArray(jsonStr)` |
| `flash("Exito", "success")` | `exito = true; mensaje = "Exito";` |
| `redirect(url_for('factura.index'))` | `vista = "listar"; await CargarFacturas();` |
| Vista con `render_template(vista='listar')` | `@if (vista == "listar") { ... }` |

---

## 12. Pruebas con curl

### Listar facturas
```bash
curl -X POST http://localhost:5035/api/procedimientos/ejecutarsp \
  -H "Content-Type: application/json" \
  -d '{"nombreSP":"sp_listar_facturas_y_productosporfactura","p_resultado":null}'
```

### Crear factura
```bash
curl -X POST http://localhost:5035/api/procedimientos/ejecutarsp \
  -H "Content-Type: application/json" \
  -d '{
    "nombreSP":"sp_insertar_factura_y_productosporfactura",
    "p_fkidcliente":1,
    "p_fkidvendedor":1,
    "p_productos":"[{\"codigo\":\"PR001\",\"cantidad\":2}]",
    "p_resultado":null
  }'
```

### Consultar factura
```bash
curl -X POST http://localhost:5035/api/procedimientos/ejecutarsp \
  -H "Content-Type: application/json" \
  -d '{"nombreSP":"sp_consultar_factura_y_productosporfactura","p_numero":1,"p_resultado":null}'
```

### Eliminar factura
```bash
curl -X POST http://localhost:5035/api/procedimientos/ejecutarsp \
  -H "Content-Type: application/json" \
  -d '{"nombreSP":"sp_borrar_factura_y_productosporfactura","p_numero":1,"p_resultado":null}'
```

### SP que no existe (error 400)
```bash
curl -X POST http://localhost:5035/api/procedimientos/ejecutarsp \
  -H "Content-Type: application/json" \
  -d '{"nombreSP":"sp_inexistente","p_resultado":null}'
```
