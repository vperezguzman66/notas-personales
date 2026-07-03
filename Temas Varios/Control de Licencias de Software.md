[[Varios]]
# LicenseControl

[](https://github.com/vperezguzman66/license-control#licensecontrol)

App de escritorio para gestionar licencias de software en pequeñas y medianas empresas. Registra, monitorea y recibe alertas sobre vencimientos, con cifrado de claves, trazabilidad de asignaciones y log de auditoría completo.

## Documentación estratégica

[](https://github.com/vperezguzman66/license-control#documentaci%C3%B3n-estrat%C3%A9gica)

- [Roadmap y estado de implementación](https://github.com/vperezguzman66/license-control/blob/main/docs/ROADMAP.md) — qué está hecho, qué falta y prioridades

## Características

[](https://github.com/vperezguzman66/license-control#caracter%C3%ADsticas)

### Gestión de licencias

[](https://github.com/vperezguzman66/license-control#gesti%C3%B3n-de-licencias)

- **Inventario completo** — alta, edición y eliminación de licencias con búsqueda y filtros por estado y tipo
- **Asignaciones por licencia** — registra qué usuario/equipo usa cada licencia; conteo en tiempo real contra el total disponible con alerta visual si se supera el límite
- **Copiar clave** — botón de portapapeles en cada fila para copiar el license key con un clic
- **Importar CSV** — importación masiva de licencias desde archivo CSV (máx. 2MB / 10.000 filas)
- **Exportación** — descarga el inventario filtrado en CSV (compatible con Excel) o PDF

### Proveedores

[](https://github.com/vperezguzman66/license-control#proveedores)

- CRUD completo con RUT formateado y validado con algoritmo mod-11, teléfono, contacto, email y sitio web
- Búsqueda en tiempo real por nombre, RUT, contacto o email

### Alertas y notificaciones

[](https://github.com/vperezguzman66/license-control#alertas-y-notificaciones)

- **Alertas** — listado de licencias vencidas, licencias por vencer y renovaciones pendientes en los próximos 30 días
- **Badge en sidebar** — rojo si hay vencidas, ámbar si solo hay próximas a vencer
- **Notificaciones del SO** — al abrir la app notifica licencias vencidas, por vencer y renovaciones pendientes

### Seguridad

[](https://github.com/vperezguzman66/license-control#seguridad)

- **Login con rate limiting** — bloqueo de 30 segundos tras 5 intentos fallidos; contador activo en la UI
- **Política de contraseñas** — mínimo 8 caracteres, al menos una mayúscula y un número; validada en backend y con hint en UI
- **Roles de acceso** — `admin` (acceso total + usuarios), `editor` (crear/editar), `viewer` (solo lectura); aplicados en el proceso principal, no solo en la UI
- **Hardening de navegación Electron** — se bloquean `window.open`, `webview` y navegaciones fuera de orígenes permitidos (`localhost` en dev y `file://` en producción)
- **Cifrado AES-256-GCM** — las claves de licencia se almacenan cifradas en la BD; cada registro usa un IV único; backup incluye la llave de cifrado
- **Contraseñas** — hash `scrypt` + salt aleatorio de 16 bytes; comparación con `crypto.timingSafeEqual`
- **CSP reforzada** — política en producción sin `unsafe-inline`, con `object-src 'none'`, `base-uri 'none'`, `frame-ancestors 'none'` y `form-action 'self'`
- **Queries parametrizadas** — sin interpolación de strings en ninguna consulta SQL
- **Adjuntos protegidos** — apertura de archivos validada por ID de adjunto y ruta canónica dentro de `attachmentsDir` (mitiga path traversal)
- **Minimización de datos sensibles por rol** — `viewer` no recibe `license_key` en claro en `db:getLicenses`

### Compliance SAM / ELP

[](https://github.com/vperezguzman66/license-control#compliance-sam--elp)

- **Effective License Position** — calcula automáticamente `entitled − consumed` por producto y métrica; estados `compliant`, `under_licensed` y `over_licensed`
- **Recálculo automático ELP** — se recalcula tras crear/eliminar asignaciones, crear/editar/eliminar entitlements o importar CSV; operación no crítica (no bloquea la mutación principal)
- **Entitlements** — registro formal de derechos de licencia con product_key, métrica, cantidad, vigencia, contrato y PO
- **Optimización de costos** — identifica licencias sobredimensionadas con ahorro estimado proporcional; lista licencias con costo sin entitlement registrado
- **Calidad de datos** — wizard de 4 pasos con score 0–100; detecta vacíos críticos (proveedor, fechas, product_key, cupos), duplicados normalizados, licencias activas sin entitlement y entitlements vencidos
- **Score de completitud** — porcentaje por licencia (proveedor, fecha de compra, costo, license key; vencimiento para subscriptions/trials); promedio global en el Dashboard con mini-barra en la tabla de licencias
- **Score de madurez SAM** — evalúa 4 áreas ponderadas (Inventario 25%, Compliance 35%, Renovaciones 20%, Calidad de datos 20%); niveles Inicial / En desarrollo / Definido / Optimizado con puntos de mejora expandibles por área
- **Catálogo normalizado** — sincroniza licencias y entitlements en un product_key canónico; detecta candidatos a consolidación por similitud de nombre y proveedor
- **Fusión asistida real** — permite ejecutar merge operativo de productos candidatos (actualiza licencias, entitlements y catálogo)
- **SAM Tasks** — crea tareas accionables desde riesgos ELP y candidatos de consolidación, con prioridad automática y workflow de estados
- **Export audit pack** — exporta paquete de evidencia en JSON + 3 archivos CSV (compliance, tasks, audit log) listo para auditoría de vendor
- **Export SBOM** — genera Software Bill of Materials en formato SPDX 2.3 JSON y CycloneDX 1.4 XML desde el catálogo de licencias

### Descubrimiento

[](https://github.com/vperezguzman66/license-control#descubrimiento)

- **Importar CSV o JSON** — importa inventario de software instalado o listado de usuarios desde exportaciones de OCS Inventory, GLPI, Lansweeper u otras herramientas (máx. 5 MB / 50.000 filas)
- **Auto-detección de columnas** — detecta automáticamente los campos relevantes por nombre de cabecera; ajuste manual mediante selectores
- **Reconciliación de software** — cruza software detectado contra licencias registradas; estados `sin_licencia`, `excede_seats`, `sin_detección` y `cubierto`
- **Reconciliación de usuarios** — cruza usuarios importados contra asignaciones activas; estados `sin_asignación`, `asignación sin usuario` y `cubierto`
- **Acciones directas** — crea licencias faltantes (pre-rellena LicenseModal con datos detectados) o asigna usuarios descubiertos a licencias existentes desde la misma pantalla

### Workflow de renovaciones

[](https://github.com/vperezguzman66/license-control#workflow-de-renovaciones)

- **Estados de renovación** — `pending`, `in_review`, `approved`, `cancelled` con badge de color; selector inline para editor/admin
- **Responsable asignado** — campo de usuario propietario de la renovación (select de usuarios del sistema)
- **Checklist de aprobación** — 4 ítems por defecto (cotización, presupuesto, contrato, orden de compra); toggle individual con persistencia inmediata
- **Alertas proactivas** — sección "Renovaciones pendientes en 30 días" en la página de Alertas con badge de urgencia

### Administración (solo admin)

[](https://github.com/vperezguzman66/license-control#administraci%C3%B3n-solo-admin)

- **Usuarios** — gestión de cuentas y roles; protección del último administrador
- **Log de auditoría** — registro inmutable de todas las acciones (crear, editar, eliminar licencias/usuarios/proveedores/asignaciones, login, logout, backup, restore, importación CSV)
- **Backup y restore** — exporta e importa la BD completa con diálogo nativo; el backup incluye la llave de cifrado para ser autocontenido

## Tech Stack

[](https://github.com/vperezguzman66/license-control#tech-stack)

|Capa|Tecnología|
|---|---|
|App de escritorio|Electron 42|
|UI|React 19 + Vite + Tailwind CSS|
|Iconos|@tabler/icons-react|
|Base de datos|sql.js (SQLite en JS puro — sin compilación nativa)|
|Cifrado|Node.js `crypto` — AES-256-GCM|
|Exportación|CSV (compatible con Excel, sin dependencias) + jsPDF + jspdf-autotable|
|Empaquetado|electron-builder|

## Requisitos

[](https://github.com/vperezguzman66/license-control#requisitos)

- Node.js 18+
- npm 9+

## Instalación (entorno de desarrollo)

[](https://github.com/vperezguzman66/license-control#instalaci%C3%B3n-entorno-de-desarrollo)

```shell
# 1. Dependencias raíz (Electron + sql.js)
npm install

# 2. Dependencias del renderer (React + electron-builder)
cd licensecontrol && npm install
```

## Desarrollo

[](https://github.com/vperezguzman66/license-control#desarrollo)

```shell
cd licensecontrol
npm run electron:dev
```

Inicia Vite en `http://localhost:5173` y Electron en paralelo con DevTools abierto.

**Primer acceso seguro:** en una instalación nueva se crea `admin` con contraseña aleatoria de bootstrap, guardada en `BOOTSTRAP_ADMIN_CREDENTIALS.txt` dentro del directorio `userData` (permisos `0600`). Cámbiala inmediatamente después del primer inicio de sesión.

---

## Distribución e instaladores

[](https://github.com/vperezguzman66/license-control#distribuci%C3%B3n-e-instaladores)

Esta sección cubre todo lo necesario para empaquetar y distribuir la app como instalador nativo.

### Cómo generar un instalador

[](https://github.com/vperezguzman66/license-control#c%C3%B3mo-generar-un-instalador)

Los instaladores se generan desde la carpeta `licensecontrol/`:

```shell
# macOS — genera un .dmg universal (Intel + Apple Silicon)
cd licensecontrol && npm run dist:mac

# Windows — genera un instalador NSIS .exe
cd licensecontrol && npm run dist:win

# Linux — genera un AppImage
cd licensecontrol && npm run dist:linux

# Plataforma actual (detecta automáticamente)
cd licensecontrol && npm run dist
```

> **Importante:** para generar el instalador de Windows desde macOS se necesita Wine o correrlo directamente desde una máquina Windows. Lo mismo aplica a la inversa.

El resultado queda en `dist-electron/` en la raíz del proyecto.

### Estructura del output

[](https://github.com/vperezguzman66/license-control#estructura-del-output)

```
dist-electron/
├── LicenseControl-1.0.0.dmg              # macOS — imagen de disco
├── LicenseControl-1.0.0-arm64.dmg        # macOS Apple Silicon
├── LicenseControl Setup 1.0.0.exe        # Windows — instalador NSIS
├── LicenseControl-1.0.0.AppImage         # Linux
└── mac/  |  win-unpacked/                # Contenido sin empaquetar (debug)
```

### Íconos de la app

[](https://github.com/vperezguzman66/license-control#%C3%ADconos-de-la-app)

Sin íconos personalizados, electron-builder usa el ícono por defecto de Electron. Para usar tu propio ícono:

|Plataforma|Archivo|Ubicación|Tamaño mínimo|
|---|---|---|---|
|macOS|`icon.icns`|`build/icon.icns`|512×512 px|
|Windows|`icon.ico`|`build/icon.ico`|256×256 px|
|Linux|`icon.png`|`build/icon.png`|512×512 px|

Coloca los archivos en la carpeta `build/` en la raíz del proyecto (créala si no existe). electron-builder los detecta automáticamente.

**Herramientas para generar íconos:**

- **macOS:** `iconutil` (incluido en Xcode) a partir de un `.iconset`
- **Online:** [iconifier.net](https://iconifier.net/) — sube un PNG de 1024×1024 y descarga `.icns` e `.ico`

### Firma de código (recomendado para distribución)

[](https://github.com/vperezguzman66/license-control#firma-de-c%C3%B3digo-recomendado-para-distribuci%C3%B3n)

Sin firma digital, macOS muestra una advertencia de "desarrollador no identificado" y Windows activa SmartScreen.

**macOS — Code Signing + Notarización:**

```shell
# Requiere una cuenta de Apple Developer ($99 USD/año)
# electron-builder lee las variables de entorno:
export CSC_LINK="ruta/a/tu/certificado.p12"
export CSC_KEY_PASSWORD="contraseña_del_certificado"
export APPLE_ID="tu@email.com"
export APPLE_APP_SPECIFIC_PASSWORD="xxxx-xxxx-xxxx-xxxx"
export APPLE_TEAM_ID="XXXXXXXXXX"

cd licensecontrol && npm run dist:mac
```

**Windows — Code Signing:**

```shell
export CSC_LINK="ruta/a/tu/certificado.pfx"
export CSC_KEY_PASSWORD="contraseña"

cd licensecontrol && npm run dist:win
```

Si no cuentas con certificados, los usuarios pueden instalar de todas formas:

- **macOS:** clic derecho → Abrir → confirmar apertura
- **Windows:** clic en "Más información" → "Ejecutar de todas formas" en SmartScreen

### Datos de usuario en producción

[](https://github.com/vperezguzman66/license-control#datos-de-usuario-en-producci%C3%B3n)

La base de datos y la llave de cifrado **no van dentro del instalador**. Se crean al primer arranque en el directorio de usuario del sistema operativo:

|Archivo|macOS|Windows|
|---|---|---|
|Base de datos|`~/Library/Application Support/licensecontrol/licensecontrol.db`|`%APPDATA%\licensecontrol\licensecontrol.db`|
|Llave de cifrado|`~/Library/Application Support/licensecontrol/licensecontrol.key`|`%APPDATA%\licensecontrol\licensecontrol.key`|

> **Crítico:** la llave de cifrado (`licensecontrol.key`) es necesaria para descifrar las claves de licencia. Si se pierde sin backup, los license keys almacenados quedan irrecuperables. Siempre mantener copias de seguridad con la función de backup integrada (incluye la llave automáticamente).

### Primer arranque tras instalación

[](https://github.com/vperezguzman66/license-control#primer-arranque-tras-instalaci%C3%B3n)

1. La app crea la BD y la llave de cifrado si no existen
2. Si la tabla `users` está vacía, se crea usuario **`admin`** con contraseña aleatoria de bootstrap (archivo `BOOTSTRAP_ADMIN_CREDENTIALS.txt` en `userData`)
3. **Cambiar la contraseña del admin inmediatamente** en Administración → Usuarios y eliminar el archivo de bootstrap

---

## Estado de implementación (2026-06)

[](https://github.com/vperezguzman66/license-control#estado-de-implementaci%C3%B3n-2026-06)

### Ya implementado

[](https://github.com/vperezguzman66/license-control#ya-implementado)

Fases 1, 2 y 3 completas:

- **Compliance / SAM completo:** Entitlements CRUD, ELP con motor extraído (`elp-engine.cjs`), recálculo automático en eventos clave (asignaciones, entitlements, importación CSV), catálogo normalizado, consolidación profunda, fusión operativa, SAM Tasks, export de auditoría.
- **Optimización de costos:** ahorro estimado proporcional por sobredimensionamiento; lista de licencias con costo sin entitlement.
- **Calidad de datos:** wizard de 4 pasos, score 0–100, 6 tipos de señales (críticas/medias/bajas), detección de duplicados normalizados, licencias sin entitlement y entitlements vencidos.
- **Score de completitud:** cálculo por licencia (4–5 campos) y promedio global en Dashboard; mini-barra visual en tabla de licencias.
- **Score de madurez SAM (P8):** evalúa 4 áreas ponderadas (Inventario 25%, Compliance 35%, Renovaciones 20%, Calidad de datos 20%); niveles Inicial/En desarrollo/Definido/Optimizado; puntos de mejora expandibles por área; lógica pura en `lib/sam-maturity.js`.
- **Export SBOM (P7):** genera Software Bill of Materials en SPDX 2.3 JSON y CycloneDX 1.4 XML desde el catálogo de licencias; implementado en `handlers/sbom.cjs`.
- **Descubrimiento (P9):** importa CSV/JSON de inventario (OCS Inventory, GLPI, Lansweeper); auto-detección de columnas; reconciliación de software y usuarios contra licencias y asignaciones; acciones directas (crear licencia, asignar usuario); implementado en `handlers/discovery.cjs` + `lib/discovery.js` + página `Discovery.jsx`.
- **Workflow de renovaciones:** estados `pending/in_review/approved/cancelled`, responsable asignado, checklist de 4 ítems, alertas proactivas en página Alertas y notificación del SO.
- **Política de contraseñas:** mínimo 8 caracteres + mayúscula + número; `validatePassword` en backend y frontend con hint visible.
- **Suite de tests (actualizado 2026-07-02):** 146 tests en 16 suites con Jest 30 (raíz) + 13 tests de renderer con Vitest — 159 en total.
- **Hardening pre-producción:** TOCTOU en restore, AES authenticate-then-decrypt, path traversal en adjuntos, migraciones atómicas por transacción, bloqueo total de navegación en producción.
- **Modularización del renderer:** `Compliance.jsx` reducido de 1929 → 661 líneas; `Discovery.jsx` extraído en 6 componentes en `components/discovery/`; lógica pura en `lib/complianceHelpers.js`, `lib/sam-maturity.js` y `lib/discovery.js`.

### Tests

[](https://github.com/vperezguzman66/license-control#tests)

```shell
# Desde la raíz del proyecto
npx jest --no-coverage
# → 146 tests, 16 suites, ~7s
```

|Suite|Tests|Cobertura|
|---|---|---|
|`ipc-schemas.test.js`|29|Contratos IPC y validaciones de payload|
|`sam-normalize.test.js`|25|`normalizeProductKey`, `normalizeMetricType`, similitud Jaccard|
|`validators.test.js`|25|Normalizadores CSV + `validatePassword`|
|`attachment-security.test.js`|11|Validaciones de seguridad en adjuntos (incl. parser de cabecera ZIP real)|
|`crypto.test.js`|10|`encryptLicenseKey` / `decryptLicenseKey` round-trip y datos corruptos|
|`csv-security.test.js`|6|Reglas de seguridad para importación CSV|
|`restore-security.test.js`|6|Validaciones críticas del flujo de restore|
|`stepup-rate-limit.test.js`|6|Rate-limiting, audit log de fallos y validación de scope en `auth:stepUp`|
|`elp-utils.test.js`|5|`classifyElpStatus` (over/under/compliant, strings numéricos, decimales)|
|`attachments-sig-validated.test.js`|4|Columna `sig_validated` — adjuntos pre/post migración v17|
|`backup-crypto.test.js`|4|Cifrado y compatibilidad de backups|
|`ipc-safe.test.js`|4|Patch global de `ipcMain.handle` — errores internos no se filtran|
|`backup-cross-session.test.js`|3|Validación de sesión en `pendingRestoreData`|
|`key-store.test.js`|3|Fallback a path legacy si `safeStorage` no está disponible|
|`renewals-checklist.test.js`|3|Defaults de checklist vacío vs explícito|
|`rh06-stepup-backup-restore.test.js`|2|Integración de seguridad: `stepUp` requerido + backup/restore cifrado con contraseña|

### Hardening de seguridad y correcciones de integridad (2026-06)

[](https://github.com/vperezguzman66/license-control#hardening-de-seguridad-y-correcciones-de-integridad-2026-06)

Correcciones aplicadas en la revisión pre-producción, agrupadas por área.

**Seguridad IPC y autenticación**

- `auth:deleteUser` usa `ctx.sessionUserId` (proceso principal) para el guard de auto-eliminación — nunca el `currentUserId` enviado por el renderer. Un renderer malicioso no puede suplantar el ID de sesión para eliminar otro usuario.
- `auth:login` rechaza con `{ success: false }` si `username.length > 64` o `password.length > 256`, antes de tocar el Map de intentos o la BD. Previene agotamiento de memoria vía strings muy largos.
- `will-navigate` rechaza _cualquier_ URL en producción (`allowed = false`); solo en dev se permite `localhost:5173`. Antes, la condición errónea podría dejar pasar navegaciones no esperadas.
- `onThemeChange` / `onUpdaterStatus` (preload) almacenan la referencia exacta del listener en variables de módulo y usan `ipcRenderer.removeListener` en vez de `removeAllListeners`, que afectaría a otros suscriptores del mismo canal.

**Cifrado y manejo de claves**

- `loadOrCreateKey` verifica que el archivo leído tenga exactamente 32 bytes; un archivo corrupto o truncado lanza error descriptivo en lugar de continuar con una clave inválida.
- En `db:restoreWithPassword`, el plaintext se obtiene como `Buffer.concat([decipher.update(ct), decipher.final()])` antes de `JSON.parse`. `decipher.final()` es quien verifica el auth tag GCM; procesarlo después viola authenticate-then-decrypt.

**Backup / Restore — race condition**

- `db:restore` almacena los bytes del archivo en memoria en lugar de la ruta. `db:restoreWithPassword` consume esos bytes directamente sin releer desde disco. Antes, un archivo modificado entre los dos IPC calls podía bypassear la validación de 50 MB o inyectar contenido diferente al validado (TOCTOU).

**Adjuntos — path traversal**

- `attachments:delete` valida la ruta canónica con `path.resolve + startsWith(baseDir + path.sep)` antes de `unlinkSync`. Un `filename` con `../` en BD no puede salir del directorio de adjuntos.
- `attachments:upload` valida la extensión contra la lista permitida _antes_ de `statSync` y de copiar. El filtro del diálogo nativo es advisory; el OS no garantiza el tipo del archivo.

**Migraciones — atomicidad**

- Cada `m.up(db)` se ejecuta dentro de su propia transacción `BEGIN / COMMIT / ROLLBACK`. Un fallo parcial hace rollback completo y no avanza la versión en `schema_version`.

**ELP / SAM — correcciones de lógica**

- `elp:recalculate` agrupa licencias por `product_key` en un `Map` antes del loop. Ejecuta exactamente un `DELETE + INSERT` por par `(key, metricType)`, acumulando el `consumed` de todas las licencias del grupo. Antes, cada licencia sobreescribía la posición de la anterior con el mismo `product_key`.
- Para métricas no-seat (`named_user`, `device`, `concurrent`, `core`), el `consumed` se toma del último `consumption_snapshot`. Si no existe snapshot, la posición se omite en lugar de insertar `consumed = 0` y reportar falsamente `over_licensed`.
- `normalizeProductKey` en `Compliance.jsx` incluye `.trim()` inicial, igual que `lib/sam-normalize.cjs`. Sin ese trim, nombres con espacios generan claves distintas entre frontend y backend.

**Frontend — manejo de errores**

- Los ~10 handlers async de `Compliance.jsx` (recalculate, create entitlement, merge, exportAuditPack, etc.) tienen `try/catch/finally`. El `finally` resetea los flags de estado aunque el IPC falle, evitando spinners permanentes.
- La función `load()` está envuelta en `try/catch` con `setError()`. Antes, un error en la carga inicial dejaba la página en blanco.
- `handleMergeCatalogCandidate` muestra un `confirm()` antes de ejecutar la fusión, que es destructiva e irreversible.

### Actualizar la versión distribuida

[](https://github.com/vperezguzman66/license-control#actualizar-la-versi%C3%B3n-distribuida)

Para publicar una nueva versión:

1. Actualizar `"version"` en `package.json` (raíz)
2. Hacer el build: `cd licensecontrol && npm run dist:mac` (o la plataforma que corresponda)
3. Distribuir el nuevo `.dmg` / `.exe`
4. Los datos del usuario (BD + llave) se conservan intactos entre versiones

### Consideraciones para distribución masiva (múltiples equipos)

[](https://github.com/vperezguzman66/license-control#consideraciones-para-distribuci%C3%B3n-masiva-m%C3%BAltiples-equipos)

- Cada instalación tiene su propia BD y llave de cifrado independiente
- Para compartir datos entre equipos **no** se copian archivos — se usa la función Backup/Restore
- El backup es autocontenido: incluye tanto la BD como la llave de cifrado
- Un backup generado en equipo A puede restaurarse en equipo B y las claves de licencia seguirán siendo legibles

---

## Estructura del proyecto

[](https://github.com/vperezguzman66/license-control#estructura-del-proyecto)

```
Licencias/
├── electron.cjs          # Entry point delgado: ventana, lifecycle, registro de handlers
├── preload.cjs           # contextBridge — expone window.electronAPI al renderer
├── package.json          # Dependencias raíz + configuración de electron-builder
├── handlers/             # Handlers IPC por dominio (proceso principal)
│   ├── context.cjs       # Estado mutable compartido (db, session, keys)
│   ├── helpers.cjs       # scalar, rowsOf, persist, logAudit, hasRole, validatePassword…
│   ├── constants.cjs     # VALID_ROLES, VALID_ENTITLEMENT_METRICS, MIN_PASSWORD_LENGTH…
│   ├── migrations.cjs    # MIGRATIONS v1–v16
│   ├── db-init.cjs       # initDatabase, loadOrCreateKey
│   ├── elp-engine.cjs    # recalculateElpForKey() — motor ELP reutilizable sin dep Electron
│   ├── licenses.cjs      # db:getLicenses/create/update/delete/importCSV
│   ├── auth.cjs          # auth:login/logout/getUsers…
│   ├── compliance.cjs    # entitlements:*, elp:*, tasks:*
│   ├── products.cjs      # products:list/syncCatalog/consolidation/merge
│   ├── renewals.cjs      # renewals:getByLicense/create/update/delete/getAll/getUpcoming
│   ├── audit.cjs         # audit:getLog/exportPack
│   ├── sbom.cjs          # sbom:export — SPDX 2.3 JSON y CycloneDX 1.4 XML
│   ├── discovery.cjs     # discovery:importFile — parseo CSV/JSON para reconciliación
│   └── …                 # vendors, assignments, attachments, contracts, backup, system
├── lib/
│   ├── validators.cjs    # normalizeText, normalizeDate, normalizeCurrency… (puro, testeable)
│   ├── elp-utils.cjs     # classifyElpStatus — clasificación ELP sin dep de Electron
│   ├── sam-normalize.cjs # normalizeProductKey, normalizeMetricType, catalogPairKey…
│   └── crypto-utils.cjs  # encryptLicenseKey, decryptLicenseKey
├── tests/
│   └── unit/
│       ├── crypto.test.js      # 28 tests — cifrado AES-256-GCM round-trip y validaciones
│       ├── sam-normalize.test.js # 18 tests — normalización de claves y similitud
│       ├── elp-utils.test.js   # 5 tests — classifyElpStatus
│       └── validators.test.js  # 14 tests — normalizadores CSV + validatePassword
├── build/                # Íconos para el instalador (icon.icns, icon.ico, icon.png)
└── licensecontrol/       # App React (renderer)
    ├── src/
    │   ├── App.jsx
    │   ├── components/
    │   │   ├── Layout.jsx            # Shell: sidebar + badge alertas + backup/restore
    │   │   ├── LicenseModal.jsx      # Modal de alta/edición de licencia
    │   │   ├── AssignmentsModal.jsx  # Modal de gestión de asignaciones
    │   │   ├── RenewalsModal.jsx     # Modal de renovaciones con workflow, checklist y responsable
    │   │   ├── compliance/           # Componentes extraídos de Compliance.jsx
    │   │   │   ├── SummaryCard.jsx         # Tarjeta KPI reutilizable
    │   │   │   ├── CostOptimizationPanel.jsx
    │   │   │   ├── CatalogPanel.jsx        # Catálogo + consolidación profunda
    │   │   │   ├── DataQualityPanel.jsx    # Modo de trabajo + wizard de saneamiento
    │   │   │   ├── RiskTasksPanel.jsx      # Top 5 riesgos + tareas SAM
    │   │   │   ├── ElpPositionsPanel.jsx   # Tabla ELP + drift side-by-side
    │   │   │   ├── EntitlementPanel.jsx    # Filtro/tabla/formulario de entitlements
    │   │   │   └── SamMaturityPanel.jsx    # Score de madurez SAM por dominio + nivel + mejoras
    │   │   └── discovery/            # Componentes extraídos de Discovery.jsx
    │   │       ├── ColMapper.jsx           # Selectores de mapeo de columnas
    │   │       ├── SummaryBar.jsx          # Barra de filtros por estado de reconciliación
    │   │       ├── StatusBadge.jsx         # Badge de estado (sin_licencia, cubierto, etc.)
    │   │       ├── SoftwareTable.jsx       # Tabla de resultados de software
    │   │       ├── UsersTable.jsx          # Tabla de resultados de usuarios
    │   │       └── AssignModal.jsx         # Modal para asignar usuario a licencia
    │   ├── pages/
    │   │   ├── Login.jsx             # Pantalla de inicio de sesión (con rate limiting)
    │   │   ├── Dashboard.jsx         # Stats + score de completitud global
    │   │   ├── Licenses.jsx          # Tabla con filtros, exportación, CSV, mini-barra completitud
    │   │   ├── Alerts.jsx            # Licencias vencidas, por vencer y renovaciones pendientes
    │   │   ├── Compliance.jsx        # Orquestador ELP: estado, computed, handlers (~660 líneas)
    │   │   ├── Discovery.jsx         # Flujo 3 pasos: importar → mapear → reconciliar
    │   │   ├── Vendors.jsx           # CRUD de proveedores con RUT y teléfono formateados
    │   │   ├── Users.jsx             # Gestión de usuarios y roles (solo admin)
    │   │   └── AuditLog.jsx          # Log de auditoría paginado (solo admin)
    │   └── lib/
    │       ├── db.js                 # Wrapper de window.electronAPI
    │       ├── export.js             # Exportación CSV (compatible con Excel) y PDF
    │       ├── completeness.js       # computeLicenseCompleteness / computeGlobalCompleteness
    │       ├── sam-maturity.js       # computeSamMaturity — scoring puro por 4 dominios
    │       ├── discovery.js          # reconcileSoftware, reconcileUsers, autoDetectColMap
    │       └── complianceHelpers.js  # Funciones puras SAM: formatMoney, priorityBadge,
    │                                 # buildDataQualityWizard, emptyForm, downloadCsv…
    ├── index.html
    ├── vite.config.js                # base: './' para compatibilidad con file://
    └── tailwind.config.js
```

## Arquitectura

[](https://github.com/vperezguzman66/license-control#arquitectura)

```
Renderer (React)
    │  window.electronAPI.*
    ▼
preload.cjs  (contextBridge)
    │  ipcRenderer.invoke()
    ▼
electron.cjs  (entry point)
    │  require('./handlers/*.cjs')
    ▼
handlers/  (proceso principal — IPC handlers en 14 módulos)
    │  sql.js · roles · AES-256-GCM · audit_log · transacciones
    ▼
licensecontrol.db  (userData)
licensecontrol.key (userData — clave AES-256, permisos 0600)
```

El proceso principal valida el rol del usuario en cada handler IPC. El renderer nunca toca Node.js directamente.

## Seguridad — detalle técnico

[](https://github.com/vperezguzman66/license-control#seguridad--detalle-t%C3%A9cnico)

|Mecanismo|Detalle|
|---|---|
|Contraseñas|`scrypt(password, salt, 64)` — salt de 16 bytes aleatorios; comparación con `timingSafeEqual`|
|License keys|AES-256-GCM con IV aleatorio por registro; formato `ENC:iv:ct:tag`; clave en archivo `0600`|
|Rate limiting|Map en memoria por username; bloqueo de 30s tras 5 intentos fallidos|
|IPC roles|`hasRole()` en cada handler: verifica `sessionRole` antes de ejecutar|
|SQL|100% parametrizado; sin interpolación de strings|
|CSP|Producción sin `unsafe-inline`; aplicada via `session.webRequest.onHeadersReceived` con `object-src 'none'`, `base-uri 'none'`, `frame-ancestors 'none'` y `form-action 'self'`|
|Sandbox|`nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`|
|Backup|Incluye `.key` junto al `.db`; restore valida tablas requeridas y tamaño máximo (50MB)|

## Base de datos

[](https://github.com/vperezguzman66/license-control#base-de-datos)

La BD se guarda en `{userData}/licensecontrol.db`.

- **macOS:** `~/Library/Application Support/licensecontrol/licensecontrol.db`
- **Windows:** `%APPDATA%\licensecontrol\licensecontrol.db`

### Tabla `licenses`

[](https://github.com/vperezguzman66/license-control#tabla-licenses)

|Campo|Tipo|Descripción|
|---|---|---|
|`id`|INTEGER PK|Autoincremental|
|`name`|TEXT|Nombre del software|
|`vendor_id`|INTEGER FK|Referencia a `vendors.id`|
|`license_key`|TEXT|Clave cifrada AES-256-GCM (`ENC:iv:ct:tag`)|
|`type`|TEXT|`subscription` / `perpetual` / `trial`|
|`seats`|INTEGER|Número de licencias disponibles|
|`assigned_to`|TEXT|Departamento o área (campo libre)|
|`purchase_date`|TEXT|Fecha de compra `YYYY-MM-DD`|
|`expiry_date`|TEXT|Fecha de vencimiento `YYYY-MM-DD` (null = perpetua)|
|`cost`|REAL|Costo de la licencia|
|`currency`|TEXT|Código de moneda (default `CLP`)|
|`status`|TEXT|`active` / `expired` / `cancelled`|
|`notes`|TEXT|Notas adicionales|
|`created_at`|TEXT|Timestamp de creación (UTC)|

### Tabla `assignments`

[](https://github.com/vperezguzman66/license-control#tabla-assignments)

|Campo|Tipo|Descripción|
|---|---|---|
|`id`|INTEGER PK|Autoincremental|
|`license_id`|INTEGER FK|Referencia a `licenses.id` (ON DELETE CASCADE)|
|`asignado_a`|TEXT|Usuario, equipo o responsable|
|`fecha_asignacion`|TEXT|Fecha de asignación `YYYY-MM-DD`|
|`notas`|TEXT|Notas de asignación (opcional)|
|`created_at`|TEXT|Timestamp de creación (UTC)|

### Tabla `users`

[](https://github.com/vperezguzman66/license-control#tabla-users)

|Campo|Tipo|Descripción|
|---|---|---|
|`id`|INTEGER PK|Autoincremental|
|`username`|TEXT UNIQUE|Nombre de usuario|
|`password_hash`|TEXT|Hash scrypt (hex, 64 bytes)|
|`salt`|TEXT|Salt aleatorio (16 bytes hex)|
|`role`|TEXT|`admin` / `editor` / `viewer`|
|`created_at`|TEXT|Timestamp de creación (UTC)|

### Tabla `vendors`

[](https://github.com/vperezguzman66/license-control#tabla-vendors)

|Campo|Tipo|Descripción|
|---|---|---|
|`id`|INTEGER PK|Autoincremental|
|`nombre`|TEXT|Nombre del proveedor|
|`rut`|TEXT|RUT chileno `XX.XXX.XXX-K` (opcional)|
|`contacto`|TEXT|Nombre del contacto (opcional)|
|`telefono`|TEXT|Teléfono `9 XXXX XXXX` (opcional)|
|`email`|TEXT|Correo electrónico (opcional)|
|`sitio_web`|TEXT|URL del sitio web (opcional)|
|`notas`|TEXT|Notas adicionales (opcional)|
|`created_at`|TEXT|Timestamp de creación (UTC)|

### Tabla `audit_log`

[](https://github.com/vperezguzman66/license-control#tabla-audit_log)

|Campo|Tipo|Descripción|
|---|---|---|
|`id`|INTEGER PK|Autoincremental|
|`user_id`|INTEGER|ID del usuario que realizó la acción|
|`username`|TEXT|Nombre de usuario (snapshot al momento de la acción)|
|`action`|TEXT|`login` / `logout` / `create` / `update` / `delete` / `import` / `backup` / `restore`|
|`entity`|TEXT|`license` / `user` / `vendor` / `assignment` / `session` / `database`|
|`entity_id`|INTEGER|ID del registro afectado (null cuando no aplica)|
|`details`|TEXT|Descripción legible (nombre del software, usuario, etc.)|
|`created_at`|TEXT|Timestamp UTC de la acción|

## Importar CSV

[](https://github.com/vperezguzman66/license-control#importar-csv)

El CSV debe tener cabeceras en la primera fila. La columna `name` es obligatoria; el resto es opcional.

|Columna|Descripción|
|---|---|
|`name`|Nombre del software (**requerido**)|
|`license_key`|Clave o serial (se cifra automáticamente al importar)|
|`type`|`subscription` / `perpetual` / `trial`|
|`seats`|Número de licencias disponibles (default: 1)|
|`assigned_to`|Departamento o área|
|`expiry_date`|Fecha en formato `YYYY-MM-DD`|
|`cost`|Costo numérico|
|`currency`|Código de moneda (default: `CLP`)|
|`status`|`active` / `expired` / `cancelled`|
|`notes`|Notas adicionales|
|`purchase_date`|Fecha de compra `YYYY-MM-DD`|

Límites: máximo **2MB** y **10.000 filas** de datos.

---
title: LicenseControl - Mapa Visual Operativo
tags: [licensecontrol, roadmap, arquitectura, sam, release]
project: LicenseControl
updated: 2026-06-26
---

# LicenseControl — Mapa Visual Operativo

## Estado general

- RH-01 ✅
- RH-02 ✅
- RH-03 ✅
- RH-04 ✅
- RH-05 ✅
- RH-06 ✅
- RH-07 ✅
- RH-08 ✅

---

## Roadmap visual (RH-01 → RH-08)

```mermaid
flowchart LR
  RH01[RH-01<br/>Electron patch] --> RH02[RH-02<br/>Fix dompurify]
  RH02 --> RH03[RH-03<br/>Alineación docs]
  RH03 --> RH04[RH-04<br/>Code splitting]
  RH04 --> RH05[RH-05<br/>Smoke empaquetado]
  RH05 --> RH06[RH-06<br/>Step-up + Backup/Restore]
  RH06 --> RH07[RH-07<br/>Bundle budget gate]
  RH07 --> RH08[RH-08<br/>Plantilla release evidence]

  classDef done fill:#163,stroke:#3f6,color:#dff;
  class RH01,RH02,RH03,RH04,RH05,RH06,RH07,RH08 done;
  
  
  
  