---
proyecto: "consulta-patente-web"
ruta: "Proyectos/consulta-patente-web"
cliente: "Propio"
stack: "Cloudflare Workers + D1 + Durable Object + Turnstile + Boostr API"
estado: "Scaffold completo, commit inicial hecho — sin desplegar (D1/Boostr/Turnstile/Workers Paid pendientes de aprovisionar)"
ultimo_cambio: 2026-07-12
---

[[Varios]]

Herramienta pública: el usuario ingresa una patente chilena y recibe por correo un reporte con los datos básicos del vehículo (marca, modelo, año, color, combustible). Fuente de datos: [Boostr API](https://boostr.cl/patente) ($50 CLP por consulta nueva, cacheada en D1 para no pagar dos veces por la misma patente). No hay repo público todavía — se scaffoldeó localmente, sin publicar en GitHub.

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

### Planes de Boostr (referencia julio 2026)

| Plan | Precio | Consultas | Duración |
|---|---|---|---|
| Free | Gratis | 5/día | Solo 1 mes (prueba, no permanente) |
| Créditos | $50 CLP c/u | — | Mínimo $10.000 (200 consultas), no expiran |
| Starter | $5.000 CLP/mes | 50/día (~1.500/mes) | Ilimitado en el tiempo |
| Pro | $20.000 CLP/mes | 100/día | Ilimitado |
| Full | $90.000 CLP/mes | 300/día | Ilimitado |

Como cada patente se cachea 30 días, el costo real depende solo de patentes *nuevas* por mes: bajo ~100/mes conviene la modalidad de créditos, sobre eso el plan Starter sale más barato. Plan recomendado para partir: **Free**, para validar demanda a costo $0 antes de comprometerse a un plan pago.

## Estado (2026-07-12)

Scaffold completo: backend, frontend (formulario + panel admin), 45 tests en verde (`node --test`, sin red, D1 mockeado con `node:sqlite`), build verificado (`esbuild` minifica y hashea `public/` → `dist/`). Frontend revisado visualmente en preview (formulario y admin renderizan bien).

Commit inicial hecho en el repo git local (`feat: scaffold consulta-patente-web`), sin publicar en GitHub. Pendiente antes del primer deploy real:
- ⏳ Crear las D1 (`wrangler d1 create`) y actualizar `database_id` en `wrangler.jsonc`.
- ⏳ Contratar créditos en Boostr y configurar `BOOSTR_API_KEY`.
- ⏳ Crear sitio de Turnstile en Cloudflare y configurar sus keys.
- ⏳ Definir `ADMIN_TOKEN` y el dominio/subdominio de producción.
- ⏳ Activar el plan Workers Paid (US$5/mes) en la cuenta de Cloudflare.
