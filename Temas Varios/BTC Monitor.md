---
proyecto: "btc-monitor"
ruta: "Proyectos/btc-monitor"
cliente: "Propio"
stack: "React + Vite"
estado: "Funcional — Stop Loss / Take Profit simulados por posición"
ultimo_cambio: 2026-06-19
---

[[Varios]]
# Crypto Monitor ₿

[](https://github.com/vperezguzman66/BTC-MONITOR#crypto-monitor-)

App web para monitorear, analizar e invertir en criptomonedas en tiempo real. Construida con **React 18 + Vite**, usando la API pública gratuita de [CoinGecko](https://www.coingecko.com/api) (sin API key).

## Funcionalidades

[](https://github.com/vperezguzman66/BTC-MONITOR#funcionalidades)

### Mercado en vivo

[](https://github.com/vperezguzman66/BTC-MONITOR#mercado-en-vivo)

- Precio actualizado cada 30 segundos con variación 24h
- Gráfico histórico con rangos 24h / 7d / 30d / 90d
- 25 criptomonedas disponibles (BTC, ETH, SOL, BNB, XRP, DOGE, ADA, AVAX, y más)
- 4 divisas: CLP, USD, EUR, MXN
- Selector personalizable: elige qué monedas aparecen en tu dropdown

### Indicadores técnicos (sobre el gráfico)

[](https://github.com/vperezguzman66/BTC-MONITOR#indicadores-t%C3%A9cnicos-sobre-el-gr%C3%A1fico)

- **MA 50** — Media móvil de 50 períodos (línea amarilla)
- **MA 200** — Media móvil de 200 períodos (línea morada)
- Detección automática de **Golden Cross** (MA50 > MA200, señal alcista) y **Death Cross** (señal bajista)

### Señal de trading automática

[](https://github.com/vperezguzman66/BTC-MONITOR#se%C3%B1al-de-trading-autom%C3%A1tica)

Analiza 365 días de datos diarios y combina 4 indicadores para emitir una señal **COMPRAR / MANTENER / VENDER** con fuerza porcentual:

|Indicador|Peso|Señal de compra|Señal de venta|
|---|---|---|---|
|RSI (14)|3 pts|RSI < 30 (sobreventa)|RSI > 70 (sobrecompra)|
|MACD (12,26,9)|3 pts|Cruce alcista / MACD > señal|Cruce bajista / MACD < señal|
|Bollinger (20,2σ)|2 pts|Precio cerca de banda inferior|Precio cerca de banda superior|
|MA 50 / MA 200|2 pts|Golden Cross + precio > MA50|Death Cross + precio < MA50|

Incluye gauge visual del RSI, barra de consenso y explicación textual de cada indicador.

### Portafolio con P&L en vivo

[](https://github.com/vperezguzman66/BTC-MONITOR#portafolio-con-pl-en-vivo)

- Registra tus compras: cantidad, precio de compra y fecha
- Calcula en tiempo real la ganancia o pérdida en pesos/dólares y en porcentaje
- Total invertido, valor actual y cantidad acumulada por moneda
- Los datos se guardan en `localStorage` (sin cuenta, sin servidor)

### Calculadora DCA (Dollar Cost Averaging)

[](https://github.com/vperezguzman66/BTC-MONITOR#calculadora-dca-dollar-cost-averaging)

Simula qué habría pasado si hubieras invertido una cantidad fija de forma periódica:

- Montos y frecuencias configurables: diario, semanal, quincenal, mensual
- Períodos históricos: 3, 6 o 12 meses de datos reales
- Muestra: total invertido, monedas acumuladas, precio promedio de entrada y ROI

### Alertas de precio

[](https://github.com/vperezguzman66/BTC-MONITOR#alertas-de-precio)

- Avísame cuando una cripto suba o baje de un umbral
- Notificación push del navegador al dispararse
- Persisten en `localStorage` entre sesiones

### Guía de indicadores

[](https://github.com/vperezguzman66/BTC-MONITOR#gu%C3%ADa-de-indicadores)

Panel educativo integrado con 8 secciones desplegables que explican, en español y sin jerga financiera, cómo leer cada herramienta: gráficos, MAs, RSI, MACD, Bollinger, señal compuesta, DCA y gestión del riesgo.

---

## Instalación y uso

[](https://github.com/vperezguzman66/BTC-MONITOR#instalaci%C3%B3n-y-uso)

```shell
npm install
npm run dev      # servidor de desarrollo → http://localhost:5173
npm run build    # build de producción en dist/
npm run preview  # previsualizar el build local
```

---

## Estructura del proyecto

[](https://github.com/vperezguzman66/BTC-MONITOR#estructura-del-proyecto)

```
src/
├── api/
│   └── coingecko.js          API client con caché, deduplicación y retry con backoff
├── store/
│   └── storage.js            Capa centralizada de localStorage (alertas, portafolio, monedas)
├── hooks/
│   ├── useFetch.js           Hook genérico: fetch + loading + error + limpieza de datos stale
│   ├── useAlerts.js          Estado de alertas + verificación automática de precios
│   ├── usePortfolio.js       Estado del portafolio con persistencia automática
│   └── useLivePrice.js       Polling de precio en vivo cada 30s con degradación elegante
├── utils/
│   ├── format.js             Formateo de moneda, porcentaje y fechas (Intl)
│   ├── indicators.js         MA simple y cálculo de DCA
│   └── signals.js            Motor de señales: RSI, MACD, Bollinger, MA, score compuesto
├── components/
│   ├── PriceChart.jsx        Gráfico de área + líneas MA50/MA200 (recharts)
│   ├── AlertsPanel.jsx       Crear, listar y eliminar alertas de precio
│   ├── SignalPanel.jsx       Panel de señal de trading con gauge RSI y breakdown
│   ├── PortfolioPanel.jsx    Seguimiento de posiciones y P&L en tiempo real
│   ├── DCACalculator.jsx     Simulador de inversión periódica con datos históricos
│   ├── CoinPicker.jsx        Modal para seleccionar qué monedas activar
│   ├── GuidePanel.jsx        Guía educativa de indicadores (acordeón)
│   └── Tooltip.jsx           Componente reutilizable de tooltip al hover
├── App.jsx                   Orquestador: combina hooks, renderiza layout
├── main.jsx                  Entry point
└── styles.css                Tema oscuro completo (CSS puro, sin framework)
```

---

## Arquitectura

[](https://github.com/vperezguzman66/BTC-MONITOR#arquitectura)

### Hooks personalizados

[](https://github.com/vperezguzman66/BTC-MONITOR#hooks-personalizados)

|Hook|Responsabilidad|
|---|---|
|`useLivePrice`|Polling de precio cada 30s. Llama `onPriceTick` sin restartar el intervalo cuando cambia el callback (ref pattern).|
|`useAlerts`|Estado de alertas + persistencia en localStorage + disparo de notificaciones.|
|`usePortfolio`|Posiciones del portafolio + persistencia automática.|
|`useFetch`|Patrón genérico `active flag + loading/error + limpieza de stale`. Usado en SignalPanel, DCACalculator y gráfico.|

### Capa de red (`api/coingecko.js`)

[](https://github.com/vperezguzman66/BTC-MONITOR#capa-de-red-apicoingeckojs)

Tres mecanismos para tolerar el límite de la API gratuita (~1 req/s):

1. **Caché en memoria** — el histórico se sirve desde caché durante 2 minutos; el precio durante 25 segundos.
2. **Deduplicación de in-flight** — si dos componentes piden la misma URL simultáneamente, solo se lanza un fetch real; el segundo espera el mismo `Promise`.
3. **Retry con backoff exponencial y jitter** — ante un 429 o "Failed to fetch", reintenta hasta 4 veces con esperas de ~1.5s, 3s, 6s (+ jitter). Respeta el header `Retry-After`.

---

## Monedas disponibles

[](https://github.com/vperezguzman66/BTC-MONITOR#monedas-disponibles)

|Categoría|Monedas|
|---|---|
|**Principales**|BTC, ETH, BNB, XRP, SOL|
|**Layer 1**|ADA, AVAX, DOT, NEAR, SUI, APT, TRX, HBAR, HYPE, TON|
|**DeFi**|LINK, UNI, ATOM, XLM|
|**Meme**|DOGE, SHIB, PEPE|
|**Clásicas**|LTC, BCH, XMR|

La selección activa se guarda en `localStorage`. Por defecto: BTC, ETH, SOL, ADA.

---

## Notas importantes

[](https://github.com/vperezguzman66/BTC-MONITOR#notas-importantes)

- **No es asesoramiento financiero.** Los indicadores técnicos son herramientas de análisis, no garantías de rendimiento. Invierte solo lo que estés dispuesto a perder.
- **Datos de CoinGecko.** La API gratuita puede devolver errores 429 si se excede el límite de peticiones. La app reintenta automáticamente.
- **Sin backend.** Todo corre en el navegador. No hay cuentas, no hay servidor propio, no se envía ningún dato a terceros salvo las peticiones a CoinGecko.
- **Persistencia local.** Portafolio, alertas y selección de monedas se guardan en `localStorage` del navegador. Limpiar la caché del navegador los borra.