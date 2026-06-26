==Revisar!==

[[Varios]]

[](https://github.com/vperezguzman66/ControlLicencias#control-de-licencias-para-pymes)

Aplicación MVP para centralizar la gestión de licencias tecnológicas en pequeñas y medianas empresas (pymes): inventario, vencimientos, uso de licencias y visibilidad operativa.

## Objetivo del proyecto

[](https://github.com/vperezguzman66/ControlLicencias#objetivo-del-proyecto)

Reducir riesgos de incumplimiento y compras innecesarias con una solución ligera que permita:

- registrar licencias por software y proveedor,
- controlar licencias compradas vs usadas,
- detectar licencias vencidas o próximas a vencer,
- contar con trazabilidad mínima para operación y auditoría.

## Funcionalidades actuales

[](https://github.com/vperezguzman66/ControlLicencias#funcionalidades-actuales)

- Alta, consulta, actualización y eliminación de licencias (API REST + UI web).
- Catálogo de áreas responsables con código y descripción (alta + mantenimiento) para evitar texto libre en asignación de área.
- Login único con sesión compartida para todos los módulos web.
- Navegación multipágina: Inicio, Licencias, Parámetros e Informes (placeholder).
- Normalización automática de parámetros de áreas a MAYÚSCULAS (`code` y `description`).
- Dashboard resumido con métricas clave.
- Búsqueda por software, proveedor y área responsable.
- Validación con esquema JSON (`draft 2020-12`) usando Ajv.
- Reglas de negocio cruzadas (ej. licencias usadas no mayores a compradas).
- Mapeo gradual de campos:
    - **legacy** (`softwareName`, `endDate`, `owner`, `cost`, `seats*`, etc.)
    - **canónico** (`software_name`, `contract_end_date`, `owner_area`, `cost_amount`, `licenses_*`, etc.)
- Compatibilidad de respuesta para frontend actual:
    - `status`: estado UI en español (`activa`, `por_vencer`, `vencida`)
    - `status_code`: estado canónico (`active`, `expiring_soon`, `expired`, `suspended`)

## Stack tecnológico

[](https://github.com/vperezguzman66/ControlLicencias#stack-tecnol%C3%B3gico)

- **Runtime:** Node.js
- **Backend:** Express + CORS + Morgan + Dotenv
- **Validación:** Ajv2020 + ajv-formats
- **Frontend:** HTML + CSS + JavaScript (vanilla)
- **Persistencia:** archivo JSON local (sin base de datos externa)

## Arquitectura

[](https://github.com/vperezguzman66/ControlLicencias#arquitectura)

- `src/server.js`: servidor HTTP, endpoints API y respuestas de validación.
- `src/db.js`: fachada de persistencia por proveedor (`sqlite` o `json`).
- `src/license-repository-sqlite.js`: repositorio relacional SQLite (MVP v0.4 semana 3).
- `src/license-repository-json.js`: fallback de compatibilidad para almacenamiento JSON.
- `src/license-model.js`: normalización/mapeo legacy-canónico + reglas de negocio.
- `src/license-validation.js`: validación de payload con `license.schema.json`.
- `docs/schemas/license.schema.json`: contrato canónico del dominio.
- `public/`: interfaz web estática servida por Express.
- `data/licenses.json`: almacenamiento local de datos (autogenerado si no existe).

## Estructura del proyecto

[](https://github.com/vperezguzman66/ControlLicencias#estructura-del-proyecto)

```
ControlLicencias/
├─ .env
├─ .env.example
├─ package.json
├─ src/
│  ├─ server.js
│  ├─ db.js
│  ├─ license-model.js
│  └─ license-validation.js
├─ docs/
│  └─ schemas/
│     ├─ license.schema.json
│     ├─ license_schema.sql
│     └─ README.md
├─ public/
│  ├─ index.html
│  ├─ login.html
│  ├─ licenses.html
│  ├─ parameters.html
│  ├─ reports.html
│  ├─ styles.css
│  ├─ auth-guard.js
│  ├─ home-app.js
│  ├─ login-app.js
│  ├─ licenses-app.js
│  ├─ parameters-app.js
│  └─ parameter-normalization.js
└─ data/
   └─ licenses.json
```

## Requisitos

[](https://github.com/vperezguzman66/ControlLicencias#requisitos)

- Node.js 18+ (recomendado LTS).

## Configuración de entorno

[](https://github.com/vperezguzman66/ControlLicencias#configuraci%C3%B3n-de-entorno)

Archivo `.env`:

```dotenv
PORT=3000
DB_PROVIDER=sqlite
DB_FILE=data/licenses.db
LEGACY_JSON_FILE=data/licenses.json
```

Variables:

- `PORT`: puerto del servidor Express.
- `DB_PROVIDER`: proveedor de persistencia (`sqlite` recomendado, `json` para compatibilidad).
- `DB_FILE`: ruta del archivo de persistencia. Para SQLite usar por ejemplo `data/licenses.db`.
- `LEGACY_JSON_FILE`: ruta del JSON legacy a migrar al inicializar SQLite (solo si la base está vacía).
- `JWT_SECRET`: secreto para firmar tokens JWT.
- `JWT_EXPIRES_IN`: tiempo de vida del token (ej. `8h`).
- `CORS_ORIGIN`: origen(s) permitidos para CORS (separados por coma).
- `AUTH_USERS_JSON`: usuarios demo en formato JSON para login local.
- `BACKUP_DIR`: directorio donde se guardan respaldos SQLite (por defecto `backups`).
- `BACKUP_KEEP_LAST`: cantidad mínima de backups recientes a conservar (por defecto `30`).
- `BACKUP_MAX_AGE_DAYS`: antigüedad máxima en días para conservar backups (por defecto `30`).
- `BACKUP_STALENESS_HOURS`: umbral de horas para marcar último backup como potencialmente desactualizado (por defecto `24`).
- `NOTIFY_WINDOW_DAYS`: ventana de días para evaluar alertas de renovación (por defecto `30`).
- `NOTIFY_MIN_SEVERITY`: severidad mínima a notificar (`medium`, `high`, `critical`; por defecto `medium`).
- `NOTIFY_CHANNEL`: canal de entrega (`console` o `webhook`; por defecto `console`).
- `NOTIFY_WEBHOOK_URL`: URL para envío cuando el canal configurado es `webhook`.

Ejemplo de usuarios demo (`AUTH_USERS_JSON`):

```json
[
  { "username": "admin", "password": "admin123", "role": "admin" },
  { "username": "viewer", "password": "viewer123", "role": "viewer" }
]
```

## Ejecución local

[](https://github.com/vperezguzman66/ControlLicencias#ejecuci%C3%B3n-local)

1. Instalar dependencias: `npm install`
2. Iniciar en desarrollo: `npm run dev`
3. Iniciar en modo normal: `npm start`
4. Abrir en navegador: `http://localhost:3000`

Navegación de pantallas:

- Login único: `http://localhost:3000/login.html`
- Inicio: `http://localhost:3000/index.html`
- Licencias: `http://localhost:3000/licenses.html`
- Parámetros: `http://localhost:3000/parameters.html`
- Informes (placeholder): `http://localhost:3000/reports.html`

> Flujo recomendado: ingresar por `login.html` y desde ahí navegar por módulos. Si una página requiere sesión y no hay token válido, redirige automáticamente al login.

## Scripts

[](https://github.com/vperezguzman66/ControlLicencias#scripts)

- `npm run dev`: ejecuta `nodemon src/server.js`.
- `npm start`: ejecuta `node src/server.js`.
- `npm run backup:sqlite`: crea respaldo one-shot de la base SQLite actual.
- `npm run status:sqlite-backups`: muestra métricas operativas y riesgos del estado de backups.
- `npm run prune:sqlite-backups`: aplica política de retención sobre carpeta de backups.
- `npm run migrate:json-to-sqlite`: migra datos de JSON legacy a SQLite (one-shot).
- `npm run rollback:sqlite`: restaura la base SQLite desde backup (último disponible o el indicado).
- `npm run verify:sqlite-integrity`: verifica integridad de la base SQLite contra el manifest del backup.
- `npm run notify:renewals`: ejecuta job de notificaciones de renovación.

Migración one-shot (opcional):

- `npm run migrate:json-to-sqlite`
- `npm run migrate:json-to-sqlite -- --force`
- `npm run migrate:json-to-sqlite -- --source=data/licenses.json --target=data/licenses.db --force`

Semana 3.2 - operación segura (backup + rollback):

- Backup manual:
    - `npm run backup:sqlite`
    - `npm run backup:sqlite -- --target=data/licenses.db --backup-dir=backups`
- Migración con backup automático (por defecto):
    - `npm run migrate:json-to-sqlite -- --target=data/licenses.db`
    - `npm run migrate:json-to-sqlite -- --skip-backup --force`
- Rollback:
    - `npm run rollback:sqlite`
    - `npm run rollback:sqlite -- --backup=backups/licenses.db.20260528-183000.bak`

Semana 3.3 - verificación de integridad post-rollback:

- El backup genera un manifest (`<backup>.manifest.json`) con fingerprint (`row_count`, `max_updated_at`, `sha256`).
- El rollback valida automáticamente contra ese manifest.
- Comando de verificación manual:
    - `npm run verify:sqlite-integrity`
    - `npm run verify:sqlite-integrity -- --backup=backups/licenses.db.20260528-183000.bak`
- Flags útiles en rollback:
    - `--skip-verify` omite validación de integridad
    - `--allow-mismatch` no falla el comando ante mismatch

Semana 3.4 - retención automática de backups:

- Política aplicada: conservar por **cantidad** (`BACKUP_KEEP_LAST`) y por **antigüedad** (`BACKUP_MAX_AGE_DAYS`).
- Integración automática:
    - `backup:sqlite` aplica retención al finalizar.
    - `migrate:json-to-sqlite` aplica retención tras crear backup (si no usas `--skip-backup`).
- Comando manual de limpieza:
    - `npm run prune:sqlite-backups`
    - `npm run prune:sqlite-backups -- --dry-run`
    - `npm run prune:sqlite-backups -- --retention-keep=20 --retention-days=15`
- Flags útiles:
    - `--skip-retention` en `backup:sqlite` y `migrate:json-to-sqlite`

Semana 3.5 - observabilidad operativa de backups:

- `GET /api/health` incluye bloque `backup_observability` cuando el proveedor activo es SQLite.
- Métricas incluidas:
    - total de backups
    - último backup (ruta, antigüedad, manifest)
    - presión de retención (overflow/aging)
    - señales de riesgo (`no_backups`, `latest_stale`, etc.)
- CLI de observabilidad:
    - `npm run status:sqlite-backups`
    - `npm run status:sqlite-backups -- --json`
    - `npm run status:sqlite-backups -- --staleness-hours=12`

Semana 4.0 - alertas operativas de vencimiento:

- Nuevo endpoint: `GET /api/alerts/renewals?days=30`
- Objetivo: priorizar licencias vencidas y próximas a vencer por severidad.
- Incluye:
    - `summary` por niveles (`critical`, `high`, `medium`, `low`)
    - `items` con `days_to_expiry`, `severity`, `utilization_pct` y metadatos clave.
- Acceso: roles `admin` y `viewer`.

Semana 4.1 - visualización de alertas en frontend:

- Sección "Alertas de renovación" en `public/index.html`.
- Filtros operativos en UI:
    - ventana (`7`, `30`, `60`, `365` días)
    - severidad (`critical`, `high`, `medium`, `low`)
- Resumen visual por severidad + tabla con:
    - software/proveedor
    - días al vencimiento
    - severidad y estado
    - uso (`licenses_used/licenses_purchased`) y porcentaje.
- Comportamiento de sesión:
    - sin token: se ocultan datos y se muestra mensaje de acceso
    - `viewer`: consulta habilitada
    - `admin`: consulta + edición completa.

Semana 4.2 - notificaciones operativas automáticas:

- Nuevo job CLI de notificaciones:
    - `npm run notify:renewals`
    - `npm run notify:renewals -- --dry-run`
    - `npm run notify:renewals -- --window-days=60 --min-severity=high --json`
- Capacidades:
    - deduplicación diaria por licencia/severidad/estado
    - modo `dry-run` sin entrega ni escritura de logs
    - canal `console` (default) y `webhook` (opcional)
- Persistencia operativa:
    - tabla `notifications_log` en SQLite para trazabilidad de envíos/fallos.

Semana 4.3 - catálogo de áreas responsables:

- Se incorpora catálogo de áreas gestionable por la empresa:
    - `GET /api/areas` (consulta)
    - `POST /api/areas` (alta, solo `admin`)
    - `PUT /api/areas/:id` (edición, solo `admin`)
    - `DELETE /api/areas/:id` (eliminación, solo `admin`)
- `owner_area` en licencias deja de ser texto libre en UI y pasa a selección controlada.
- El backend valida que el código de área exista antes de `POST/PUT /api/licenses`.
- `code` y `description` se normalizan automáticamente a MAYÚSCULAS en frontend y backend.

## Modelo de datos canónico (resumen)

[](https://github.com/vperezguzman66/ControlLicencias#modelo-de-datos-can%C3%B3nico-resumen)

### Campos imprescindibles (MVP real)

[](https://github.com/vperezguzman66/ControlLicencias#campos-imprescindibles-mvp-real)

Para operar el control de licencias en una pyme con un mínimo viable real, se exigen estos campos:

- `software_name`
- `vendor`
- `license_type`
- `licenses_purchased`
- `licenses_used`
- `contract_end_date`
- `renewal_type`
- `owner_area`

El resto de campos permanece disponible como opcional para enriquecer trazabilidad, costos y compliance.

Campos operativos clave:

- `software_name`, `vendor`
- `license_type`, `plan_or_edition`
- `licenses_purchased`, `licenses_used`
- `contract_start_date`, `contract_end_date`
- `renewal_type`, `status`
- `owner_area`
- `cost_amount`, `currency`, `billing_period`
- `supplier_contract_id`

Campos adicionales de trazabilidad/compliance:

- `proof_document`, `license_terms_url`
- `compliance_risk`, `notice_days_before_renewal`, `grace_period_days`
- `notes`, `updated_by`

> Nota: el almacenamiento también conserva aliases legacy (`cost`, `start_date`, `end_date`, `owner`) para compatibilidad gradual con la UI MVP.

## API REST

[](https://github.com/vperezguzman66/ControlLicencias#api-rest)

Base URL: `http://localhost:3000`

### Salud

[](https://github.com/vperezguzman66/ControlLicencias#salud)

- `GET /api/health`

Respuesta incluye metadatos de persistencia activa:

```json
{
  "ok": true,
  "service": "control-licencias-pyme",
  "persistence": {
    "provider": "sqlite",
    "db_file": "data/licenses.db",
    "active_provider": "sqlite"
  }
}
```

### Autenticación

[](https://github.com/vperezguzman66/ControlLicencias#autenticaci%C3%B3n)

- `POST /api/auth/login`

Body:

```json
{
  "username": "admin",
  "password": "admin123"
}
```

Respuesta:

```json
{
  "token": "<jwt>",
  "token_type": "Bearer",
  "role": "admin",
  "username": "admin"
}
```

> A partir de la v0.4 semana 2, los endpoints de negocio requieren `Authorization: Bearer <token>`.

- `viewer`: solo lectura (`GET`).
- `admin`: lectura + escritura (`GET/POST/PUT/DELETE`).

### Dashboard

[](https://github.com/vperezguzman66/ControlLicencias#dashboard)

- `GET /api/dashboard`

### Alertas de renovación

[](https://github.com/vperezguzman66/ControlLicencias#alertas-de-renovaci%C3%B3n)

- `GET /api/alerts/renewals`
- Query opcional: `days` (entero > 0, default `30`).

Ejemplo de respuesta:

```json
{
  "generated_at": "2026-05-28T21:00:00.000Z",
  "window_days": 30,
  "summary": {
    "total": 2,
    "critical": 1,
    "high": 0,
    "medium": 1,
    "low": 0
  },
  "items": [
    {
      "id": 12,
      "software_name": "Adobe Creative Cloud",
      "severity": "critical",
      "status": "vencida",
      "days_to_expiry": -5
    }
  ]
}
```

### Catálogo de áreas responsables

[](https://github.com/vperezguzman66/ControlLicencias#cat%C3%A1logo-de-%C3%A1reas-responsables)

- `GET /api/areas`
- `POST /api/areas`
- `PUT /api/areas/:id`
- `DELETE /api/areas/:id`

Body para creación:

```json
{
  "code": "FIN",
  "description": "FINANZAS"
}
```

Respuesta de consulta:

```json
{
  "total": 2,
  "items": [
    {
      "id": 1,
      "code": "TI",
      "description": "TECNOLOGÍA DE LA INFORMACIÓN",
      "created_at": "2026-05-28T22:00:00.000Z",
      "updated_at": "2026-05-28T22:00:00.000Z"
    }
  ]
}
```

### Historial de notificaciones

[](https://github.com/vperezguzman66/ControlLicencias#historial-de-notificaciones)

- `GET /api/notifications/logs`
- Query opcional: `limit` (1..200, default `50`).
- Acceso: roles `admin` y `viewer`.

Ejemplo de respuesta:

```json
{
  "provider": "sqlite",
  "limit": 50,
  "total": 1,
  "items": [
    {
      "id": 1,
      "license_id": 12,
      "severity": "high",
      "delivery_status": "sent",
      "channel": "console",
      "created_at": "2026-05-28T22:00:00.000Z"
    }
  ]
}
```

### Listar licencias

[](https://github.com/vperezguzman66/ControlLicencias#listar-licencias)

- `GET /api/licenses`
- `GET /api/licenses?search=adobe`

### Obtener una licencia

[](https://github.com/vperezguzman66/ControlLicencias#obtener-una-licencia)

- `GET /api/licenses/:id`

### Crear licencia

[](https://github.com/vperezguzman66/ControlLicencias#crear-licencia)

- `POST /api/licenses`

Ejemplo (payload legacy compatible):

```json
{
  "softwareName": "Adobe Creative Cloud",
  "vendor": "Adobe",
  "licensesPurchased": 10,
  "licensesUsed": 7,
  "cost": 500,
  "currency": "USD",
  "startDate": "2026-01-10",
  "endDate": "2026-12-31",
  "owner": "Diseño",
  "notes": "Licencias de equipo creativo"
}
```

Ejemplo (payload canónico):

```json
{
  "software_name": "Notion",
  "vendor": "Notion Labs",
  "license_type": "subscription",
  "plan_or_edition": "Business",
  "licenses_purchased": 20,
  "licenses_used": 10,
  "contract_start_date": "2026-06-01",
  "contract_end_date": "2027-05-31",
  "renewal_type": "auto",
  "owner_area": "TI",
  "cost_amount": 1200,
  "currency": "USD",
  "billing_period": "annual",
  "supplier_contract_id": "NOTION-2026-01"
}
```

### Actualizar licencia

[](https://github.com/vperezguzman66/ControlLicencias#actualizar-licencia)

- `PUT /api/licenses/:id`

### Eliminar licencia

[](https://github.com/vperezguzman66/ControlLicencias#eliminar-licencia)

- `DELETE /api/licenses/:id`

Respuesta exitosa: `204 No Content`.

## Formato de errores de validación

[](https://github.com/vperezguzman66/ControlLicencias#formato-de-errores-de-validaci%C3%B3n)

Cuando un payload no cumple el esquema o reglas de negocio, la API responde `400` con esta estructura:

```json
{
  "message": "No se pudo guardar la licencia porque hay campos inválidos.",
  "error_count": 2,
  "details": [
    {
      "source": "schema",
      "code": "pattern",
      "field": "currency",
      "message": "Moneda no cumple el formato esperado. Use código ISO de 3 letras, por ejemplo USD o MXN."
    },
    {
      "source": "business",
      "code": "licenses_used_exceeds_purchased",
      "field": "licenses_used",
      "message": "Las licencias usadas no pueden superar las licencias compradas. Ajusta el consumo o incrementa la compra de licencias."
    }
  ]
}
```

## Breaking changes y deprecaciones (integradores API)

[](https://github.com/vperezguzman66/ControlLicencias#breaking-changes-y-deprecaciones-integradores-api)

- **Cambio canónico:** `seats_purchased` y `seats_used` fueron reemplazados por `licenses_purchased` y `licenses_used`.
- **Payload recomendado:** usar `licensesPurchased`/`licensesUsed` (camelCase) o `licenses_purchased`/`licenses_used` (snake_case).
- **Compatibilidad temporal:** el backend aún acepta `seatsPurchased`, `seatsUsed`, `seats_purchased` y `seats_used` por retrocompatibilidad.
- **Plan recomendado de migración:**
    1. Actualizar clientes para enviar `licenses*`.
    2. Validar respuestas usando `licenses_*`.
    3. Retirar uso de `seats*` en integraciones internas.

## Interfaz web

[](https://github.com/vperezguzman66/ControlLicencias#interfaz-web)

La aplicación incluye una interfaz simple en `public/index.html` para:

- registrar nuevas licencias,
- consultar resumen del estado,
- buscar en tiempo real,
- eliminar registros existentes.

## Limitaciones actuales del MVP

[](https://github.com/vperezguzman66/ControlLicencias#limitaciones-actuales-del-mvp)

- Sin autenticación/autorización.
- Persistencia local JSON (no orientado a alta concurrencia).
- Sin historial completo de auditoría.
- Sin notificaciones automáticas de vencimiento.
- Sin importación/exportación CSV/Excel.

## Roadmap sugerido

[](https://github.com/vperezguzman66/ControlLicencias#roadmap-sugerido)

1. Autenticación y roles (admin, compras, auditor).
2. Multiempresa (tenant por pyme).
3. Alertas de vencimiento por correo/Teams.
4. Carga masiva de datos (CSV/Excel).
5. Evidencias adjuntas (facturas, contratos, órdenes de compra).
6. Reportes de costo y proyección de renovaciones.
7. Migración a base de datos relacional (PostgreSQL/MySQL) para producción.

## Release notes (breve)

[](https://github.com/vperezguzman66/ControlLicencias#release-notes-breve)

### v0.2.0 - 2026-05-27

[](https://github.com/vperezguzman66/ControlLicencias#v020---2026-05-27)

- Integración de modelo canónico de licencias con mapeo gradual de campos legacy.
- Validación de payload con JSON Schema (draft 2020-12) usando Ajv2020.
- Reglas de negocio cruzadas para consistencia operativa (licencias y fechas de contrato).
- Respuestas de error estructuradas por campo (`source`, `code`, `field`, `message`) con mensajes orientados a negocio.
- Documentación técnica actualizada (`README.md` y `docs/schemas/*`).

### v0.3.0 - 2026-05-27

[](https://github.com/vperezguzman66/ControlLicencias#v030---2026-05-27)

- Estandarización de nomenclatura de capacidad de licencia a `licenses_*`.
- Compatibilidad retroactiva para entradas legacy `seats*` y `seats_*` (deprecadas).
- UI y documentación alineadas a terminología local: **Licencias compradas/usadas**.

## Licencia

[](https://github.com/vperezguzman66/ControlLicencias#licencia)

MIT.