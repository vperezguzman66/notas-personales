
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