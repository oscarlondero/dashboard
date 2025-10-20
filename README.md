# dashboard
Dashboard portable segun stack
# Documento 1 — Arquitectura y Organización del Código
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
