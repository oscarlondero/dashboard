# dashboard - portable segun stack

## Documento 1 — Arquitectura y Organización del Código
**Proyecto:** Dashboard + ABM en una sola página (PHP + MySQL + JS)

## 1) Visión general
- **Patrón:** Single-File App (UI + API en `index.php`).
- **Transporte:** siempre **POST** (sin GET en URLs).
- **Frontend:** HTML + Bootstrap + JS “vanilla” con `fetch()`.
- **Backend:** PHP (mysqli) con prepared statements y **CSRF**.
- **DB:** MySQL / InnoDB.

### Flujo
```
[UI Buttons] --POST--> index.php (ajax=1, mod, op, data) --SQL--> MySQL
       ↑                                                       ↓
       └------------------ JSON <------------------------------┘
```

## 2) Estructura de `index.php`
1. **Setup**  
   - `require config/config.php` → `$con (mysqli)` + `utf8mb4`.  
   - `session_start`, `csrf_token()` y `bootstrap($con)` para DDL + seeds.

2. **API por POST**  
   Cuando llega `$_POST['ajax']`:
   - `assert_csrf()` valida token.
   - Lee `mod`, `op`, `data` (JSON).
   - `switch ("$mod:$op")` ejecuta CRUD o métricas y responde **JSON**.

3. **Render de UI (HTML)**  
   - Sidebar con botones (POST “interno”).
   - `<template id="tpl-dashboard">` y `<template id="tpl-table">`.
   - Modal con `<form id="formEntity">` (único form reutilizable).
   - JS inicializa vista: `loadView('dashboard')`.

## 3) Frontend (JS)
- `api(mod, op, data)`: envía **FormData** con `ajax, csrf, mod, op, data(JSON)` a la **misma** URL.
- `viewDashboard()`: pide `dashboard:metrics` y dibuja tarjetas de estados y proyectos (con progreso).
- `viewTable(title, columns, fetchList, buildForm, saveHandler, deleteHandler)`: componente genérico para cualquier ABM.
- Modal: el **contenido** cambia con `buildForm(row)`, pero el `<form id="formEntity">` es siempre el mismo.  
  Guardado:
  ```js
  $formEntity.onsubmit = async (e)=>{
    e.preventDefault();
    const data = Object.fromEntries(new FormData($formEntity));
    const ok = await saveHandler(row, data);
    if (ok) { modal.hide(); refresh(); }
  };
  ```

## 4) Backend (PHP)
- Seguridad: `assert_csrf()` + prepared statements + `utf8mb4`.
- Utilidades:
  - `qall($con, $sql)` → SELECT simple → `array` asociativo.
  - `qprep($con, $sql, $types, $params)` → INSERT/UPDATE/DELETE preparados.
- Router de acciones: `switch ("$mod:$op")` con bloques para
  - `estados|clientes|equipos|proyectos|modulos` → `list|create|update|delete`
  - `dashboard:metrics` (consultas agregadas).

## 5) Regla de negocio clave
**Avance de proyecto** = `AVG(modulos.avance)` si hay módulos; si no, usa `proyectos.avance`.

## 6) Buenas prácticas para extender
- Soft delete (`deleted_at`), auditoría (`_trace`), paginación, filtros por estado/fechas.
- Roles/Permisos y login con middleware.
- Pruebas: crear cliente/estado/equipo → proyecto → módulos → validar promedio de avance.

---

**Licencia educativa:** Libre uso y adaptación con fines docentes.

---

## Documento 2 — Contrato de API, Modelo de Datos y Portabilidad
**Proyecto:** Dashboard + ABM en una sola página (PHP + MySQL + JS)

## 1) Contrato de API (POST único a `/index.php`)
**Parámetros (FormData):**
- `ajax` = `"1"`
- `csrf` = `"<token de sesión>"`
- `mod`  ∈ `{estados, clientes, equipos, proyectos, modulos, dashboard}`
- `op`   ∈ `{list, create, update, delete, metrics}`
- `data` = `JSON.stringify({...})`

**Respuesta JSON estándar:**
```json
{ "ok": true, "items": [...], "id": 123, "error": "..." }
```

### 1.1 Ejemplos
- **Listar proyectos**
```
ajax=1 & csrf=... & mod=proyectos & op=list & data={}
```
- **Crear módulo** (`modulos:create`)
```json
{
  "proyecto_id": 3,
  "nombre": "ABM Productos",
  "tipo": "producto",
  "equipo_id": 2,
  "avance": 40
}
```
- **Actualizar estado** (`estados:update`)
```json
{ "id": 5, "nombre": "En pruebas", "color": "#fd7e14" }
```

## 2) Modelo de Datos (MySQL)
### 2.1 Tablas
- **estados** `(id, nombre, color)`
- **clientes** `(id, nombre, contacto)`
- **equipos** `(id, nombre, lider, email)`
- **proyectos** `(id, nombre, cliente_id*, estado_id*, fecha_inicio, fecha_fin, avance)`  
  FKs: `cliente_id → clientes(id)` (SET NULL), `estado_id → estados(id)` (SET NULL)
- **modulos** `(id, proyecto_id*, nombre, tipo{producto|rubro|pedido}, equipo_id*, avance)`  
  FKs: `proyecto_id → proyectos(id)` (CASCADE), `equipo_id → equipos(id)` (SET NULL)

### 2.2 Cálculo de avance
```sql
SELECT p.id, p.nombre,
       COALESCE((SELECT ROUND(AVG(m.avance),0)
                 FROM modulos m WHERE m.proyecto_id=p.id), p.avance) AS avance
FROM proyectos p;
```

## 3) Snippets clave
### 3.1 JS — función API
```js
async function api(mod, op, data={}) {
  const fd = new FormData();
  fd.append('ajax','1'); fd.append('csrf', CSRF);
  fd.append('mod', mod); fd.append('op', op);
  fd.append('data', JSON.stringify(data));
  const res = await fetch('', { method:'POST', body: fd });
  return await res.json();
}
```

### 3.2 PHP — utilidades DB
```php
function qall(mysqli $db, string $sql): array {
  $rs = $db->query($sql);
  if (!$rs) throw new RuntimeException($db->error);
  $out = [];
  while ($row = $rs->fetch_assoc()) { $out[] = $row; }
  $rs->free();
  return $out;
}
function qprep(mysqli $db, string $sql, string $types, array $params): mysqli_stmt {
  $st = $db->prepare($sql);
  if (!$st) throw new RuntimeException($db->error);
  if ($types !== '') { $st->bind_param($types, ...$params); }
  if (!$st->execute()) throw new RuntimeException($st->error);
  return $st;
}
```

## 4) Portabilidad a otros stacks
> Mantener **el mismo contrato** (`ajax, csrf, mod, op, data(JSON)` por POST a una **única URL**).

### 4.1 Node.js (Express)
- Ruta `app.post('/')`: si `req.body.ajax === '1'` → JSON; de lo contrario, HTML.
- CSRF con `csurf` o token manual de sesión.
- DB con `mysql2`/Knex/Prisma.

### 4.2 Python (Flask / FastAPI)
- Endpoint `POST '/'` con lectura de FormData y JSON.
- CSRF (Flask-WTF) o token manual.
- Conectores: `mysql-connector-python` / SQLAlchemy.

### 4.3 .NET (Minimal APIs)
- `app.MapPost("/")` para FormData.
- Antiforgery de ASP.NET Core.
- `MySqlConnector` + Dapper/EF Core.

## 5) Extensiones recomendadas
- Soft delete y auditoría (`_trace`).
- Paginación y filtros por estado / fechas.
- Roles/Permisos y autenticación.
- Validaciones del lado del servidor y del cliente.
- Tests de integración del contrato (Postman/Insomnia).

---

**Licencia educativa:** Libre uso y adaptación con fines docentes.

