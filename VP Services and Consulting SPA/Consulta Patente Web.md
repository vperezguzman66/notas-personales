---
proyecto: "consulta-patente-web"
ruta: "Proyectos/consulta-patente-web"
cliente: "Propio"
stack: "Cloudflare Workers + D1 + Durable Object + Turnstile + Boostr API"
estado: "Repo publicado en GitHub (privado). Code review de seguridad hecho (XSS en admin corregido + 7 hallazgos adicionales cerrados). ADMIN_TOKEN y Turnstile (modo de prueba) configurados. Bloqueado en Boostr: la API key existente es inválida (403) y Victor no quiere pagar todavía — evaluando alternativas, ninguna mejor encontrada hasta ahora. Falta Workers Paid antes de producción"
ultimo_cambio: 2026-07-18
---

[[Varios]]

Herramienta pública: el usuario ingresa una patente chilena y recibe por correo un reporte con los datos básicos del vehículo (marca, modelo, año, color, combustible). Fuente de datos: [Boostr API](https://boostr.cl/patente) ($50 CLP por consulta nueva, cacheada en D1 para no pagar dos veces por la misma patente). Repo: `github.com/vperezguzman66/consulta-patente-web` (privado).

## Origen de la idea y decisión de negocio clave

Idea inicial: capturar los datos de cada consulta (incluyendo nombre y RUT del dueño del vehículo, que muchas veces no es quien hace la consulta) para armar una base de personas y venderla, más publicidad. Al revisar el marco legal chileno se descartó explícitamente esa parte del modelo:

- **Ley 19.628** (vigente) y sobre todo **Ley 21.719** — reforma que entra en plena vigencia el 1 de diciembre de 2026, crea la agencia fiscalizadora APDP con multas de hasta 20.000 UTM (~$1.400 millones CLP) o 4% de ventas anuales. Vender datos personales de terceros sin su consentimiento es exactamente el tipo de caso que la ley sanciona.
- Los TyC de Boostr no autorizan explícitamente la reventa de los datos obtenidos vía su API; dejan toda la responsabilidad legal en quien la usa.

**Modelo final**: lead-gen legítimo. El usuario deja su email para recibir su propio reporte (consentimiento propio, igual que hacen autofact.cl, carvuk.com, patentechile.com). Se construye una lista de usuarios que se suscribieron voluntariamente — no una base de terceros para reventa. Monetización vía publicidad y reportes premium a futuro, no venta de datos.

## Alcance por fases

1. **Fase 1 (scaffold actual)**: datos básicos del vehículo.
2. Fase 2: estado legal/documental (permiso de circulación, revisión técnica, SOAP, prendas/embargos).
3. Fase 3: multas e infracciones.
4. Fase 4: historial de propietarios.

Boostr devuelve todo en una sola respuesta por el mismo costo, así que se cachea el JSON crudo completo en D1 desde la Fase 1 — las fases siguientes solo agregan UI/email que muestre más campos del mismo dato ya cacheado, sin llamadas nuevas a la API. Cada fase que exponga datos del propietario (nombre, RUT) debe revisarse contra la Ley 21.719 antes de mostrarse.

## Arquitectura

Reutiliza patrones ya probados de los proyectos hermanos:
- **De `vpservices-web`**: verificación Turnstile, rate-limiting por Durable Object (sliding window), envío de correo vía binding `EMAIL` (Cloudflare Email Service).
- **De `buscaprecios-web`/`polizas-web`**: caché D1 con expiración, cliente HTTP externo simple (fetch + throw en `!ok`), scripts de build/test/deploy, mock de D1 sobre `node:sqlite` en los tests.

Se omite a propósito el módulo de sesiones de usuario (`auth.js`) — es una herramienta pública de una sola consulta, no una app multiusuario — y el patrón SSE (una sola llamada a una sola fuente, no hay fan-out que progresar).

Admin (`GET /admin`, `GET /api/leads`) protegido por un único secret compartido (`ADMIN_TOKEN`, bearer token) en vez de Cloudflare Access — más simple para un solo administrador.

### Modelo de datos D1

```
lookups   (patente PK, data_json, fetched_at, expires_at)   -- caché de la respuesta cruda de Boostr, TTL 30 días
leads     (id, created_at, ip, email, patente, channel, delivery_status)  -- auditoría de envíos, única "base" que se conserva
```

## Costos

Workers, D1, el Durable Object (backend SQLite) y Turnstile caben en el **plan free** de Cloudflare para el volumen esperado. La excepción es el envío de correo: `mailer.js` manda el reporte al email que el usuario escribe (destinatario arbitrario, no verificado de antemano), y Cloudflare exige el **plan Workers Paid (US$5/mes mínimo)** para eso — incluye 3.000 correos/mes, luego US$0.35 cada 1.000 adicionales. Se evaluó migrar a un proveedor externo (Resend, gratis hasta 3.000/mes sin el cargo fijo) pero Victor decidió explícitamente mantener `env.EMAIL` y pagar los US$5/mes por simplicidad.

Costo variable real fuera de Cloudflare: **Boostr, $50 CLP por consulta nueva no cacheada**. No existe alternativa oficial gratuita equivalente: el SII solo tasa si ya conoces marca/modelo/año (no funciona patente → marca), y el Certificado de Anotaciones Vigentes del Registro Civil cuesta ~$1.430–1.560 CLP, más caro que Boostr. Sitios "gratis" como Carvuk/AutoRiesgo/PatentesChile no exponen API — solo consulta manual, integrarlos implicaría scrapear (mismo riesgo legal que ya se descartó para el propio negocio).

**Re-evaluado 2026-07-18** (Victor no quiere pagar Boostr todavía): se buscó de nuevo si había surgido algo mejor. Apareció **GetAPI** (`getapi.cl/patente`, mismos campos básicos) pero su plan más barato con volumen recurrente es $7.000 CLP/mes + IVA por 10.000 consultas/día — pensado para alto volumen, más caro que Boostr para los ~100 patentes nuevas/mes que se estiman acá (Boostr cobra por consulta real, sin mínimo mensual). Solo ofrece una demo de 50 consultas únicas en 24h, no un free tier permanente. **Autodata.cl** parece ser solo un buscador manual (como Carvuk), sin API pública clara. Conclusión: Boostr sigue siendo la opción más barata para el volumen esperado; no se encontró alternativa mejor.

### Planes de Boostr (referencia julio 2026)

| Plan | Precio | Consultas | Duración |
|---|---|---|---|
| Free | Gratis | 5/día | Solo 1 mes (prueba, no permanente) |
| Créditos | $50 CLP c/u | — | Mínimo $10.000 (200 consultas), no expiran |
| Starter | $5.000 CLP/mes | 50/día (~1.500/mes) | Ilimitado en el tiempo |
| Pro | $20.000 CLP/mes | 100/día | Ilimitado |
| Full | $90.000 CLP/mes | 300/día | Ilimitado |

Como cada patente se cachea 30 días, el costo real depende solo de patentes *nuevas* por mes: bajo ~100/mes conviene la modalidad de créditos, sobre eso el plan Starter sale más barato. Plan recomendado para partir: **Free**, para validar demanda a costo $0 antes de comprometerse a un plan pago.

## Estado (2026-07-18)

Scaffold completo: backend, frontend (formulario + panel admin), 50 tests en verde (`node --test`, sin red, D1 mockeado con `node:sqlite`), build verificado (`esbuild` minifica y hashea `public/` → `dist/`). Frontend revisado visualmente en preview (formulario y admin renderizan bien).

Desde la revisión anterior: se crearon y migraron las D1 reales de producción (`consulta-patente-db`) y staging (`consulta-patente-db-staging`) — `database_id` ya no es placeholder en `wrangler.jsonc`, y se agregó el route de producción `patente.vpservices-it.com`. Se hizo un deploy de prueba a staging, que encontró un bug real: el router reescribía `/admin` a `/admin.html`, pero Cloudflare Assets ya sirve `admin.html` directo en `/admin` (html_handling por defecto), así que la reescritura causaba un redirect 307 a sí mismo. Corregido.

**Repo publicado** en `github.com/vperezguzman66/consulta-patente-web` (privado).

**Revisión de seguridad (2026-07-18)**: se encontró y corrigió un **XSS almacenado** real en el panel admin — `admin.js` renderizaba `lead.email` vía `tr.innerHTML` sin escapar, y el regex de validación de email del backend aceptaba `<`/`>`/comillas, así que un atacante podía guardar un payload como "email" de una consulta legítima y ejecutarlo en la sesión del admin al abrir `/admin` (robando el `ADMIN_TOKEN` de `sessionStorage`). Corregido con construcción DOM + `textContent`, y se endureció el regex de email en el backend para cerrar la causa raíz (no solo el renderer). Un `/code-review` de seguimiento sobre esos mismos fixes encontró 7 hallazgos más (todos corregidos): CSP que no se aplicaba al 500 de emergencia, un 500 que quedaba cacheable 5 minutos, el try/catch de seguridad duplicado en vez de centralizado (ahora vive en `handleRequest`), los OPTIONS de `/api/consulta`/`/api/leads` sin headers de seguridad, URL parseada dos veces por request, y `admin.js` sin manejo de errores en `loadLeads`.

**Secretos configurados** (prod + staging, vía `wrangler secret put`):
- ✅ `ADMIN_TOKEN` — generado nuevo (uno distinto por entorno, guardado en el gestor de contraseñas de Victor, no en el repo).
- ✅ `TURNSTILE_SECRET_KEY` / `TURNSTILE_SITE_KEY` — configurados en **modo de prueba** con los sitekeys dummy que documenta Cloudflare (`1x00000000000000000000AA` / secret correspondiente, visible, "always passes"). Sirve para validar el flujo end-to-end sin tener aún un sitio de Turnstile real — hay que reemplazarlos por un widget real antes de producción de verdad.
- ⚠️ `BOOSTR_API_KEY` — **ya existía configurada de antes**, pero es inválida: Boostr responde `403 INVALID_API_KEY`. Se intentó probar con el endpoint de prueba sin key de Boostr (`/vehicle/fake/{patente}.json`, documentado en `docs.boostr.cl/reference/car-plate-testing`), pero el WAF de Cloudflare de Boostr bloquea ese endpoint como tráfico bot incluso viniendo de otro Worker de Cloudflare — no es viable probarlo así. Bloqueado en que Victor no quiere pagar todavía (ver sección de Costos, re-evaluación de alternativas del 2026-07-18: ninguna mejor que Boostr).

**Deploy**: verificado en staging (`consulta-patente-web-staging.vperezguzman.workers.dev`) — `/api/health`, `/api/config`, `/admin`, `/`, headers de seguridad y auth de `/api/leads` funcionan correctamente. `/api/consulta` no se pudo probar de punta a punta por la key de Boostr inválida.

Pendiente antes del primer deploy a producción:
- ✅ ~~Crear las D1~~ — hecho, prod y staging migradas.
- ✅ ~~Definir `ADMIN_TOKEN`~~ — hecho.
- ✅ ~~Turnstile~~ — hecho en modo de prueba; falta el sitio real antes de ir a producción de verdad.
- ⏳ Conseguir una `BOOSTR_API_KEY` válida (la actual da 403). Pendiente de que Victor decida pagar o siga buscando alternativas.
- ⏳ Activar el plan Workers Paid (US$5/mes) en la cuenta de Cloudflare.
