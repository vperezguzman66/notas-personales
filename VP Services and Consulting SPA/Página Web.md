---
proyecto: "vpservices-web"
ruta: "Proyectos/vpservices-web"
cliente: "VP Services (propio)"
stack: "Cloudflare Workers + D1"
estado: "Desplegado — SEO en curso, 1 de 4 páginas de servicio ya indexada"
ultimo_cambio: 2026-07-04
---

Landing corporativa + formulario de contacto en Cloudflare Workers, con backend robusto, trazabilidad de leads y hardening de seguridad en producción.

[[Página Web]]

## Estado y pendientes (2026-07-04)

Todo el trabajo hasta el PR #66 (`fix(diseño): equilibra contacto y diversifica acentos de color`) está mergeado a `main` y pusheado a `origin/main` — sin cambios locales pendientes de subir.

Las tres rutinas programadas para hoy (2026-07-04) ya se ejecutaron — resultados y detalle completo en la sección "Google Search Console" más abajo. Resumen: 1 de 4 páginas nuevas ya indexada, 2 en cola normal de descubrimiento, 1 (`soporte-y-mantenimiento`) aún no descubierta por Google. El OAuth temporal quedó revocado y confirmado muerto.

**Pendiente — 2026-07-11:** rutina de recordatorio (`trig_016zg6z5kvRiMKuLs9U1JYpb`) para revisar manualmente si `soporte-y-mantenimiento` ya fue descubierta/indexada (ver detalle abajo).

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

## SEO — auditoría técnica y páginas de servicio (2026-07-01)

Auditoría completa del sitio (Lighthouse mobile: Accesibilidad 100, Best Practices 100, SEO 100) confirmó que lo técnico ya estaba resuelto — el problema real de visibilidad era falta de contenido indexable (una sola página) y de señales externas (Search Console, Google Business Profile, backlinks).

### Páginas de servicio nuevas

Se agregaron 4 páginas de contenido indexables bajo `public/servicios/`, cada una con meta tags únicos, breadcrumb, JSON-LD (`Service` + `BreadcrumbList` + `FAQPage`) y CTA hacia `/#contacto`:

| Página | URL final |
|---|---|
| Consultoría TI | `vpservices-it.com/servicios/consultoria-ti` |
| Desarrollo de Software | `vpservices-it.com/servicios/desarrollo-de-software` |
| Inteligencia Artificial | `vpservices-it.com/servicios/inteligencia-artificial` |
| Soporte y Mantenimiento | `vpservices-it.com/servicios/soporte-y-mantenimiento` |

Detalle importante: Cloudflare Workers Assets redirige automáticamente `/pagina.html` → `/pagina` (307). Todas las URLs canónicas, OG, JSON-LD y enlaces internos quedaron apuntando a la ruta **sin** `.html` para evitar ese salto de redirección innecesario.

`sitemap.xml` actualizado con las 5 URLs (antes solo tenía el home). Home (`index.html`) ahora incluye enlaces "Leer más" desde cada tarjeta de servicio hacia su página de detalle.

### Fix de bug en build

`scripts/build.mjs` (el script que minifica y hashea `.css`/`.js`) solo reescribía referencias de archivo sueltas (`"style.css"`), no rutas absolutas (`"/style.css"`) — las páginas nuevas (que usan rutas absolutas porque viven en un subdirectorio) quedaban con CSS/JS en 404. Corregido para soportar ambos formatos.

`public/main.js`: el manejo del formulario de contacto ahora usa optional chaining, porque solo el home tiene ese formulario — sin esto, las páginas de servicio rompían el navbar/menú móvil por un error de JS al cargar.

### Deploy y control de versiones

- Deployado a producción (`npx wrangler deploy`) y verificado en vivo (200 sin redirects en las 5 URLs).
- Commit `feat(seo): agregar páginas de servicio y actualizar sitemap` → PR [#53](https://github.com/vperezguzman66/vpservices-web/pull/53) → mergeado a `main` tras pasar CI (squash + rama eliminada).

## Google Search Console (2026-07-01)

- Propiedad `sc-domain:vpservices-it.com` ya estaba verificada y con el sitemap enviado desde antes (30 jun), pero solo tenía 1 página descubierta.
- Sitemap reenviado manualmente tras el deploy; confirmado por API que pasó de 1 a 5 URLs "submitted" el mismo día.

### Chequeo automatizado de indexación (temporal)

Para verificar en unos días si Google indexó las 4 páginas nuevas, se creó infraestructura temporal de solo lectura:

- Proyecto GCP nuevo: `vpservices-search-console` (ID `firm-streamer-501200-f5`), con la Search Console API habilitada.
- OAuth client tipo "App de escritorio", modo prueba (usuario de prueba: `vperezguzman@gmail.com`), scope único `webmasters.readonly`.
- Dos rutinas programadas (Claude Code routines, one-shot) para **2026-07-04**:
  - `14:00 UTC` — "GSC indexing check - vpservices-it.com service pages": consulta sitemap + URL Inspection API de las 4 páginas nuevas y el home, entrega reporte.
  - `14:15 UTC` — "Revoke vpservices-search-console OAuth access": revoca el refresh token vía API y verifica que quedó inválido.
- El proyecto GCP en sí queda existiendo (vacío, sin costo) después de la revocación, salvo que se pida borrarlo explícitamente.
- Ambas rutinas tienen notificación push (Claude Code / Remote Control) habilitada, pero **solo la disparan si algo falla** (refresh token rechazado, revoke que no invalidó el token de verdad). Si todo sale bien, no avisan proactivamente — el reporte queda en el dashboard de rutinas.
- Fallback manual si el revoke automático falla: <https://myaccount.google.com/permissions> con la cuenta `vperezguzman@gmail.com`, buscar "VP Services Search Console Checker" y quitar el acceso.
- Documentación técnica detallada en el repo: `docs/ops/gsc-indexing-check.md`.

### Resultados del chequeo (2026-07-04)

Las tres rutinas corrieron según lo programado y se revisaron en el dashboard (https://claude.ai/code/routines/):

| URL | coverageState | Verdict |
|---|---|---|
| `/servicios/consultoria-ti` | Submitted and indexed | ✅ PASS |
| `/servicios/desarrollo-de-software` | Discovered, aún no indexada | Neutral (normal a 3 días) |
| `/servicios/inteligencia-artificial` | Discovered, aún no indexada | Neutral (normal a 3 días) |
| `/servicios/soporte-y-mantenimiento` | **URL unknown to Google** (no descubierta) | Neutral, rezagada |
| `/` (home, baseline) | Submitted and indexed | ✅ PASS |

Sitemap: 7 submitted / 0 indexed (métrica que va rezagada respecto al estado real por URL, no es señal de alarma).

Evaluación de la rutina: "on track, pero desparejo" — 1 de 4 páginas nuevas indexada a los 3 días es buena señal temprana; el caso a vigilar es `soporte-y-mantenimiento`, que ni siquiera fue descubierta (a diferencia de las otras 3 que sí, aunque no indexadas aún).

La revocación del OAuth (10:15 AM) se confirmó exitosa: revoke devolvió HTTP 200, y el intento posterior de refrescar el token con el mismo `refresh_token` devolvió `invalid_grant` — credenciales confirmadas muertas. El recordatorio de las 10:30 AM se disparó según lo esperado.

### Pendiente — recheck de `soporte-y-mantenimiento` (2026-07-11)

Se creó una rutina one-shot (`trig_016zg6z5kvRiMKuLs9U1JYpb`, 2026-07-11 10:00 AM GMT-4) para recordar revisar manualmente el estado de esa URL en Search Console.

**Por qué es manual y no vía API:** al intentar generar credenciales OAuth nuevas para automatizar este recheck (usando el mismo client_id del proyecto `vpservices-search-console` vía OAuth Playground), Google rechazó la autorización con **Error 400: redirect_uri_mismatch** — el OAuth Client no tiene registrado `https://developers.google.com/oauthplayground` como redirect URI válido (el refresh_token original de esta app se generó por otra vía, no por el Playground).

Para automatizar el próximo chequeo por API haría falta: entrar a <https://console.cloud.google.com/apis/credentials?project=firm-streamer-501200-f5>, abrir el OAuth Client existente y agregar `https://developers.google.com/oauthplayground` como Authorized redirect URI — luego se puede regenerar un refresh_token vía el Playground con scope `webmasters.readonly`, y crear una rutina calcada a la del 2026-07-04. Mientras tanto, el recheck del 11 de julio queda como recordatorio manual.

## Productos propios publicados en la web (2026-07-01)

Se publicaron en el sitio los dos productos de software que Victor ya tenía desarrollados, hasta ahora sin presencia en la web corporativa:

| Producto | Proyecto de código | URL |
|---|---|---|
| Control de Licencias de Software | `Proyectos/license-control/` | `vpservices-it.com/productos/control-de-licencias` |
| Buscador de Parches y Vulnerabilidades en Servidores | `Proyectos/vulnapp/` | `vpservices-it.com/productos/buscador-de-vulnerabilidades` |

### Qué se agregó

- Nueva sección **"Productos"** en el home (`#productos`), con nav/footer actualizados, siguiendo el mismo patrón visual y de SEO que las páginas de servicio (meta tags únicos, canonical, OG/Twitter, JSON-LD `SoftwareApplication` + `BreadcrumbList` + `FAQPage`).
- Dos páginas de detalle nuevas bajo `public/productos/`, con la misma estructura que las de servicios (hero, "¿Qué incluye?", "¿Es para ti?", "Cómo trabajamos", FAQ, CTA final).
- `sitemap.xml` ampliado de 5 a 7 URLs.
- JSON-LD del home (`hasOfferCatalog`) ahora incluye los 2 productos junto a los 4 servicios (catálogo renombrado a "Servicios y Productos").
- CTA "Agendar demo" con precarga inteligente del formulario de contacto: el link `/?producto=<slug>#contacto` hace que `main.js` seleccione automáticamente el producto de interés y sugiera un mensaje inicial.

### Deploy, control de versiones y Search Console

- Commit `feat(productos): publicar Control de Licencias y Buscador de Vulnerabilidades` → PR [#55](https://github.com/vperezguzman66/vpservices-web/pull/55) → CI verde → mergeado a `main` (squash + rama eliminada).
- Deployado a producción (`npx wrangler deploy`) y verificado en vivo (200 en ambas páginas nuevas y en `sitemap.xml`).
- Sitemap reenviado manualmente en Search Console: pasó de 5 a **7 páginas descubiertas** el mismo día, status "Success".
- `README.md` del repo documentado con una sección "Productos propios" (URLs, mecanismo de precarga del CTA, referencia a PR #55).

## Auditoría de diseño con Impeccable (2026-07-02)

Se instaló [Impeccable](https://impeccable.style/) como skill de Claude Code (`.claude/skills/impeccable/`, en `.gitignore` — no se versiona) para detectar "AI slop" (clichés visuales de IA) y problemas reales de contraste/accesibilidad en `public/`. Se escribió `PRODUCT.md` en la raíz del repo (sí versionado) con el contexto estratégico de marca: registro "brand", audiencia (dueños de pyme), personalidad ("técnico y de confianza"), y anti-referencias explícitas (nada de plantilla genérica de SaaS con IA).

### Revisión completa (dual-agent)

Se corrió una revisión de diseño completa sobre el home: dos evaluaciones aisladas en paralelo —una revisión de diseño estilo director de arte (heurísticas de Nielsen, carga cognitiva, personas de usuario) y una de evidencia de detector/navegador (CLI + screenshots)— sintetizadas en un solo reporte. Puntaje inicial: **21/40** (banda "Aceptable", baja). Halló, entre otras cosas, que el sitio en vivo violaba las anti-referencias que el propio `PRODUCT.md` acababa de fijar (gradient-text, border-left de acento, eyebrow tags — exactamente lo que se pidió evitar).

### Hallazgos y qué se corrigió

Primera corrida del detector: 23 anti-patrones. Se corrigieron los que eran bugs reales o se pidió explícitamente resolver:

| Fix | Detalle | PR |
|---|---|---|
| Contraste 1.0:1 en `admin-leads.css` | `input, select, button` y una regla `button` aparte competían por `background`/`color` sobre el mismo elemento (igual especificidad) — el detector leía la regla equivocada como fondo real. `button` ahora tiene su propio bloque autocontenido. | [#57](https://github.com/vperezguzman66/vpservices-web/pull/57) |
| Contraste 2.7:1 en `.t-comment` (terminal animado del hero) | Color muy oscuro `#3d5a78` → `var(--text-muted)` (~7.5:1). | [#58](https://github.com/vperezguzman66/vpservices-web/pull/58) |
| Contraste 1.8:1 en `.featured-badge` ("Más solicitado") y `.founder-avatar` ("VP") | Texto blanco sobre gradiente cian→morado fallaba en el extremo cian. Ahora fondo cian sólido + texto oscuro, mismo patrón que el logo y el botón primario (~10.9:1). | [#58](https://github.com/vperezguzman66/vpservices-web/pull/58) |
| `icon-tile-stack` en las 6 cards (4 servicios + 2 productos) | El ícono en cuadrado redondeado sobre el título es "la plantilla universal de feature-card de IA". Se pidió explícitamente arreglarlo (a diferencia de las otras decisiones de marca, que se dejaron intactas en ese momento). Se reemplazó por ícono y título en línea, sin contenedor propio. | [#59](https://github.com/vperezguzman66/vpservices-web/pull/59) |
| **[P0]** Formulario sin validación | `novalidate` sin reemplazo en JS — un envío vacío llegaba a la red y mostraba un error genérico que sonaba a falla de servidor. Ahora `form.reportValidity()` bloquea el envío antes del `fetch()` y enfoca el primer campo inválido. | [#60](https://github.com/vperezguzman66/vpservices-web/pull/60) |
| **[P0]** Scroll-reveal podía dejar secciones invisibles para siempre | Las cards de servicios/tecnologías/valores partían en `opacity:0` sin ningún fallback si `main.js` fallaba (bloqueador de anuncios, error previo, JS deshabilitado), ni respetaba `prefers-reduced-motion`. Ahora son visibles por defecto; la animación es una mejora progresiva opcional. | [#60](https://github.com/vperezguzman66/vpservices-web/pull/60) |
| **[P0]** Colisión de texto en el hero móvil | `.hero-scroll-hint` se superponía con `.hero-stats` en viewports ~390-432px, tapando "TECNOLOGÍAS DOMINADAS" con "EXPLORAR". Oculto bajo `max-width:700px`. | [#60](https://github.com/vperezguzman66/vpservices-web/pull/60) |
| **[P1]** El sitio violaba su propio brief anti-slop | `PRODUCT.md` prohíbe gradient-text, border-left de acento y eyebrow tags sobre cada sección — el sitio en vivo tenía los tres. Corregido de punta a punta: **tipografía** Inter/Space Grotesk → IBM Plex Mono (títulos, refuerza el motivo de terminal ya presente) + IBM Plex Sans (cuerpo); **gradient-text** del H1 → color sólido (`.text-accent`); **border-left** de la cita del fundador → fondo cian tenue; **eyebrow tags** quitados en las 7 páginas (35 instancias); **stats duplicados** (hero vs. founder) → quitado el bloque del fundador; **bounce-easing** del chevron de scroll → easing exponencial suave con `prefers-reduced-motion`. | [#62](https://github.com/vperezguzman66/vpservices-web/pull/62) |
| **[P1]** Fallo silencioso en Turnstile | Si `/api/contact-config` fallaba, el script de Turnstile no cargaba, o faltaba el `sitekey`, el widget "Verificación anti-bot" quedaba vacío para siempre y el formulario se volvía imposible de enviar sin ninguna explicación. Ahora muestra un mensaje visible + botón "Reintentar", más un timeout de 8s de red de seguridad. De paso se corrigió la asociación de accesibilidad del label vía `aria-labelledby`. | [#63](https://github.com/vperezguzman66/vpservices-web/pull/63) |
| **[P2]** Avatar de iniciales en vez de foto real | Contradecía el principio "el experto es la marca" del propio `PRODUCT.md`. Reemplazado por una foto real de Víctor (`public/assets/social/founder-photo.jpg`) con anillo cian de marca. | [#65](https://github.com/vperezguzman66/vpservices-web/pull/65) |
| **[P2]** Jerga técnica sin traducir | Las cards de productos del home mostraban NVD, CISA KEV, EPSS, WinRM, SAM/ELP sin contexto para audiencia pyme no técnica. Reescritas a lenguaje benefit-first; las páginas de detalle conservan la explicación técnica completa. | [#65](https://github.com/vperezguzman66/vpservices-web/pull/65) |
| **[P2]** Accesibilidad del menú móvil | Drawer sin cierre por Escape/click-afuera/ícono X; hamburguesa de 32×24px (bajo el mínimo táctil de 44×44). Se agregó backdrop, cierre con Escape y click-afuera, animación de hamburguesa a X, y el botón ahora mide 44×44px. | [#65](https://github.com/vperezguzman66/vpservices-web/pull/65) |
| **[P2]** Select sin agrupar | "Servicio de interés" mezclaba 7 opciones de servicios y productos en una lista plana. Ahora usa `<optgroup>` para separar ambos grupos. | [#65](https://github.com/vperezguzman66/vpservices-web/pull/65) |

Deployados a producción tras cada merge (`npx wrangler deploy`), verificados visualmente en navegador y con pruebas funcionales en vivo (envío de formulario vacío, viewport móvil, fallo forzado de Turnstile, drawer móvil con backdrop/Escape/click-afuera).

Progreso del detector sobre `public/`: 23 → 22 (PR #57) → 19 (PR #58) → 13 (PR #59) → **7 anti-patrones** (estado actual, sin cambios tras el PR #65 — los fixes P2 eran de UX/contenido, no anti-patrones detectables por la herramienta).

### Falsos positivos investigados (sin cambios necesarios)

- `clipped-overflow-container` (`body { overflow-x: hidden }`): guard intencional contra scroll horizontal por los fondos decorativos full-bleed; nada necesita "escapar" de él hoy.
- `cramped-padding` ×2 en `<section class="hero">`: el detector confunde capas de fondo decorativas sin texto (`.hero-grid`, `.hero::before`) con contenido pegado al borde. El contenido real sí tiene inset propio.
- `overused-font`/`single-font` en `admin-leads.*`: panel interno (registro "product" en `PRODUCT.md`, no "brand"), fuera del alcance de esta auditoría.

Con el PR #65 se cerraron todos los P0, P1 y P2 identificados en la revisión dual-agent del 2026-07-02. Reporte completo guardado en `.impeccable/critique/` dentro del repo (no versionado).

## Ajuste manual de balance visual y color (2026-07-02, post-auditoría)

Retoque adicional de Victor sobre el mismo home, fuera del flujo de Impeccable. Mergeado a `main` vía PR [#66](https://github.com/vperezguzman66/vpservices-web/pull/66) y desplegado a producción.

- Links sociales (LinkedIn, Instagram, TikTok, X) agregados a la columna de contacto, para llenar el espacio vacío que quedaba junto al formulario.
- Texto del logo oculto en mobile (`<480px`) para evitar wrap junto al botón de menú.
- Colores morado (`--secondary`) y verde (`--accent`) de la paleta —ya definidos pero casi sin uso— aplicados a íconos de servicio, stats del hero, badges del terminal y el ícono de WhatsApp, en vez de repetir el mismo cian en todo.
