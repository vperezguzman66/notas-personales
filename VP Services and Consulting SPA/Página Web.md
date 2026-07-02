Landing corporativa + formulario de contacto en Cloudflare Workers, con backend robusto, trazabilidad de leads y hardening de seguridad en producción.

[[Página Web]]
## Estado final implementado (2026-06)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#estado-final-implementado-2026-06)

Implementación cerrada y operativa con:

- Contact form con validación backend + Turnstile + honeypot.
- Envío de correo principal (`EMAIL.send`) y fallback resiliente a Formspree.
- Persistencia de eventos/leads en D1 (`contact_leads`).
- Rate limiting distribuido con Durable Object.
- API de leads protegida por token + throttling de intentos fallidos.
- Panel admin protegido con Cloudflare Access JWT (Zero Trust) + allowlist de email.
- Bloqueo preventivo de rutas/archivos sensibles comúnmente escaneados por bots.
- Worker modularizado (`src/`) y tests automáticos base.

## Arquitectura

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#arquitectura)

1. Frontend estático (`public/`) envía contactos a `POST /api/contact`.
2. Worker valida payload, anti-bot, límites y persiste en D1.
3. Entrega de correo:
    - Primaria: Cloudflare Email Service.
    - Fallback: Formspree si la primaria falla.
4. Leads se consultan por `GET /api/leads` con doble capa de seguridad:
    - Access JWT + email allowlist.
    - `LEADS_API_TOKEN`.

## Integración con redes sociales (implementado)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#integraci%C3%B3n-con-redes-sociales-implementado)

No fue necesario crear un proyecto nuevo: se implementó sobre este mismo sitio.

Incluye:

- Metadatos Open Graph (`og:*`) para vista previa en LinkedIn/Facebook.
- Metadatos `twitter:*` para tarjeta en X/Twitter.
- URL canónica y `theme-color`.
- Imagen social: `public/assets/social/og-image.svg`.
- Botones de compartir en hero (LinkedIn, Facebook, X, WhatsApp) con URL dinámica del sitio.

## Estructura del proyecto

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#estructura-del-proyecto)

- `public/index.html`: landing y formulario.
- `public/main.js`: lógica UI + Turnstile + submit API.
- `public/style.css`: estilos.
- `public/admin-leads.html`: panel admin de leads.
- `worker.js`: entrypoint del Worker (delgado).
- `src/router.js`: routing principal.
- `src/handlers/contact.js`: flujo de contacto.
- `src/handlers/leads.js`: consulta de leads.
- `src/security.js`: headers y bloqueo de rutas sensibles.
- `src/access.js`: validación Cloudflare Access JWT + allowlist.
- `src/rate-limiter.js`: rate limit y Durable Object.
- `src/helpers.js`, `src/responses.js`, `src/constants.js`: utilitarios.
- `tests/*.test.js`: pruebas (`node --test`).
- `migrations/`: migraciones de D1 (aplicadas automáticamente en cada deploy).
- `wrangler.jsonc`: configuración de runtime/bindings/rutas.

## Configuración Cloudflare (bindings y variables)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#configuraci%C3%B3n-cloudflare-bindings-y-variables)

### Bindings

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#bindings)

- `EMAIL` (send_email)
- `LEADS_DB` (D1)
- `CONTACT_RATE_LIMITER` (Durable Object)
- `ASSETS` (static assets)

### Variables (no secretas)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#variables-no-secretas)

- `CONTACT_TO_EMAIL`
- `CONTACT_FROM_EMAIL`
- `CONTACT_FALLBACK_URL`
- `TURNSTILE_SITE_KEY`
- `ADMIN_ACCESS_MODE` (`off` | `header`)
- `ADMIN_ACCESS_REQUIRE_JWT` (`true` recomendado)
- `ADMIN_ACCESS_TEAM_DOMAIN` (ej: `tu-team.cloudflareaccess.com`)
- `ADMIN_ACCESS_AUD` (Audience tag de Access app)
- `ADMIN_ACCESS_EMAIL_DOMAIN` (opcional)
- `ADMIN_ACCESS_EMAILS` (CSV de correos permitidos)

### Secrets

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#secrets)

- `TURNSTILE_SECRET_KEY`
- `LEADS_API_TOKEN`

## Setup rápido

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#setup-r%C3%A1pido)

1. Autenticación:
    - `npx wrangler whoami`
2. Secret Turnstile:
    - `npx wrangler secret put TURNSTILE_SECRET_KEY`
3. Secret de API leads:
    - `npx wrangler secret put LEADS_API_TOKEN`
4. D1 migrations:
    - `npx wrangler d1 migrations apply vpservices-leads --remote`
5. Deploy:
    - `npx wrangler deploy`

## Zero Trust final (implementado)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#zero-trust-final-implementado)

El proyecto quedó en modo de seguridad final para admin/leads:

- `ADMIN_ACCESS_MODE=header`
- `ADMIN_ACCESS_REQUIRE_JWT=true`
- `ADMIN_ACCESS_TEAM_DOMAIN` configurado
- `ADMIN_ACCESS_AUD` configurado
- `ADMIN_ACCESS_EMAILS` restringido

Comportamiento esperado sin sesión Access válida:

- `/admin-leads` → redirección Access (`302`) o denegación (`403`) según flujo.
- `/api/leads` → redirección/denegación antes de evaluar token.

## Endpoints

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#endpoints)

- `GET /api/contact-config`
- `POST /api/contact`
- `GET /api/leads`
- `GET /admin/leads`

### `POST /api/contact`

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#post-apicontact)

Payload esperado:

- `nombre` (req)
- `empresa` (opt)
- `email` (req)
- `servicio` (opt)
- `mensaje` (req)
- `website` (honeypot)
- `turnstileToken` (req si Turnstile secret está activo)

Errores comunes:

- `400` validación/captcha
- `415` content-type inválido
- `429` rate limit
- `502` fallback también falló
- `503` servicio no disponible

### `GET /api/leads`

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#get-apileads)

Requiere:

1. Pasar validación Access JWT + allowlist (si `ADMIN_ACCESS_MODE=header`).
2. Header `Authorization: Bearer <LEADS_API_TOKEN>`.

Query params:

- `limit` (1-100)
- `offset`
- `status`
- `channel`
- `email`

## Seguridad implementada (resumen)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#seguridad-implementada-resumen)

- Validación backend estricta de payload.
- Honeypot anti-bot.
- Turnstile server-side (`siteverify`).
- Rate limit para contacto y auth fallida de leads.
- D1 para trazabilidad de entregas/bloqueos.
- Headers de hardening (`CSP`, `HSTS`, `X-Frame-Options`, etc.).
- `X-Robots-Tag: noindex, nofollow` en `/api/*` y superficies admin.
- Bloqueo explícito de rutas sensibles:
    - `/.env`, `/.git/*`, `/.wrangler/*`, `.claude/*`
    - `/credentials.json`, `/secrets.json`, `/id_rsa`, `/.aws/credentials`, etc.
- Zero Trust Access JWT (fail-closed) + allowlist de email.

## Verificación funcional realizada

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#verificaci%C3%B3n-funcional-realizada)

- Turnstile de prueba reemplazado por llave real.
- API leads con token válido responde `200`.
- Admin/leads bajo Access responden con redirección/denegación sin sesión válida.
- Rutas sensibles escaneadas por bots responden `404`.
- Tests locales: `npm test` pasando.

## Desarrollo y operación

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#desarrollo-y-operaci%C3%B3n)

- Dev local: `npx wrangler dev`
- Test: `npm test`
- Smoke de seguridad: `npm run smoke:security`
- Watch de alertas de seguridad: `npm run alerts:security`
- Daemon 24/7 de alertas: `npm run alerts:security:daemon`
- Instalar daemon 24/7 en macOS (launchd): `npm run alerts:security:install-macos`
- Check semanal manual: `npm run check:security:weekly`
- Instalar check semanal en macOS (launchd): `npm run check:security:weekly:install-macos`
- Backup D1 manual: `npm run backup:d1`
- Backup D1 dry-run: `npm run backup:d1:dry`
- Instalar backup D1 semanal en macOS (launchd): `npm run backup:d1:install-macos`
- Restore drill D1 dry-run: `npm run restore:d1:drill:dry`
- Restore drill D1 real: `npm run restore:d1:drill`
- Deploy: `npx wrangler deploy`

## CI/CD automatizado (GitHub Actions)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#cicd-automatizado-github-actions)

Se agregaron workflows para validación continua:

- `.github/workflows/ci.yml`
    
    - Trigger: `push` y `pull_request` sobre `main`.
    - Ejecuta: `npm test`, validación sintáctica de scripts `bash`, y `token:rotate:leads:dry`.
- `.github/workflows/security-smoke.yml`
    
    - Trigger: manual (`workflow_dispatch`) y semanal (lunes 13:00 UTC).
    - Ejecuta `npm run smoke:security` contra `https://vpservices-it.com`.
- `.github/dependabot.yml`
    
    - Actualización automática semanal de dependencias `npm` y `github-actions`.
    - PRs etiquetados con `dependencies` y `security`.
- `.github/workflows/dependabot-automerge.yml`
    
    - Auto-aprueba PRs de Dependabot **solo** para updates `semver-patch` y `semver-minor` en ecosistemas `npm` y `github-actions`.
    - Habilita auto-merge **solo** para updates `semver-patch` y `semver-minor`.
    - Los `semver-major` quedan manuales.

Política operativa aplicada:

- No se debe hacer merge a `main` si el workflow `CI` está en estado fallido.

### Rutina operativa sugerida (mensual)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#rutina-operativa-sugerida-mensual)

1. Ejecutar `npm run smoke:security`.
2. Revisar `wrangler tail` para picos de `401`, `403` y `429`.
3. Confirmar que `/admin-leads` y `/api/leads` sigan bajo Access (`302/403` sin sesión autorizada).
4. Validar que rutas sensibles continúen en `404`.

### Check semanal automatizado (implementado)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#check-semanal-automatizado-implementado)

Se agregó un chequeo semanal con script y scheduler para macOS:

- Script: `scripts/weekly_security_check.sh`
- Comando manual: `npm run check:security:weekly`
- Instalación launchd (lunes 09:00): `npm run check:security:weekly:install-macos`

El check semanal valida:

1. `smoke:security` contra `SMOKE_BASE_URL`.
2. Estado del servicio `com.vpservices.security-alerts`.
3. Últimas líneas del log del daemon de alertas.
4. Presencia del plist del servicio 24/7.

Variables opcionales:

- `WEEKLY_CHECK_LOG_DIR` (default: `./logs`)
- `WEEKLY_CHECK_REPORT_FILE` (default: `./logs/weekly-security-check.log`)
- `WEEKLY_CHECK_SERVICE_NAME` (default: `com.vpservices.weekly-security-check`)

### Backup D1 automatizado (implementado)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#backup-d1-automatizado-implementado)

Se agregó backup automático de la base `vpservices-leads` con retención:

- Script: `scripts/d1_backup.sh`
- Instalador launchd: `scripts/install_d1_backup_launchd.sh`
- Schedule por defecto: lunes 03:30 (hora local macOS)
- Retención por defecto: `90` días
- Salida: `backups/d1/*.sql.gz` + enlace `backups/d1/latest.sql.gz`

Variables opcionales:

- `D1_DB_NAME` (default: `vpservices-leads`)
- `D1_BACKUP_DIR` (default: `./backups/d1`)
- `D1_BACKUP_RETENTION_DAYS` (default: `90`)
- `D1_BACKUP_REMOTE` (default: `true`)
- `D1_BACKUP_COMPRESS` (default: `true`)
- `D1_BACKUP_DRY_RUN` (default: `false`)
- `D1_BACKUP_SERVICE_NAME` (default: `com.vpservices.d1-backup`)

Política detallada:

- `docs/ops/d1-backup-policy.md`

### Restore drill D1 (implementado)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#restore-drill-d1-implementado)

Se agregó ensayo real de restauración para validar recuperabilidad del backup:

- Script: `scripts/d1_restore_drill.sh`
- Dry-run: `npm run restore:d1:drill:dry`
- Ensayo real: `npm run restore:d1:drill`
- Log de evidencia: `logs/d1-restore-drill.log`

Runbook:

- `docs/ops/d1-restore-runbook.md`

### Alertado básico por umbral (implementado)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#alertado-b%C3%A1sico-por-umbral-implementado)

El Worker emite eventos estructurados `SECURITY_EVENT` para:

- `401`: token inválido en `/api/leads`
- `403`: denegación por política de Access/allowlist
- `429`: rate limit (contacto o auth fallida en leads)

El watcher `npm run alerts:security` detecta estos eventos en stream y alerta cuando se superan umbrales por ventana.

Para operación continua local (24/7), usar `npm run alerts:security:daemon`.

En macOS, se puede registrar como servicio de usuario con `npm run alerts:security:install-macos`. Esto crea un `LaunchAgent` que reinicia el watcher automáticamente si se cae.

Variables opcionales de tuning (entorno local del watcher):

- `ALERT_WINDOW_MS` (default: `300000`)
- `ALERT_401_THRESHOLD` (default: `20`)
- `ALERT_403_THRESHOLD` (default: `20`)
- `ALERT_429_THRESHOLD` (default: `15`)

Variables opcionales del daemon:

- `ALERT_DAEMON_LOG_DIR` (default: `./logs`)
- `ALERT_DAEMON_LOG_FILE` (default: `./logs/security-alert-daemon.log`)
- `ALERT_DAEMON_RESTART_DELAY_SEC` (default: `10`)

## Rotación de token (`LEADS_API_TOKEN`)

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#rotaci%C3%B3n-de-token-leads_api_token)

Implementado script de rotación:

- Aplicar nueva rotación: `npm run token:rotate:leads`
- Simulación (sin escribir secret remoto): `npm run token:rotate:leads:dry`

Política y procedimiento detallado en:

- `docs/ops/token-rotation.md`

## Runbook de incidentes de seguridad

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#runbook-de-incidentes-de-seguridad)

Se agregó runbook operativo para eventos `401/403/429` con:

- clasificación de severidad,
- triage inicial (primeros 10 min),
- acciones de contención por tipo de evento,
- recuperación y postmortem mínimo.

Documento:

- `docs/ops/security-incident-runbook.md`

## Troubleshooting rápido

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#troubleshooting-r%C3%A1pido)

- **"Debes ingresar el token LEADS_API_TOKEN"**
    
    - El panel admin pide pegar el token en su input antes de consultar.
- **`302` en `/admin-leads` o `/api/leads`**
    
    - Access está activo y te está redirigiendo a autenticación.
- **`403` en admin/leads aun autenticado**
    
    - Revisar `ADMIN_ACCESS_AUD`, `ADMIN_ACCESS_TEAM_DOMAIN` y allowlist (`ADMIN_ACCESS_EMAILS` / `ADMIN_ACCESS_EMAIL_DOMAIN`).
- **Wrangler warning de drift de rutas**
    
    - Se corrigió declarando `routes` en `wrangler.jsonc`.

## Buenas prácticas operativas

[](https://github.com/vperezguzman66/vpservices-web/blob/main/README.md#buenas-pr%C3%A1cticas-operativas)

- Rotar `LEADS_API_TOKEN` periódicamente.
- Mantener mínima la allowlist de admins.
- Auditar eventos `401/403/429` en logs.
- No commitear secretos en git (`.env` está ignorado).
## Branding LinkedIn (2026-06-30)

Assets de imagen de marca añadidos y publicados en LinkedIn:

### Página de empresa (VP Services & Consulting SpA)

- URL: `linkedin.com/company/135184612`
- Banner subido: `public/assets/social/banner-linkedin-company.png`
- Dimensiones: 1128 × 191 px
- Diseño: fondo navy oscuro, grilla diagonal, barra accent cian, nombre de empresa, tagline y URL `vpservices-it.com`
- Descripción (resumen) también configurada en el perfil de empresa

### Perfil personal (Víctor Pérez Guzmán)

- URL: `linkedin.com/in/victorperezg`
- Banner subido: `public/assets/social/banner-linkedin-personal.png`
- Dimensiones: 1584 × 396 px
- Diseño: gradiente navy, grilla diagonal, barra accent cian izquierda, nombre, cargo "Senior Developer & Tech Consultant", especialidades y empresa

### Assets en el repositorio

Ambos archivos guardados en `vpservices-web/public/assets/social/`:

| Archivo | Tamaño | Uso |
|---|---|---|
| `banner-linkedin-company.png` | ~47 KB | Portada página de empresa LinkedIn |
| `banner-linkedin-personal.png` | ~40 KB | Portada perfil personal LinkedIn |
| `og-image.svg` | — | Open Graph / vista previa en redes |

## Instagram (2026-06-30)

Cuenta creada y configurada con branding inicial:

- **Usuario**: `@vpservicesconsulting`
- **Nombre de perfil**: VP Services & Consulting SpA

### Story Highlights

5 Highlights publicados con covers diseñados en navy/cian (coherentes con identidad LinkedIn):

| Highlight | Ícono | Archivo cover |
|---|---|---|
| Servicios | Líneas horizontales | `ig-highlight-servicios.png` |
| Proyectos | `</>` | `ig-highlight-proyectos.png` |
| Cloud | Nube | `ig-highlight-cloud.png` |
| IA & Agentes | Red neuronal | `ig-highlight-ia-agentes.png` |
| Contacto | Sobre/email | `ig-highlight-contacto.png` |

Assets guardados en `vpservices-web/public/assets/social/ig-highlight-*.png` (1080×1920 px). Mergeados en PR #51.

### Foto de perfil

- Archivo: `public/assets/social/ig-profile.png`
- Dimensiones: 800×800 px
- Diseño: fondo navy con grilla diagonal, anillo cian, "VP" en blanco, línea divisora cian, "SERVICES" en cian
- Mergeado en PR #52 (2026-06-30)

### Logo página de empresa LinkedIn

- Archivo: `public/assets/social/ig-profile.png` (mismo que Instagram)
- Subido a la página de empresa `linkedin.com/company/vp-services-consulting-spa` (2026-07-01)
- Branding consistente: navy + anillo cian + "VP SERVICES" en blanco/cian
