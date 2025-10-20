# Dashboard + ABM (portable según stack)

[![Estado](https://img.shields.io/badge/status-alpha-blue)]()
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)]()
[![Stack](https://img.shields.io/badge/stack-PHP%20%2B%20MySQL%20%2B%20JS-orange)]()

Sistema **de una sola página** (SPA ligera) con **Dashboard** de avance de proyectos y **ABM** de:
**proyectos**, **estados**, **clientes**, **módulos** (producto/rubro/pedido) y **equipos**.  
Diseñado para cátedras de la **Tecnicatura Universitaria en Programación (UTN)** y para proyectos educativos o institucionales que requieran una arquitectura simple y portable entre lenguajes.

---

## ✨ Captura

![Dashboard](https://olsoft.online/dash/img/dash.jpg)

---

## 🧭 Visión General

- **Patrón:** _Single-File App_ → UI + API en `index.php`.
- **Transporte:** todo por **POST** (sin GET en URLs).
- **Frontend:** HTML + Bootstrap + JS “vanilla” (`fetch()` + `FormData`).
- **Backend:** PHP (mysqli) con **prepared statements** y **CSRF**.
- **Base de datos:** MySQL / InnoDB.

### Flujo de ejecución

```
[UI Buttons] --POST--> index.php (ajax=1, mod, op, data) --SQL--> MySQL
       ↑                                                       ↓
       └------------------ JSON <------------------------------┘
```

---

## 🗂️ Organización del Código

### `index.php`

1. **Setup**
   - `require config/config.php` → inyecta `$con (mysqli)` + `utf8mb4`.
   - `session_start()`, `csrf_token()` y `bootstrap($con)` (crea tablas + datos iniciales).

2. **API interna (POST)**
   - Si llega `$_POST['ajax'] === "1"`:
     - `assert_csrf()` valida token de sesión.
     - Lee `mod`, `op`, `data` (JSON).
     - `switch ("$mod:$op")` ejecuta el CRUD correspondiente y responde JSON.

3. **Render de UI**
   - Sidebar (navegación interna por POST).
   - `<template id="tpl-dashboard">` y `<template id="tpl-table">` (reutilizables).
   - Modal con `<form id="formEntity">` (único form global).
   - Router simple: `loadView('dashboard')`.

---

## 🧑‍💻 Frontend (JavaScript)

- `api(mod, op, data)`: envía `FormData` con `ajax, csrf, mod, op, data(JSON)` al mismo endpoint (`index.php`).
- `viewDashboard()`: solicita `dashboard:metrics` y renderiza tarjetas de estados y proyectos con su progreso.
- `viewTable(title, columns, fetchList, buildForm, saveHandler, deleteHandler)`: **componente genérico** que permite construir cualquier ABM (Estados, Clientes, Equipos, Proyectos, Módulos).

**Guardado desde el modal**
```js
$formEntity.onsubmit = async (e)=>{
  e.preventDefault();
  const data = Object.fromEntries(new FormData($formEntity));
  const ok = await saveHandler(row, data);
  if (ok) { modal.hide(); refresh(); }
};
```

---

## 🗄️ Backend (PHP)

- **Seguridad:** `assert_csrf()` + prepared statements + `utf8mb4`.
- **Funciones auxiliares:**
  - `qall($con, $sql)` → SELECT → array asociativo.
  - `qprep($con, $sql, $types, $params)` → consultas preparadas seguras.
- **Ruteo interno:**  
  `switch ("$mod:$op")` gestiona operaciones de:
  - `estados`, `clientes`, `equipos`, `proyectos`, `modulos`, `dashboard`.
  - Acciones: `list | create | update | delete | metrics`.

---

## 🧱 Modelo de Datos (MySQL)

| Tabla | Campos principales | Relaciones |
|-------|--------------------|-------------|
| **estados** | id, nombre, color | — |
| **clientes** | id, nombre, contacto | — |
| **equipos** | id, nombre, lider, email | — |
| **proyectos** | id, nombre, cliente_id, estado_id, fecha_inicio, fecha_fin, avance | FK cliente_id → clientes(id) (SET NULL)<br>FK estado_id → estados(id) (SET NULL) |
| **modulos** | id, proyecto_id, nombre, tipo, equipo_id, avance | FK proyecto_id → proyectos(id) (CASCADE)<br>FK equipo_id → equipos(id) (SET NULL) |

**Regla de negocio (avance):**
- Si el proyecto **tiene módulos**, el avance se calcula como `AVG(modulos.avance)`.
- Si **no** tiene módulos, se usa `proyectos.avance`.

```sql
SELECT p.id, p.nombre,
       COALESCE((SELECT ROUND(AVG(m.avance),0)
                 FROM modulos m WHERE m.proyecto_id=p.id), p.avance) AS avance
FROM proyectos p;
```

---

## ⚙️ Requisitos y Configuración

### Requisitos
- PHP 8.1 o superior  
- MySQL 8 / MariaDB 10.4+  
- Extensión `mysqli` habilitada

### Configuración mínima (`config/config.php`)
```php
<?php
$con = new mysqli('localhost', 'usuario', 'clave', 'mi_base');
if ($con->connect_errno) {
  die('Error de conexión: ' . $con->connect_error);
}
$con->set_charset('utf8mb4');
```

### Primera ejecución
- `index.php` crea automáticamente las tablas y carga los estados iniciales.
- Acceder desde navegador → `http://localhost/dashboard/index.php`

---

## 🚀 Guía Rápida

1. Clonar el repositorio.  
2. Crear `config/config.php` (ver ejemplo anterior).  
3. Ejecutar en servidor PHP (`php -S localhost:8080` o XAMPP).  
4. Acceder a `index.php`.  
5. Crear **Estados**, **Clientes** y **Equipos**.  
6. Crear un **Proyecto** y luego sus **Módulos** → verificar el avance promedio en el Dashboard.

---

## 🔁 Portabilidad por Stack

> Mantener el **contrato POST único** con `ajax, csrf, mod, op, data(JSON)`.

### Node.js (Express)
- `app.post('/')`: si `req.body.ajax === '1'`, devolver JSON.
- DB: `mysql2` / Knex / Prisma.
- CSRF: `csurf` o token manual en sesión.

### Python (Flask / FastAPI)
- Endpoint `POST '/'` que lea `FormData` y parse JSON.  
- Conectores: `mysql-connector-python`, SQLAlchemy.
- Protección CSRF (Flask-WTF o manual).

### .NET (Minimal APIs)
- `app.MapPost("/")` con `FormData`.  
- DB: `MySqlConnector` + Dapper/EF Core.  
- Antiforgery habilitado.

El **frontend** es completamente reutilizable en cualquier stack.

---

## 🧰 Extensiones Recomendadas

- Soft delete (`deleted_at`) y auditoría (`_trace`).
- Paginación y filtros (estado / fechas / cliente).
- Roles y permisos (login + middleware).
- Validaciones adicionales (rango de `avance`, integridad de FKs).
- Test de contrato con Postman / Insomnia.

---

## 🧑‍🏫 Nota Pedagógica (TUP)

- Demuestra **separación de responsabilidades** sin fragmentar en exceso:
  - **UI** → templates + Bootstrap.  
  - **Controlador (JS)** → router + fetch API.  
  - **Modelo/API** → PHP + SQL.
- Facilita el aprendizaje de **arquitectura de 3 capas** en entorno educativo.
- El contrato API fijo permite que cada grupo **implemente el backend en su lenguaje favorito** (PHP, Node, Python, C#, etc.), manteniendo la misma UI.
- Excelente base para prácticas de integración, seguridad y mantenimiento.

---

## 📦 Estructura Sugerida del Repositorio

```
.
├── index.php
├── config/
│   └── config.php
├── docs/
│   └── screenshot-dashboard.png
├── README.md
└── LICENSE
```

---

## 📝 Versionado

- **v1.02** — versión actual estable  
  - Fix del `FormData` con `$formEntity`.  
  - Corrección de `buildForm` (reemplazo de `formBuilder`).  
  - README profesional + estructura educativa.  
- **Próximos hitos:** filtros, login básico, métricas avanzadas.

---

## 📄 Licencia

Distribuido bajo licencia **MIT** — libre uso y adaptación con fines educativos, de investigación y enseñanza universitaria.
