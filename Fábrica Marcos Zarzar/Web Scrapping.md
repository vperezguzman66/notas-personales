---
proyecto: "buscaprecios"
ruta: "Proyectos/buscaprecios"
cliente: "Marcos / Diego"
stack: "Python + FastAPI + frontend"
estado: "Desplegado en producción (buscaprecios-web) además de la versión local"
ultimo_cambio: 2026-07-08
---

## Estado (2026-07-08)

Se creó `Proyectos/buscaprecios-web`: reescritura completa a Cloudflare Workers + D1, desplegada en producción en **`buscaprecios.vpservices-it.com`**, con login propio (usuario/clave, sin Cloudflare Access). Replica el mismo patrón que `busca-medicamentos-web` (PBKDF2 + sesiones D1, rate limiting por Durable Object, caché KV, SSE progresivo). Los 4 scrapers (Easy, Homecenter, Construmart, Imperial) se portaron de Python/httpx a JavaScript/`fetch` manteniendo la misma lógica de parsing. No incluye MercadoLibre (requiere credenciales no configuradas) ni búsqueda en lote más allá de lo ya existente en la versión local (sí incluida). Usuario de producción creado: `dzarzar@zarzartex.com`. La versión local (`Proyectos/buscaprecios`, Python/FastAPI) sigue existiendo para desarrollo/pruebas. Repo: `github.com/vperezguzman66/buscaprecios-web` (privado).

De paso se corrigió un bug real en la versión local: al buscar, la página se quedaba pegada en el spinner indefinidamente pese a que el backend respondía bien — la causa fue un *service worker* huérfano de otro proyecto que también corrió alguna vez en `localhost:8000`, interceptando las peticiones. Se agregó además `AbortController` + timeout de 30s en el frontend para que una búsqueda colgada muestre error en vez de spinner infinito.

**Reset de clave (2026-07-08):** se generó una clave nueva para el usuario de producción `dzarzar@zarzartex.com` (UPDATE directo sobre D1 remoto vía `wrangler d1 execute --remote`, hash PBKDF2 con `hashPassword` de `src/auth.js`) y se guardó en 1Password. La clave en texto plano no se guarda en esta nota — está solo en 1Password.

Nota de costo: las consultas puntuales de administración sobre D1 (SELECT/UPDATE vía `wrangler d1 execute --remote`) no generan cargo — el free tier de D1 incluye 5M lecturas/día y 100K escrituras/día, muy por encima de este tipo de uso ocasional.

---

En reunión con [[Marcos - Diego]] se generó la necesidad de crear esta aplicación que permita extraer los precios de productos que existen en la competencia, para ello se genero una app que está en el repositorio en GitHub con el link https://github.com/vperezguzman66/buscaprecios.git

Aplicación web para comparar precios de productos en las principales tiendas de mejoramiento del hogar de Chile: **Easy**, **Homecenter (Sodimac)**, **Construmart** y **MercadoLibre**.

## Características

[](https://github.com/vperezguzman66/buscaprecios#caracter%C3%ADsticas)

- **Búsqueda simple** — escribe un producto y obtén resultados de todas las tiendas en paralelo, ordenados de menor a mayor precio
- **Búsqueda en lote (CSV)** — sube un archivo con hasta 30 productos; los resultados llegan en tiempo real con barra de progreso
- **Exportar a CSV** — descarga los resultados en formato compatible con Excel (con BOM para acentos)
- **Filtro por tienda** — selecciona en qué tiendas buscar
- **Historial de búsquedas** — las últimas 10 búsquedas se guardan en el navegador como accesos rápidos
- **Caché de resultados** — búsquedas repetidas son instantáneas (caché en memoria de 5 minutos)
- Sin Playwright ni Selenium: todo con `httpx` puro (rápido y liviano)

## Requisitos

[](https://github.com/vperezguzman66/buscaprecios#requisitos)

- Python 3.11 o superior
- Conexión a internet


Bug fix: precios sin mostrar en Homecenter

Problema: Los productos de Homecenter/Sodimac aparecían todos con "Sin precio" al buscar.

Causa: La función _best_price en homecenter.py solo buscaba los tipos internetPrice y cmrPrice, pero Sodimac devuelve eventPrice (precio oferta activo) y normalPrice (precio tachado). Como ninguno de los dos tipos hardcodeados aparecía, siempre retornaba None.

Solución: Se reemplazó la búsqueda hardcodeada por una lista de prioridad ordenada:
_PRICE_PRIORITY = ["eventPrice", "internetPrice", "offerPrice", "cmrPrice", "normalPrice"]

---
Nueva funcionalidad: vista tabla con desviación del promedio

Qué se hizo:
- Se reemplazó la grilla de tarjetas de la búsqueda simple por una tabla con columnas: Tienda, Producto, Precio, Desviación del promedio.
- Se calculó la desviación como ((precio - promedio) / promedio) × 100 sobre todos los resultados con precio válido.
- Badges de color: verde para productos más baratos que el promedio, rojo para los más caros, gris para los que están cerca del promedio (< 0.5%).
- La columna de desviación también se incluyó en el CSV exportado.

---
Se agregó **Imperial.cl** como quinta tienda (Oracle Commerce Cloud). También se empaquetó la app como ejecutable Windows standalone (PyInstaller) para que Marcos/Diego la usen sin instalar Python.

---
Bug fix (2026-07-03): Imperial devolvía 0 resultados en búsquedas genéricas

Problema: búsquedas como "taladro" (que Imperial redirige a una página de categoría en vez de resultados directos) siempre devolvían 0 productos, sin error visible.

Causa: en `imperial.py`, `_from_category` recibía del endpoint de categoría una lista de objetos `{"id": "..."}`, pero el código hacía `str(item)` sobre el dict completo en vez de extraer `item["id"]`. Los IDs de producto quedaban corruptos (`"{'id': '136859'}"`) y la consulta posterior a `/ccstore/v1/products` no encontraba nada.

Solución: extraer `item["id"]` correctamente antes de convertir a string.

Bug fix relacionado: precio faltante en resultados de categoría

Problema: incluso con los IDs corregidos, los productos de búsquedas por categoría aparecían siempre como "Sin precio".

Causa: ese flujo no pasa por `sku.activePrice` (solo lo hace el flujo de resultados directos); el endpoint `/ccstore/v1/products` no traía campos de precio en el request.

Solución: se agregaron los campos `listPrice,salePrice,childSKUs.listPrice,childSKUs.salePrice` al request y se usa como fallback cuando no hay precio del flujo de records directos. Verificado con búsquedas reales: pasó de 0/5 a 3/5 productos con precio (los 2 restantes probablemente sin SKU con stock activo).

---
Hardening posterior (code review, mismo día): se corrieron 8 ángulos de revisión + verificación sobre el fix de Imperial y salieron 3 problemas reales adicionales, ya corregidos:

1. El fallback de precio se disparaba también en el flujo de resultados directos (no solo en el de categoría), pudiendo pisar un "Sin precio" legítimo de `sku.activePrice` con un precio de otro endpoint. Fix: distinguir "el flujo nunca tuvo precios" (`pid not in prices`) de "este registro no tiene precio" (`pid` presente con valor `None`).
2. `_extract_product_price` no capturaba errores al convertir el precio a `float`, a diferencia de `_parse_price`; un valor no numérico en un solo producto podía tumbar toda la búsqueda de Imperial. Fix: delegar en `_parse_price` (que ya tiene el try/except), eliminando además la duplicación de lógica.
3. La extracción de IDs de categoría asumía que todos los elementos eran dicts; si el endpoint alguna vez devuelve otra forma, lanzaba `TypeError` sin capturar. Fix: filtrar con `isinstance(item, dict)` antes de acceder a `item["id"]`.

Todo probado contra la API real de Imperial (sin regresión) y pusheado a GitHub (`24f36ef`).