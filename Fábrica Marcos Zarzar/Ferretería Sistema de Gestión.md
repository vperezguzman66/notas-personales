---
proyecto: "Ferreteria"
ruta: "Ferreteria"
cliente: "Marcos / Diego"
stack: "Electron + React + TypeScript + SQLite"
estado: "Activo — módulo comercial completo, hardening v3"
ultimo_cambio: 2026-06-26
---

[[Marcos - Diego]]

Aplicación de escritorio para gestión de ventas, productos, costos y clientes en una ferretería chilena. Construida con Electron + React + SQLite, funciona completamente offline.

---

## Stack tecnológico

[](https://github.com/vperezguzman66/Ferreteria#stack-tecnol%C3%B3gico)

|Capa|Tecnología|
|---|---|
|Shell de escritorio|Electron 30|
|Bundler/dev server|electron-vite 5 + Vite 7|
|UI|React 18 + TypeScript 5|
|Base de datos|SQLite via better-sqlite3 12|
|Iconos|lucide-react|

---

## Estado técnico (2026-06-08)

[](https://github.com/vperezguzman66/Ferreteria#estado-t%C3%A9cnico-2026-06-08)

**Módulo comercial** implementado por fases T1..T5 (Vendedores, Cartera, Gestiones, Ventas atribuibles, Dashboard Comercial, exportes CSV y SOP).

**Hardening de seguridad y deuda técnica** (sprints 1–3):

- Seguridad base Electron + IPC: `contextIsolation`, `sandbox`, `nodeIntegration: false`, allowlist IPC, validación de origen y rate limiting.
- Guardia de rol en IPC mediante `APP_ROLE` con default `admin`.
- **v2:** modo de autorización por sesión (`APP_AUTH_MODE=session`) con login/logout/me vía IPC y tabla `app_users` con hash `scrypt`.
- **v2.1:** bootstrap admin endurecido; lockout anti brute-force; `AuthGate` en renderer.
- **CSP** activa en dev y prod: `onHeadersReceived` registrado incondicionalmente con ternario dev/prod.
- **Migraciones versionadas** (`PRAGMA user_version`): 6 bloques incrementales (v1–v6) en una única transacción atómica. Sin `ALTER TABLE` en arranques limpios; sin `PRAGMA table_info` en producción.
- **Índices de BD** (v5): 32 índices creados en migración, cubriendo FKs y columnas de alta frecuencia.
- **Transacciones multi-tabla**: `productos:create`, `productos:update`, `ventas:create`, `ventas:anular`, `cartera:reasignarCliente`, `zonas:comunas:set` envueltos en `db.transaction()`.
- **Paginación server-side** en `cartera:list` (LIMIT/OFFSET + total); paginación client-side en `ventas:list` y `productos:list`.
- **Audit log** (`audit_log` v6): toda acción IPC lleva `actor` via `AsyncLocalStorage`; eventos de auth y operaciones críticas persisten en BD.
- **Suite de tests de regresión**: 51 tests estáticos de análisis de fuente, sin dependencias externas en runtime de tests.
- **Windows**: build NSIS listo para despliegue en Windows 10/11.

---

## Estructura del proyecto

[](https://github.com/vperezguzman66/Ferreteria#estructura-del-proyecto)

```
src/
├── main/
│   ├── index.ts       — Bootstrap app, seguridad IPC, registro de handlers
│   ├── db.ts          — Conexión SQLite, esquema y migraciones versionadas (v1-v6)
│   ├── audit.ts       — auditActorStore (AsyncLocalStorage) + logAuditEvent
│   └── ipc/
│       ├── common/
│       │   ├── public-error.ts   — Error público compartido para IPC
│       │   └── validators.ts     — Validaciones compartidas de entrada
│       └── handlers/
│           ├── dashboard.ts      — Handler `dashboard:summary`
│           ├── categorias.ts     — Handlers `categorias:*`
│           ├── maestros.ts       — Handlers `materias:*` y `clientes:*`
│           ├── productos.ts      — Handlers `productos:*`
│           ├── comercial.ts      — Handlers `zonas:*`, `vendedores:*`, `cartera:*`, `gestiones:*`
│           └── ventas.ts         — Handlers `ventas:*`
├── preload/
│   └── index.ts       — Bridge seguro entre main y renderer
└── renderer/src/
    ├── App.tsx         — Layout y navegación
    ├── index.css       — Sistema de diseño (variables CSS, componentes)
    ├── types.ts        — Interfaces TypeScript compartidas
    ├── utils.ts        — Formateo CLP, cálculo de márgenes
    ├── pages/
    │   ├── Ventas.tsx         — POS + historial de ventas
    │   ├── Clientes.tsx       — Gestión de clientes con RUT
    │   ├── Vendedores.tsx     — Gestión de fuerza de ventas
    │   ├── Cartera.tsx        — Cartera comercial y reasignación
    │   ├── Gestiones.tsx      — Agenda e historial comercial
    │   ├── Productos.tsx      — Gestión de productos e inventario
    │   ├── Familias.tsx       — Gestión de categorías
    │   └── MateriasPrimas.tsx — Gestión de insumos base
    └── components/
        ├── ProductoModal.tsx  — Formulario crear/editar producto
        └── HistorialModal.tsx — Historial de precios de un producto
tests/
├── audit-log-regression.test.mjs
├── comercial-hardening-regression.test.mjs
├── db-indices-regression.test.mjs
├── db-schema-version-regression.test.mjs
├── db-transactions-regression.test.mjs
├── handler-invariants.test.mjs
└── ventas-stock-regression.test.mjs   (+ otros tests de seguridad)
```

---

## Base de datos

[](https://github.com/vperezguzman66/Ferreteria#base-de-datos)

Archivo SQLite en:

- **macOS:** `~/Library/Application Support/ferreteria/ferreteria.db`
- **Windows:** `%APPDATA%\ferreteria\ferreteria.db`

### Migraciones versionadas

[](https://github.com/vperezguzman66/Ferreteria#migraciones-versionadas)

Al arrancar, `db.ts` lee `PRAGMA user_version`. Si es menor a `SCHEMA_VERSION` (actualmente **6**), ejecuta los bloques faltantes dentro de una única transacción atómica y actualiza `user_version`. No se pierden datos al actualizar.

|Versión|Contenido|
|---|---|
|v1|`productos.sku`, `productos.stock_minimo`, `categorias.parent_id`|
|v2|Columnas comerciales de `clientes` (10 columnas: vendedor, zona, clasificación, etc.)|
|v3|Columnas comerciales de `ventas` (`vendedor_id`, `zona_id`, `canal_venta`, `gestion_id`)|
|v4|`app_users` extendido (`role`, `activo`, `updated_at`, `last_login_at`)|
|v5|32 índices en FKs y columnas de alta frecuencia|
|v6|Tabla `audit_log` + índices por timestamp, actor y action|

### Esquema principal

[](https://github.com/vperezguzman66/Ferreteria#esquema-principal)

```sql
-- Catálogo
categorias (id, nombre UNIQUE, parent_id → categorias, created_at)

materias_primas (id, nombre, unidad, costo_por_unidad, stock, created_at)

-- Productos
productos (id, nombre, categoria_id, tipo IN('simple','formula'),
           costo_manual, precio_venta, stock, stock_minimo, unidad, sku, created_at)

formula_items (id, producto_id, materia_prima_id, cantidad)
  -- CASCADE DELETE desde productos

historial_precios (id, producto_id, costo, precio_venta, fecha)
  -- snapshot al crear/editar producto

-- Fuerza de ventas
zonas (id, nombre, activo)
zona_comunas (id, zona_id, comuna)
vendedores (id, nombre, telefono, email, codigo, zona_principal_id,
            meta_mensual, comision_pct, activo, observaciones, updated_at)

-- Clientes
clientes (id, nombre, rut, telefono, email,
          vendedor_id, zona_id, comuna, clasificacion,
          frecuencia_visita, canal_principal, estado_comercial,
          proxima_visita, ultima_visita, observaciones_comerciales, created_at)

auditoria_asignaciones (id, cliente_id, vendedor_anterior_id,
                        vendedor_nuevo_id, motivo, actor, fecha)

-- Gestiones comerciales
gestiones_comerciales (id, cliente_id, vendedor_id, zona_id,
                       tipo, fecha, proxima_fecha, resultado,
                       estado_agenda IN('pendiente','realizada','reagendada'),
                       notas, venta_id)

-- Ventas
ventas (id, numero, cliente_id, vendedor_id, zona_id,
        canal_venta, gestion_id, fecha,
        subtotal, descuento_global_pct, neto,
        iva,   -- neto × 0.19
        total, -- neto + iva
        medio_pago, estado IN('completada','anulada'), notas)

venta_items (id, venta_id, producto_id,
             nombre_producto, sku,       -- snapshot
             cantidad, precio_unitario, descuento_pct, subtotal)

-- Seguridad
app_users (id, username UNIQUE, password_hash, role, activo,
           failed_attempts, locked_until, updated_at, last_login_at)

audit_log (id, ts DEFAULT now, actor, action, target, detail JSON)
```

**Tipos de producto:**

- `simple` — costo ingresado manualmente (`costo_manual`)
- `formula` — costo calculado sumando `cantidad × costo_por_unidad` de cada `formula_item`

**Precios:** todos los `precio_venta` y `precio_unitario` se almacenan como precio **neto** (sin IVA). El IVA 19% se calcula y persiste al registrar una venta (`iva = Math.round(neto * 0.19)`).

---

## Variables de entorno

[](https://github.com/vperezguzman66/Ferreteria#variables-de-entorno)

Configuradas en `.env` (copiar desde `.env.example`):

```shell
# Modo de autenticación: 'env' (sin login) | 'session' (con login/logout)
APP_AUTH_MODE=env

# Rol activo en modo 'env': admin | supervisor | vendedor | caja
APP_ROLE=admin

# Usuario y contraseña del admin bootstrap (solo en APP_AUTH_MODE=session)
APP_ADMIN_USER=admin
APP_ADMIN_PASSWORD=          # Obligatorio, mínimo 12 caracteres

# Brute-force lockout (modo session)
AUTH_LOGIN_MAX_ATTEMPTS=5
AUTH_LOGIN_WINDOW_MS=300000   # 5 minutos
AUTH_LOGIN_LOCK_MS=600000     # 10 minutos

# Logs de seguridad
SECURITY_LOG_PATH=            # Vacío = userData/security.log
SECURITY_LOG_MAX_BYTES=1048576
SECURITY_LOG_BACKUPS=3

# Rate limiting IPC
IPC_RATE_LIMIT_READ_MAX=240
IPC_RATE_LIMIT_WRITE_MAX=60
```

En **Windows**, si no se usa `.env`, las variables pueden setearse como variables de entorno del sistema antes de lanzar el `.exe`.

---

## API IPC (main ↔ renderer)

[](https://github.com/vperezguzman66/Ferreteria#api-ipc-main--renderer)

Todos los handlers son `ipcMain.handle` (async, invocados via `window.api.*`).

### Categorías

[](https://github.com/vperezguzman66/Ferreteria#categor%C3%ADas)

|Canal|Parámetros|Retorna|
|---|---|---|
|`categorias:list`|—|`Categoria[]`|
|`categorias:create`|`{ nombre }`|`Categoria`|
|`categorias:update`|`{ id, nombre }`|`Categoria`|
|`categorias:delete`|`id`|`{ ok: true }`|

### Materias Primas

[](https://github.com/vperezguzman66/Ferreteria#materias-primas)

|Canal|Parámetros|Retorna|
|---|---|---|
|`materias:list`|—|`MateriaPrima[]`|
|`materias:create`|`{ nombre, unidad, costo_por_unidad, stock }`|`MateriaPrima`|
|`materias:update`|`{ id, nombre, unidad, costo_por_unidad, stock }`|`MateriaPrima`|
|`materias:delete`|`id`|`{ ok: true }`|

### Productos

[](https://github.com/vperezguzman66/Ferreteria#productos)

|Canal|Parámetros|Retorna|
|---|---|---|
|`productos:list`|—|`Producto[]` (con `costo_calculado` y `formula_items`)|
|`productos:create`|`Producto` (sin id)|`{ id }`|
|`productos:update`|`Producto`|`{ ok: true }`|
|`productos:delete`|`id`|`{ ok: true }`|
|`productos:historial`|`id`|`HistorialPrecio[]` (últimos 50)|

`productos:create` y `productos:update` registran automáticamente en `historial_precios`.

### Clientes

[](https://github.com/vperezguzman66/Ferreteria#clientes)

|Canal|Parámetros|Retorna|
|---|---|---|
|`clientes:list`|—|`Cliente[]`|
|`clientes:create`|`{ nombre, rut?, telefono?, email? }`|`Cliente`|
|`clientes:update`|`{ id, nombre, rut?, telefono?, email? }`|`Cliente`|
|`clientes:delete`|`id`|`{ ok: true }`|

### Zonas

[](https://github.com/vperezguzman66/Ferreteria#zonas)

|Canal|Parámetros|Retorna|
|---|---|---|
|`zonas:list`|`{ includeInactive? }`|`Zona[]`|
|`zonas:create`|`{ nombre, activo? }`|`Zona`|
|`zonas:update`|`{ id, nombre, activo }`|`Zona`|
|`zonas:delete`|`id`|`{ ok: true }` (desactivación lógica)|
|`zonas:comunas:list`|`zonaId?`|`ZonaComuna[]`|
|`zonas:comunas:set`|`{ zona_id, comunas[] }`|`ZonaComuna[]`|

### Vendedores

[](https://github.com/vperezguzman66/Ferreteria#vendedores)

|Canal|Parámetros|Retorna|
|---|---|---|
|`vendedores:list`|`{ includeInactive?, zona_id? }`|`Vendedor[]`|
|`vendedores:create`|`VendedorInput`|`Vendedor`|
|`vendedores:update`|`VendedorInput & { id }`|`Vendedor`|
|`vendedores:delete`|`id`|`{ ok: true }` (desactivación lógica)|
|`vendedores:stats`|`id`|`VendedorStats`|

### Cartera

[](https://github.com/vperezguzman66/Ferreteria#cartera)

|Canal|Parámetros|Retorna|
|---|---|---|
|`cartera:list`|`CarteraFilters`|`{ rows: CarteraRow[], total: number }`|
|`cartera:updateClienteComercial`|`ClienteComercialInput`|`Cliente`|
|`cartera:reasignarCliente`|`{ cliente_id, vendedor_nuevo_id, motivo? }`|`{ ok: true }`|

`cartera:list` usa LIMIT/OFFSET server-side (default `limit: 25`, máx `200`). `cartera:reasignarCliente` registra en `auditoria_asignaciones` y `audit_log`.

### Gestiones

[](https://github.com/vperezguzman66/Ferreteria#gestiones)

|Canal|Parámetros|Retorna|
|---|---|---|
|`gestiones:list`|`GestionFilters`|`Gestion[]`|
|`gestiones:create`|`GestionInput`|`Gestion`|
|`gestiones:update`|`GestionInput & { id }`|`Gestion`|
|`gestiones:cerrar`|`{ id, resultado, proxima_fecha? }`|`Gestion`|

### Ventas

[](https://github.com/vperezguzman66/Ferreteria#ventas)

|Canal|Parámetros|Retorna|
|---|---|---|
|`ventas:list`|—|`Venta[]` (con atribución comercial)|
|`ventas:get`|`id`|`Venta & { items: VentaItem[] }`|
|`ventas:create`|`VentaInput`|`{ id, numero }`|
|`ventas:anular`|`id`|`{ ok: true }`|

`ventas:create` y `ventas:anular` usan `db.transaction()` — stock se actualiza atómicamente. `ventas:anular` escribe en `audit_log`.

---

## Módulos

[](https://github.com/vperezguzman66/Ferreteria#m%C3%B3dulos)

### Ventas (POS)

[](https://github.com/vperezguzman66/Ferreteria#ventas-pos)

- Lista con filtros por período (hoy / semana / mes / todo) y stats de recaudación (total con IVA, neto)
- **POS:** búsqueda de productos por nombre o SKU, Enter para agregar el primer resultado
- Carrito editable: cantidad (fracciones), precio unitario y descuento % por línea
- Descuento global sobre el total
- Cálculo en tiempo real: subtotal → neto → IVA 19% → total
- Forma de pago: efectivo (con vuelto automático), débito, crédito, transferencia, otro
- Al cobrar: boleta modal con opción de imprimir (`window.print()` con `@media print`)
- Anular venta: restaura el stock de cada ítem automáticamente
- Atribución comercial: hereda `vendedor_id` y `zona_id` del cliente si no se especifican
- Atajo **Registrar gestión** para abrir `Gestiones` precompletado y vinculado por `venta_id`

### Clientes

[](https://github.com/vperezguzman66/Ferreteria#clientes-1)

- CRUD completo con búsqueda por nombre o RUT
- Validación de RUT chileno con algoritmo de dígito verificador
- Asociación opcional a cada venta
- Atajo **Registrar gestión** por fila

### Vendedores

[](https://github.com/vperezguzman66/Ferreteria#vendedores-1)

- Listado con filtros por estado y zona
- Alta, edición y desactivación lógica
- KPIs operativos por vendedor (`vendedores:stats`)
- Navegación rápida hacia `Cartera` filtrada

### Cartera

[](https://github.com/vperezguzman66/Ferreteria#cartera-1)

- Paginación server-side (25 por página, máx 200)
- Filtros: vendedor, zona, clasificación, búsqueda libre, sin compra N días, visita pendiente
- Edición de bloque comercial del cliente
- Reasignación cliente–vendedor con registro en auditoría
- Exportar a CSV (todos los resultados, límite 10 000)

### Gestiones

[](https://github.com/vperezguzman66/Ferreteria#gestiones-1)

- Vista agenda y vista historial con filtros
- Crear gestión, cerrar con resultado obligatorio
- `reagendada` y `proxima_accion` requieren `proxima_fecha`
- Prefill de creación desde `Clientes`, `Cartera` y `Ventas`

### Productos

[](https://github.com/vperezguzman66/Ferreteria#productos-1)

- Dos tipos de costo: `simple` (manual) o `formula` (suma de materias primas)
- Campos: nombre, SKU, categoría, unidad, stock actual, stock mínimo
- Toggle "Precio neto / Precio c/IVA (19%)" en tabla
- Indicador ⚠️ en stock bajo mínimo
- Vista plana y vista agrupada con margen promedio por grupo
- Historial de precios (últimos 50) por producto
- Paginación client-side (40 por página)

### Familias (Categorías)

[](https://github.com/vperezguzman66/Ferreteria#familias-categor%C3%ADas)

- Stats: total familias, productos sin familia
- Tabla: nombre, conteo de productos, margen promedio, valor del stock
- Bloquea eliminación si tiene productos asignados

### Materias Primas

[](https://github.com/vperezguzman66/Ferreteria#materias-primas-1)

- Buscador en tiempo real por nombre
- Tabla: nombre, unidad, costo por unidad, stock
- Unidades: kg, g, litro, ml, unidad, metro, cm, caja, bolsa, galón

---

## Lógica de márgenes

[](https://github.com/vperezguzman66/Ferreteria#l%C3%B3gica-de-m%C3%A1rgenes)

```
margen = ((precio_venta - costo) / costo) × 100
```

Semáforo visual:

- ≥ 30% → verde (`margin-good`)
- 10–29% → amarillo (`margin-warn`)
- < 10% → rojo (`margin-bad`)

---

## Suite de tests

[](https://github.com/vperezguzman66/Ferreteria#suite-de-tests)

```shell
npm test          # node --test tests/*.test.mjs
```

51 tests de análisis estático (lectura de fuentes `.ts` + regex), sin dependencias de runtime:

|Archivo|Tests|Cubre|
|---|---|---|
|`ventas-stock-regression`|4|Transacciones ventas, stock, validaciones, atribución|
|`comercial-hardening-regression`|2|Validación gestiones, CTEs dashboard|
|`db-indices-regression`|1|32 índices en migración v5|
|`db-schema-version-regression`|4|PRAGMA user_version, bloques v1-v6, transacción|
|`db-transactions-regression`|3|Transacciones productos, zonas|
|`audit-log-regression`|5|Tabla audit_log, AsyncLocalStorage, eventos auth/negocio|
|`handler-invariants`|15|Paginación cartera, reasignación, IVA, validaciones|
|_(seguridad)_|17|CSP, allowlist IPC, auth, rate-limit, etc.|

---

## Cómo ejecutar

[](https://github.com/vperezguzman66/Ferreteria#c%C3%B3mo-ejecutar)

### Primera vez (instalación limpia)

[](https://github.com/vperezguzman66/Ferreteria#primera-vez-instalaci%C3%B3n-limpia)

```shell
npm install
# Compilar better-sqlite3 para el ABI de Electron 30
npx @electron/rebuild -f -w better-sqlite3
```

### Desarrollo

[](https://github.com/vperezguzman66/Ferreteria#desarrollo)

```shell
npm run dev
```

### Build de producción — macOS

[](https://github.com/vperezguzman66/Ferreteria#build-de-producci%C3%B3n--macos)

```shell
npm run build
npx electron-builder --mac     # genera .dmg en /release
```

### Build de producción — Windows

[](https://github.com/vperezguzman66/Ferreteria#build-de-producci%C3%B3n--windows)

Ver [docs/DESPLIEGUE_WINDOWS.md](https://github.com/vperezguzman66/Ferreteria/blob/main/docs/DESPLIEGUE_WINDOWS.md) para instrucciones detalladas.

```shell
# Desde Windows (recomendado):
npm install
npm run build
npx electron-builder --win     # genera instalador NSIS .exe en /release
```

---

## Roadmap

[](https://github.com/vperezguzman66/Ferreteria#roadmap)

### Fase 3 — Proveedores y compras

[](https://github.com/vperezguzman66/Ferreteria#fase-3--proveedores-y-compras)

- Ficha de proveedor (RUT, datos bancarios, condiciones de pago)
- Órdenes de compra numeradas con envío por email
- Recepción de mercadería referenciada a la OC → actualiza stock automáticamente
- Historial de precios de compra por proveedor

### Fase 4 — Reportes y Dashboard

[](https://github.com/vperezguzman66/Ferreteria#fase-4--reportes-y-dashboard)

- Dashboard: ventas del día, stock crítico, últimas transacciones
- Reporte de ventas por período (con filtro por medio de pago y vendedor)
- Stock valorizado al costo
- Productos sin movimiento (stock muerto)

### Fase 5 — DTE / SII (a evaluar)

[](https://github.com/vperezguzman66/Ferreteria#fase-5--dte--sii-a-evaluar)

- Boleta electrónica y factura electrónica (requiere integración con proveedor DTE o stack propio)
- Opciones: API de Factura.cl / Bsale, o implementación directa del formato XML del SII

---

## Documentación técnica

[](https://github.com/vperezguzman66/Ferreteria#documentaci%C3%B3n-t%C3%A9cnica)

Centralizada en `docs/` (índice: `docs/README.md`).

- `docs/DESPLIEGUE_WINDOWS.md` — Guía de instalación en Windows
- `docs/RELEASE_CHECKLIST.md` — Checklist operativo de release
- `docs/SECURITY_PLAYBOOK.md` — Respuesta a incidentes de seguridad
- `docs/OPERACION_BACKUPS_DB.md` — Estrategia de backups SQLite
- `docs/IPC_CHANNEL_ALLOWLIST.md` — Lista blanca de canales IPC
- `docs/OBSERVABILIDAD_ALERTAS_SEGURIDAD.md` — Alertas operativas