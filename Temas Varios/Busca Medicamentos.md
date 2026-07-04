---
proyecto: "busca-medicamentos"
ruta: "Proyectos/busca-medicamentos"
cliente: "Propio"
stack: "Python + FastAPI + httpx + Playwright"
estado: "Activo — resultados progresivos SSE, promociones"
ultimo_cambio: 2026-07-03
---

Comparador de precios de medicamentos en las farmacias de cadena de Chile: **SalcoBrand**, **Cruz Verde**, **Farmacias Ahumada** y **Doctor Simi**.

Repo en GitHub: https://github.com/vperezguzman66/busca-medicamentos

**Esta es la versión local (Python/FastAPI).** También existe una versión reescrita en Cloudflare Workers, en producción con login propio en `medicamentos.vpservices-it.com` — ver [[Busca Medicamentos Web]].

[[Varios]]

## Por qué existe

Mismo problema que resuelve [[Web Scrapping]] (buscaprecios) para ferretería, pero para medicamentos. Proyecto propio, no vinculado a un cliente.

## Arquitectura — investigación previa a construir

Antes de escribir código se investigó en vivo la estructura real de cada sitio (con `curl` y Playwright) para decidir si servía `httpx` puro o hacía falta navegador. Resultado: arquitectura mixta.

| Farmacia | Plataforma | Acceso | Notas |
|---|---|---|---|
| SalcoBrand | Spree Commerce | API pública de Algolia (`httpx`) | App ID/API key están embebidos en el JS público del propio sitio, restringidos por `Referer` |
| Doctor Simi | VTEX (cuenta `farmaciasdeldrsimicl`) | API legacy `catalog_system/pub/products/search` (`httpx`) | Dominio correcto es `www.drsimi.cl`, no `farmaciasdrsimi.cl` (no resuelve) |
| Farmacias Ahumada | Salesforce Commerce Cloud | HTML server-rendered (`httpx` + BeautifulSoup) | El precio vigente no viene en atributo limpio, solo como texto `$X.XXX` dentro de `span.sales` |
| Cruz Verde | Angular SPA + Incapsula (WAF) | **Playwright** | La API real (`api.cruzverde.cl`) exige sesión vía un OAuth propio (Andes ML/Keycloak) — se probó replicar con `httpx` reusando cookies de Incapsula y igual da 401; no vale la pena perseguirlo. Renderiza bien con Chromium headless. |

Cruz Verde corre con **un solo browser Chromium compartido** a nivel de app (arrancado en el `lifespan` de FastAPI), abriendo un `context` liviano por búsqueda — no un browser nuevo por request, para no matar el rendimiento del servicio.

## Mejora aplicada desde el día 1

`base.py` incluye `format_clp_price()` compartido por los 4 scrapers — evita la duplicación de lógica de formato de precio que se detectó (y se tuvo que corregir después) en `buscaprecios` durante un code review.

## Estado

🆕 Initial release (2026-07-03). Probado de punta a punta: los 4 scrapers individualmente, búsqueda combinada en paralelo (confirmado que Cruz Verde no bloquea a los demás), y búsqueda en lote CSV vía SSE.

### Bug encontrado al probar el frontend en el navegador (mismo día)

Problema: "Paracetamol 1G Dropol ... Tubo x 20" aparecía en **$769.930** en vez de ~$7.699 — desviaciones del promedio absurdas (-98%, +2542%) delataron el error.

Causa: en `ahumada.py`, `_extract_price` limpiaba todos los dígitos del texto completo de `span.sales`. Algunos productos tienen, dentro del mismo `span.sales`, un segundo bloque con un badge de descuento "-XX%" sin relación al precio (ej. "-30%"). Sus dígitos se concatenaban con los del precio real: "$7.699" + "-30%" → "769930".

Solución: extraer solo el primer monto `$...` con regex en vez de limpiar dígitos de todo el bloque. Verificado tanto por API como visualmente en el navegador — el producto ahora muestra $7.699 correctamente. Pusheado (`588d4c2`).

### Bug de confiabilidad en Cruz Verde (mismo día, probando "colmibe 40/10")

Problema: la primera búsqueda de "colmibe 40/10" no encontró el producto en Cruz Verde (sí existe ahí); un reintento por API sí lo encontró.

Causa: `cruzverde.py` esperaba un timeout fijo de 5s tras apretar Enter en el buscador, antes de leer las tarjetas de producto — no siempre alcanzaba a que Angular terminara de renderizar la grilla.

Solución: esperar explícitamente a que la primera tarjeta aparezca en el DOM (máx. 15s) en vez de un timeout fijo, sin fallar si genuinamente no hay resultados. Verificado con 3 búsquedas distintas — encontró resultados de forma consistente las 3 veces. Pusheado (`b538f25`).

### Feature: promociones y descuentos (mismo día)

Las 4 tiendas ya exponían el precio normal/de lista junto al precio con descuento, pero solo se usaba el precio final. Se agregó `original_price`/`original_price_text` al modelo `Product`, con un helper compartido `resolve_discount()` en `base.py` (solo conserva el precio original si es mayor al actual). Se muestra como badge "-XX%" + precio tachado en la tabla, tarjetas de la vista en lote, y columnas nuevas en los exports CSV. Verificado en vivo con "paracetamol" — SalcoBrand y Ahumada mostraron descuentos reales (ej. $731 antes $1.309, -44%). Pusheado (`b00da15`).

### Feature: resultados progresivos vía SSE (mismo día)

Cruz Verde (Playwright) es ~10x más lenta que las otras 3 tiendas (httpx). `/api/search` esperaba a que las 4 terminaran (`asyncio.gather`) antes de responder — toda búsqueda tardaba ~13s aunque 3 de 4 farmacias ya tuvieran resultado.

Solución: `/api/search` pasa de JSON síncrono a **Server-Sent Events**, usando `asyncio.as_completed()` para emitir un snapshot acumulado (ordenado) cada vez que una farmacia termina, con la lista de farmacias aún pendientes. El frontend muestra un banner "Buscando aún en: X" y re-renderiza la tabla en cada evento. `_do_search` (bloqueante) se mantiene intacta para `/api/search-batch`, que no necesita este cambio.

Verificado: los primeros 3 resultados llegan en ~1s (antes ~13s para cualquier resultado), y confirmé visualmente en el navegador que Cruz Verde completa el resto sin bloquear la vista inicial. Pusheado (`2cc8b4d`).

### Bug fix (mismo día): SalcoBrand no encontraba dosis combinadas ("colmibe 40/10")

Victor reportó que "colmibe 40/10" no aparecía en SalcoBrand (sí en Cruz Verde y Ahumada, y el producto sí existe en el catálogo de SalcoBrand). Probando directo contra la API de Algolia: **cualquier query con "/" devuelve 0 resultados**, sin importar el resto del texto — confirmado con 5 variantes distintas.

Solución: normalizar patrones `N/M` (con o sin "mg") a `Nmg Mmg` antes de mandar la query, solo en `salcobrand.py`. De paso se agregó `quote()` a la query (se enviaba sin codificar URL, mismo tipo de bug latente con "&" o "+"). Verificado: "colmibe 40/10" pasó de 0 a 3 resultados (las 3 dosis de Colmibe), confirmado por API y en el navegador. Pusheado (`fb8b965`).

Nota: esta es una limitación específica de dosis combinadas con "/", común en medicamentos cardiovasculares (losartán/hidroclorotiazida, etc.) — vale la pena tenerla presente si aparecen más reportes similares en otras tiendas.

### Bug fix grave (mismo día): Doctor Simi bloqueaba TODA búsqueda de 2+ palabras

Al revisar si Ahumada/Cruz Verde/Doctor Simi tenían el mismo problema de "/" que SalcoBrand, encontré algo mucho más importante: el endpoint `?ft={term}` de Doctor Simi (VTEX) responde HTTP 400 "Bad Request! Scripts are not allowed!" (un WAF) para **cualquier término de más de una palabra**, con o sin barra, con o sin números. Confirmado con "paracetamol infantil", "ibuprofeno 400mg", etc. — todas fallaban.

Esto significaba que **cualquier búsqueda multi-palabra devolvía 0 resultados de Doctor Simi de forma silenciosa** desde el día 1 (el `except Exception: return []` se comía el error HTTP sin dejar rastro) — un problema bastante más grave que el de SalcoBrand, porque afectaba prácticamente cualquier búsqueda real con nombre + dosis.

Nota lateral: mientras investigaba esto, hice suficientes requests directos contra la API de Doctor Simi como para gatillar su propio rate-limiting (429) y bloqueos temporales — hay que ser cuidadoso probando esta tienda en particular, con pausas entre requests.

Solución: usar la variante de VTEX con el término como segmento de la URL (`/search/{term}?map=ft`) en vez de como query param (`?ft={term}`) — el mismo WAF no bloquea esta forma. Verificado con nuestro scraper real: "paracetamol infantil" e "ibuprofeno 400mg" pasaron de 0 a resultados reales, sin romper las búsquedas de una palabra. Pusheado (`2f32c53`).
