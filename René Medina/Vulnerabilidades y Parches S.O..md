[[Solicitudes de Software]]
# VulnApp V2.4.0

## Estado (2026-07-02)

Sprint **S4** cerrado: sprint de bugs (S4-01 a S4-07) + auditoría técnica formal del 2026-06-16 (`docs/AUDIT_REPORT_2026-06-16.md`, 9 hallazgos F-01 a F-09). Se remediaron las 5 prioridades de la auditoría — timeout WinRM, job queue persistida en SQLite, auditoría funcional (`audit_event`), límites de consulta homogeneizados, endurecimiento de API key/secrets — organizadas en 8 commits separados por bloque en `main`. **216/216 tests pasan.** `ROADMAP.md` y `CHANGELOG.md` ya reflejan el sprint completo.

**Pendiente de decidir:**
- Qué hacer con el diff de `app/data/vulnapp.db` (binario, cambios de datos locales de prueba, no se commiteó)
- Carpeta `memory/` sin trackear en el repo (notas propias del proyecto, no código) — decidir si se ignora o se limpia

Hallazgos de la auditoría aceptados como deuda técnica (no bloqueantes): F-03 (fallback NVD sin ventana temporal), F-06 (SQLite dentro del árbol de la app), F-07 (notificaciones sin observabilidad fuerte), F-09 (caché KEV a vigilar en alta concurrencia).

### Smoke test end-to-end (2026-07-02, tarde)

Se levantó el servidor localmente (`uvicorn app.main:app`, puerto 8000) y se verificó en vivo, no solo con la suite de tests:

- `GET /api/health` → 200; endpoints protegidos devuelven 401 sin `X-API-Key` y 200 con la key correcta (`/api/reports`, `/api/jobs`).
- Se creó y eliminó un schedule de prueba con password real en el payload — la respuesta de `POST /api/schedules` confirmó que **no expone** `collect_request` ni la contraseña (solo `linux_targets_count`), validando en vivo el fix de seguridad del sprint S4.
- Log del servidor mostró los eventos de auditoría funcionando (`AUDIT schedule_created`, `AUDIT schedule_deleted`).
- La lista de `/api/jobs` trajo jobs de sesiones de prueba anteriores — confirma en la práctica que la persistencia de la job queue en SQLite (fix F-02) sobrevive a reinicios del proceso.
- Dashboard (`/dashboard`) abierto en el navegador con la API key cargada: 18 reportes / 26 targets visibles, detalle de escaneo (`scan-hist-05`, host `srv-01`) renderizando findings, filtros por categoría técnica y severidad.
- Servidor detenido al terminar la verificación.

[](https://github.com/vperezguzman66/vulnapp#vulnapp-v240)

Aplicación API modular para **detectar vulnerabilidades (CVE)** en servidores, gestionar el ciclo de vida de la remediación y colectar inventario remoto de forma automática y programada.

Fuentes de inteligencia:

- **NVD (NIST)**: catálogo CVE y metadatos técnicos
- **CISA KEV**: vulnerabilidades conocidas como explotadas
- **FIRST EPSS**: probabilidad de explotación (score)
- **Fabricantes**: validación por dominios oficiales en referencias CVE

Capacidades V2.4.0:

- Hardening de seguridad completo: timing-safe auth, XSS, CSP, rate limiting, headers HTTP, `SecretStr` en contraseñas
- Collectors remotos con versiones de paquetes, actualizaciones pendientes e inventario automático
- Gestión de assets persistente con soporte de entornos, etiquetas, cobertura de scans e historial de riesgo
- Estado de remediación por finding con historial de auditoría y verificación de parches automatizada
- Scans asíncronos con polling de estado
- **Scheduler**: scans periódicos programados por entorno (APScheduler)
- **Notificaciones**: alertas por webhook y email (SMTP) al superar umbral de riesgo
- Collector SSH con soporte de bastion/jump host
- Registro de errores de collector por asset

## Arquitectura modular

[](https://github.com/vperezguzman66/vulnapp#arquitectura-modular)

- `app/main.py` — arranque FastAPI, middleware de seguridad, headers HTTP, CSP
- `app/api/routes.py` — endpoints de scan, reportes y jobs
- `app/api/asset_routes.py` — endpoints de assets, remediación, cobertura, historial de riesgo, verificación de parches, errores de collector
- `app/api/schedule_routes.py` — endpoints de schedules de scan
- `app/api/notification_routes.py` — endpoints de notificaciones
- `app/models/schemas.py` — contratos de entrada/salida (Pydantic v2 con `SecretStr`)
- `app/config.py` — configuración y variables de entorno
- `app/security/api_key.py` — autenticación por API key (timing-safe)
- `app/services/correlation.py` — motor de correlación y scoring
- `app/services/scan_orchestrator.py` — orquesta recolección + escaneo
- `app/services/scheduler.py` — scheduler APScheduler para scans periódicos
- `app/services/notifier.py` — envío de alertas por webhook y email
- `app/services/job_queue.py` — cola de jobs asíncronos persistida en SQLite (`job_queue_repository.py`)
- `app/services/http_client.py` — cliente HTTP con retry
- `app/services/sources/nvd_source.py` — consulta CVEs NVD
- `app/services/sources/cisa_kev_source.py` — feed CISA KEV con caché compartida de clase (TTL 4h)
- `app/services/sources/epss_source.py` — enriquecimiento EPSS en batch
- `app/services/collectors/linux_ssh_collector.py` — inventario Linux vía SSH (con soporte jump host)
- `app/services/collectors/windows_winrm_collector.py` — inventario Windows vía WinRM
- `app/services/storage/repository.py` — contrato de persistencia de reportes
- `app/services/storage/sqlite_repository.py` — backend SQLite
- `app/services/storage/json_repository.py` — backend JSON compatible
- `app/services/storage/postgres_repository.py` — backend PostgreSQL
- `app/services/storage/asset_repository.py` — persistencia de assets
- `app/services/storage/remediation_repository.py` — estado de remediación e historial
- `app/services/storage/schedule_repository.py` — persistencia de schedules (serializa `SecretStr` correctamente)
- `app/services/storage/verification_repository.py` — resultados de verificación de parches
- `app/services/storage/collector_error_repository.py` — registro de errores de collector
- `app/services/storage/repository_factory.py` — selección de backend por configuración
- `app/dashboard/routes.py` — endpoint del dashboard
- `app/dashboard/static/` — frontend del dashboard (HTML/CSS/JS, usa `sessionStorage`)

## Requisitos

[](https://github.com/vperezguzman66/vulnapp#requisitos)

- Python 3.11+

## Instalación

[](https://github.com/vperezguzman66/vulnapp#instalaci%C3%B3n)

```shell
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
python3 -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Verificación:

```shell
curl http://127.0.0.1:8000/api/health
```

## Configuración

[](https://github.com/vperezguzman66/vulnapp#configuraci%C3%B3n)

Variables de entorno relevantes (`.env`):

|Variable|Default|Descripción|
|---|---|---|
|`VULNAPP_API_KEY`|`CHANGE_ME`|API key obligatoria (falla al arrancar si tiene el valor default o menos de 32 caracteres)|
|`VULNAPP_AUTH_ENABLED`|`true`|Habilita autenticación|
|`VULNAPP_STORAGE_BACKEND`|`sqlite`|Backend de persistencia (`sqlite\|json\|postgres`)|
|`VULNAPP_SQLITE_PATH`|`app/data/vulnapp.db`|Ruta de la base de datos SQLite|
|`VULNAPP_POSTGRES_DSN`|—|DSN PostgreSQL (solo si backend=`postgres`)|
|`VULNAPP_NVD_API_KEY`|—|API key NVD (elimina rate limiting de 5 req/30s)|
|`VULNAPP_DEBUG`|`false`|Habilita `/docs`, `/redoc` y `/openapi.json`|
|`VULNAPP_SCHEDULER_ENABLED`|`false`|Activa el scheduler de scans periódicos|
|`VULNAPP_SCAN_INTERVAL_HOURS`|`24`|Intervalo por defecto para schedules (horas)|
|`VULNAPP_SCHEDULER_INTERVAL_PRODUCTION`|`24`|Intervalo para entorno `production` (horas)|
|`VULNAPP_SCHEDULER_INTERVAL_DMZ`|`48`|Intervalo para entorno `dmz` (horas)|
|`VULNAPP_SCHEDULER_INTERVAL_STAGING`|`168`|Intervalo para entorno `staging` (horas)|
|`VULNAPP_SCHEDULER_INTERVAL_DEV`|`336`|Intervalo para entorno `dev` (horas)|
|`VULNAPP_WEBHOOK_URL`|—|URL de webhook para alertas (p. ej. Slack)|
|`VULNAPP_SMTP_HOST`|—|Host SMTP para alertas por email|
|`VULNAPP_SMTP_PORT`|`587`|Puerto SMTP|
|`VULNAPP_SMTP_USER`|—|Usuario SMTP|
|`VULNAPP_SMTP_PASSWORD`|—|Contraseña SMTP|
|`VULNAPP_SMTP_FROM`|`vulnapp@localhost`|Remitente de alertas email|
|`VULNAPP_SMTP_TO`|—|Destinatario(s) email (lista separada por comas)|
|`VULNAPP_SMTP_USE_TLS`|`true`|Usar TLS en SMTP|
|`VULNAPP_ALERT_RISK_THRESHOLD`|`70`|Score de riesgo a partir del cual se envía alerta|
|`VULNAPP_ALERT_ENVIRONMENTS`|`production`|Entornos que disparan alertas (separados por comas)|
|`VULNAPP_COVERAGE_ALERT_DAYS`|`30`|Días sin scan para marcar asset como `critical` en cobertura|
|`VULNAPP_RISK_WEIGHT_CVSS`|—|Peso del componente CVSS en el score compuesto|
|`VULNAPP_RISK_WEIGHT_EPSS`|—|Peso del componente EPSS en el score compuesto|
|`VULNAPP_RISK_WEIGHT_KEV`|—|Peso del componente KEV en el score compuesto|

## Endpoints

[](https://github.com/vperezguzman66/vulnapp#endpoints)

### Escaneo y reportes

[](https://github.com/vperezguzman66/vulnapp#escaneo-y-reportes)

|Método|Endpoint|Descripción|
|---|---|---|
|`GET`|`/api/health`|Estado del servicio (sin auth)|
|`POST`|`/api/scan`|Escaneo síncrono por inventario → `ScanReport`|
|`POST`|`/api/collect-and-scan`|Recolección remota + escaneo asíncrono → `202 ScanJob`|
|`GET`|`/api/reports`|Lista reportes persistidos (`?limit=`)|
|`GET`|`/api/reports/{scan_id}`|Reporte completo|
|`GET`|`/api/jobs`|Lista de jobs recientes (`?limit=`)|
|`GET`|`/api/jobs/{job_id}`|Estado de job (`PENDING\|RUNNING\|COMPLETED\|FAILED`)|

### Assets y remediación

[](https://github.com/vperezguzman66/vulnapp#assets-y-remediaci%C3%B3n)

|Método|Endpoint|Descripción|
|---|---|---|
|`GET`|`/api/assets`|Lista assets (`?environment=production&tag=web`)|
|`POST`|`/api/assets`|Registrar asset|
|`GET`|`/api/assets/coverage`|Cobertura de scans (`?environment=&tag=`)|
|`GET`|`/api/assets/{id}`|Detalle de asset|
|`PUT`|`/api/assets/{id}`|Actualizar asset|
|`DELETE`|`/api/assets/{id}`|Eliminar asset|
|`GET`|`/api/assets/{id}/findings`|CVEs del último scan con estado de remediación|
|`PATCH`|`/api/assets/{id}/findings/{cve_id}/status`|Actualizar estado de remediación|
|`GET`|`/api/assets/{id}/findings/history`|Historial de cambios de estado|
|`GET`|`/api/assets/{id}/risk-history`|Historial de risk score (`?days=90`)|
|`POST`|`/api/assets/{id}/verify-patch`|Verificación de parches asíncrona → `202 ScanJob`|
|`GET`|`/api/assets/{id}/verify-patch/latest`|Último resultado de verificación de parches|
|`GET`|`/api/assets/{id}/errors`|Errores de collector del asset|

### Schedules de scan

[](https://github.com/vperezguzman66/vulnapp#schedules-de-scan)

|Método|Endpoint|Descripción|
|---|---|---|
|`GET`|`/api/schedules/default-interval`|Intervalo por defecto por entorno|
|`GET`|`/api/schedules`|Lista schedules (`ScanScheduleRead` — sin credenciales)|
|`POST`|`/api/schedules`|Crear schedule|
|`GET`|`/api/schedules/{id}`|Detalle de schedule|
|`PUT`|`/api/schedules/{id}`|Actualizar schedule|
|`DELETE`|`/api/schedules/{id}`|Eliminar schedule|
|`POST`|`/api/schedules/{id}/run`|Disparar schedule manualmente → `202 ScanJob`|

### Notificaciones

[](https://github.com/vperezguzman66/vulnapp#notificaciones)

|Método|Endpoint|Descripción|
|---|---|---|
|`POST`|`/api/notifications/test`|Enviar notificación de prueba|

## Flujo de uso principal

[](https://github.com/vperezguzman66/vulnapp#flujo-de-uso-principal)

### Scan asíncrono (recomendado para colección remota)

[](https://github.com/vperezguzman66/vulnapp#scan-as%C3%ADncrono-recomendado-para-colecci%C3%B3n-remota)

```shell
# 1. Lanzar scan
JOB=$(curl -s -X POST "http://127.0.0.1:8000/api/collect-and-scan" \
  -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -d @sample_collect_and_scan_request.json)
JOB_ID=$(echo $JOB | jq -r .job_id)

# 2. Polling hasta completar
until [ "$(curl -s -H "X-API-Key: $KEY" .../api/jobs/$JOB_ID | jq -r .status)" = "COMPLETED" ]; do sleep 5; done

# 3. Obtener reporte
SCAN_ID=$(curl -s -H "X-API-Key: $KEY" .../api/jobs/$JOB_ID | jq -r .scan_id)
curl -H "X-API-Key: $KEY" .../api/reports/$SCAN_ID
```

### Gestión de remediación

[](https://github.com/vperezguzman66/vulnapp#gesti%C3%B3n-de-remediaci%C3%B3n)

```shell
# Ver findings de un asset con su estado de remediación
curl -H "X-API-Key: $KEY" "/api/assets/asset-abc123/findings"

# Marcar CVE como en progreso
curl -X PATCH -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  "/api/assets/asset-abc123/findings/CVE-2024-9999/status" \
  -d '{"status": "IN_PROGRESS", "owner": "ops-team", "due_date": "2026-07-01"}'

# Verificar si el parche fue efectivo
curl -s -X POST -H "X-API-Key: $KEY" \
  "/api/assets/asset-abc123/verify-patch" | jq .job_id
```

### Crear un schedule de scan periódico

[](https://github.com/vperezguzman66/vulnapp#crear-un-schedule-de-scan-peri%C3%B3dico)

```shell
curl -X POST -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  "/api/schedules" \
  -d '{
    "name": "Scan nocturno producción",
    "interval_hours": 24,
    "enabled": true,
    "collect_request": {
      "linux_targets": [{"hostname": "srv-01", "address": "10.0.1.10", "username": "ubuntu"}],
      "windows_targets": [],
      "lookback_days": 30
    }
  }'
```

## Estados de remediación

[](https://github.com/vperezguzman66/vulnapp#estados-de-remediaci%C3%B3n)

`PENDING` → `IN_PROGRESS` → `PATCHED` → `VERIFIED`

Adicionales: `ACCEPTED_RISK`, `MITIGATED`, `FALSE_POSITIVE`

## Dashboard

[](https://github.com/vperezguzman66/vulnapp#dashboard)

`GET /dashboard` — interfaz web con filtros por categoría técnica, severidad y gestión de reportes. La API key se guarda en `sessionStorage` (se borra al cerrar la pestaña).

## Persistencia

[](https://github.com/vperezguzman66/vulnapp#persistencia)

Backend por defecto: **SQLite** (`app/data/vulnapp.db`). Recomendado para single-node.

> El archivo SQLite dentro del directorio de la aplicación genera un warning al arrancar. Configura `VULNAPP_SQLITE_PATH` fuera del árbol de la app para producción.

## Limitaciones

[](https://github.com/vperezguzman66/vulnapp#limitaciones)

- Cola de jobs persistida en SQLite: se conserva al reiniciar el servidor, pero sigue siendo single-node
- Correlación por keyword/CPE en NVD (sin fingerprinting profundo de versiones)
- Collector Linux requiere cliente `ssh` del sistema
- Collector Windows requiere `pywinrm` y WinRM habilitado en el target

## Troubleshooting

[](https://github.com/vperezguzman66/vulnapp#troubleshooting)

|Error|Causa probable|
|---|---|
|`RuntimeError: API key placeholder`|`VULNAPP_API_KEY` sigue en `CHANGE_ME`|
|`401 Unauthorized`|Header `X-API-Key` ausente o incorrecto|
|`400` en collect-and-scan|Fallo de collector (SSH caído, credencial inválida)|
|`502` en scan|NVD/CISA/EPSS no disponible temporalmente|
|Dashboard vacío|Ejecuta un scan primero|
|Schedule no dispara|Verificar `VULNAPP_SCHEDULER_ENABLED=true` y que el schedule tenga `enabled: true`|

## Documentación completa

[](https://github.com/vperezguzman66/vulnapp#documentaci%C3%B3n-completa)

- [`docs/API_V2.md`](https://github.com/vperezguzman66/vulnapp/blob/main/docs/API_V2.md)
- [`docs/ARCHITECTURE_V2.md`](https://github.com/vperezguzman66/vulnapp/blob/main/docs/ARCHITECTURE_V2.md)
- [`docs/OPERATIONS.md`](https://github.com/vperezguzman66/vulnapp/blob/main/docs/OPERATIONS.md)
- [`CHANGELOG.md`](https://github.com/vperezguzman66/vulnapp/blob/main/CHANGELOG.md)
- [`ROADMAP.md`](https://github.com/vperezguzman66/vulnapp/blob/main/ROADMAP.md)

## Uso responsable

[](https://github.com/vperezguzman66/vulnapp#uso-responsable)

Esta herramienta debe usarse **solo con autorización explícita** sobre infraestructura propia o administrada legítimamente.