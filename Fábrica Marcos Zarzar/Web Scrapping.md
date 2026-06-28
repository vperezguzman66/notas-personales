
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