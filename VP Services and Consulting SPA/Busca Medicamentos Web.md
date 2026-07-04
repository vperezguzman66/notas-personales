Versión en producción de "Busca Medicamentos" (`Proyectos/busca-medicamentos`, la app Python local) — reescrita completa a Cloudflare Workers para publicarla en `medicamentos.vpservices-it.com`, con login propio restringido a usuarios permitidos.

Repo en GitHub (privado): https://github.com/vperezguzman66/busca-medicamentos-web

[[Página Web]]

## Por qué se reescribió en vez de solo "subirla"

La versión Python usa Playwright para el scraper de Cruz Verde, y Cloudflare Workers no corre Python ni Playwright nativo. Se evaluaron 3 caminos (VPS, Mac + Cloudflare Tunnel, reescribir a Workers) y Victor eligió reescribir todo a Workers para que quedara en la misma arquitectura que `vpservices-web`, con:

- **Login propio** (usuario/clave) en D1, en vez de Cloudflare Access — hash PBKDF2-SHA256 vía `crypto.subtle`, cookie de sesión `httpOnly`.
- **Cruz Verde vía Cloudflare Browser Rendering** (`@cloudflare/playwright`) — la única forma de correr un Chromium real dentro de Workers. Funcionó a la primera con solo adaptar la sintaxis del Playwright de Python.
- Los otros 3 scrapers (SalcoBrand, Ahumada, Doctor Simi) pasaron de `httpx` a `fetch()`, reusando toda la lógica de negocio ya validada (los fixes de la barra "/" en SalcoBrand y del endpoint de Doctor Simi se portaron tal cual).

## Bloqueo de infraestructura encontrado en el camino

El Mac de Victor (MacBook Air 2015) quedó fuera del soporte oficial de Apple en macOS Monterey — Apple no ofrece actualización para ese hardware. Esto significa:
- `wrangler dev` no puede correr ahí, ni siquiera con `--remote` — el runtime `workerd` requiere macOS 13.5+.

- Colima/Docker tampoco es una alternativa viable: Homebrew ya no tiene binarios de `qemu` para Monterey (ni siquiera para compilar desde código, por una receta rota de una dependencia).
- Arreglar esto de fondo requeriría OpenCore Legacy Patcher (parche de la comunidad para instalar macOS más nuevo en hardware no soportado) — Victor decidió no tocar el sistema operativo.

**Solución adoptada:** iterar con deploys reales a un Worker de staging (`env.staging` en `wrangler.jsonc`, mismo patrón que ya usa `vpservices-web`) en vez de un entorno de desarrollo local. Cada cambio se prueba con `npm run deploy:staging` contra una URL real de Cloudflare.

## Cuota de Browser Rendering — limitación real a tener presente

Plan gratis: 10 min de browser al día, 3 sesiones concurrentes, 6 requests/minuto. Cada búsqueda de Cruz Verde usa ~15-20s, así que el free tier alcanza para ~30-40 búsquedas/día. Si el uso crece, hay que pasar al plan pago de Workers.

## Estado

✅ Desplegado en producción (2026-07-03). Verificado en el navegador real contra `medicamentos.vpservices-it.com`: login con la cuenta de Victor (`victorperez@vpservices-it.com`), búsqueda simple con las 4 farmacias (incluye Cruz Verde), descuentos mostrados correctamente, logout, y búsqueda en lote vía CSV (confirmada por Victor directamente en el navegador — yo solo pude validarla por API porque la herramienta de subida de archivos quedó bloqueada en esa sesión).

Durante la reescritura aparecieron 2 bugs nuevos (no existían en la versión Python, porque BeautifulSoup los resolvía automáticamente):
1. Entidades HTML sin decodificar (`&oacute;`, `&amp;` en nombres y URLs de imagen de Ahumada) — arreglado con un decoder propio.
2. Una regex de imagen que agarraba el ícono de un badge promocional en vez de la foto real del producto, porque el orden de atributos HTML variaba — arreglado separando la búsqueda en dos regex en vez de una alternancia `A|B` (el fallback ganaba antes de que el motor buscara más adelante).

Usuarios permitidos se agregan a mano con `scripts/create_user.mjs` (no hay registro público).
