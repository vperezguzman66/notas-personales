---
proyecto: "price_monitor"
ruta: "price_monitor"
cliente: "Propio"
stack: "Python + Playwright"
estado: "Funcional — fix logs duplicados, cron vía crontab"
ultimo_cambio: 2026-07-03
---

He creado un agente en que permite enviarme un correo cuando baje de precio.
El link en GitHub es el siguiente: https://github.com/vperezguzman66/price-monitor.git

[[Varios]]

Agente que revisa el precio de un producto en MercadoLibre Chile cada hora y envía un email cuando el precio baja un 10% o más respecto al precio inicial registrado.

## Como funciona

Cada hora (cron)
     │
     ▼
price_monitor.py
     │
     ├── Abre Chrome headless (Playwright) con stealth anti-bot
     ├── Carga la página del producto en MercadoLibre
     ├── Extrae el precio desde el meta tag itemprop="price"
     ├── Compara con el precio inicial guardado en data.json
     │
     ├── Si bajó >= 10% → envía email por Gmail SMTP
     └── Guarda el resultado en data.json e historial
     
**Por qué Playwright y no requests/curl:** MercadoLibre renderiza el precio con JavaScript (SPA). La librería requests obtiene solo el HTML vacío (~23KB); Playwright ejecuta el JS completo y obtiene el precio real. Se usan headers y fingerprint de browser real para evitar detección de bots.

## Archivos

~/price_monitor/
├── price_monitor.py   # Script principal
├── data.json          # Estado: precio inicial, historial, última notificación
├── monitor.log        # Log de cada ejecución
└── README.md          # Este archivo

## data.json -- estructura

{
  "initial_price": 29990.0,
  "initial_date": "2026-06-26T11:22:49",
  "currency": "CLP",
  "last_check": "2026-06-26T12:00:18",
  "last_price": 29990.0,
  "last_notified_price": 26500.0,     ← solo aparece si ya se notificó
  "last_notified_date": "...",
  "history": [
    { "date": "...", "price": 29990.0, "drop_pct": 0.0 },
    ...
  ]
}

## Configuración

Las variables de configuración están al inicio de `price_monitor.py`:

PRODUCT_URL        = "https://www.mercadolibre.cl/..."   # URL del producto
PRODUCT_NAME       = "Mouse Vertical..."                  # Nombre para el email
MIN_DROP_PCT       = 10     # % mínimo de bajada para notificar
MAX_DROP_PCT       = 20     # Si supera este % el email se marca como "OFERTA INUSUAL"
GMAIL_USER         = "vperezguzman@gmail.com"
GMAIL_APP_PASSWORD = "xxxx xxxx xxxx xxxx"               # App Password de Gmail
NOTIFY_EMAIL       = "vperezguzman@gmail.com"

### Cambiar el producto monitorado

[](https://github.com/vperezguzman66/price-monitor#cambiar-el-producto-monitorado)
1. Busca el producto en MercadoLibre Chile
2. Abre la página del producto y copia la URL
3. Edita `price_monitor.py` y actualiza `PRODUCT_URL` y `PRODUCT_NAME`

rm ~/price_monitor/data.json
python3 ~/price_monitor/price_monitor.py

### Cambiar el umbral de alerta

[](https://github.com/vperezguzman66/price-monitor#cambiar-el-umbral-de-alerta)

Edita `MIN_DROP_PCT` en `price_monitor.py`:

MIN_DROP_PCT = 15   # notificar solo si baja >= 15%

## Cron — programación automática

[](https://github.com/vperezguzman66/price-monitor#cron--programaci%C3%B3n-autom%C3%A1tica)

El agente corre mediante crontab del sistema. Para verlo:
crontab -l
0 * * * *  /Library/Frameworks/Python.framework/Versions/3.13/bin/python3  /Users/victor/price_monitor/price_monitor.py  >> /Users/victor/price_monitor/monitor.log  2>&1

`0 * * * *` significa: al minuto 0 de cada hora, todos los días.

**Requisito:** el Mac debe estar encendido y sin suspensión profunda en el momento de la revisión. Si está dormido, el chequeo de esa hora se salta (sin pérdida de datos).

## Comandos útiles

# Ver el log en tiempo real
tail -f ~/price_monitor/monitor.log

# Ejecutar el script manualmente (útil para probar)
python3 ~/price_monitor/price_monitor.py

# Ver el historial de precios registrado
cat ~/price_monitor/data.json

# Ver que el cron está activo
crontab -l

# Pausar el agente (elimina el cron)
crontab -r

# Restaurar el cron después de pausarlo
(crontab -l 2>/dev/null; echo "0 * * * * /Library/Frameworks/Python.framework/Versions/3.13/bin/python3 /Users/victor/price_monitor/price_monitor.py >> /Users/victor/price_monitor/monitor.log 2>&1") | crontab -

## Lógica de notificación

[](https://github.com/vperezguzman66/price-monitor#l%C3%B3gica-de-notificaci%C3%B3n)

- Solo envía email cuando el precio baja **por primera vez** a un nuevo mínimo por debajo del umbral.
- Si el precio sube de nuevo y luego vuelve a bajar aún más, envía otra notificación.
- No envía duplicados si el precio se mantiene estable en el mismo nivel.
- Si la bajada supera el `MAX_DROP_PCT` (20%), el email se etiqueta como **"OFERTA INUSUAL"** en rojo (posible error de precio o liquidación).

## Gmail App Password

[](https://github.com/vperezguzman66/price-monitor#gmail-app-password)

El script usa Gmail SMTP con un App Password (no la contraseña normal de Gmail).

Si necesitas regenerarlo:

1. Ve a [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
2. Crea un nuevo password con nombre "Price Monitor"
3. Reemplaza el valor de `GMAIL_APP_PASSWORD` en `price_monitor.py`

## Dependencias

[](https://github.com/vperezguzman66/price-monitor#dependencias)

```shell
pip3 install playwright
python3 -m playwright install chromium
```

Python 3.13 · Playwright 1.x · macOS

---
Bug fix (2026-07-03): logs duplicados en monitor.log

Problema: cada línea de log aparecía dos veces en `monitor.log`.

Causa: `logging.basicConfig` usaba un `FileHandler` (escribe directo a `monitor.log`) y un `StreamHandler` (escribe a stderr) al mismo tiempo. El crontab activo redirige `>> monitor.log 2>&1`, así que el `StreamHandler` volvía a escribir lo mismo en el mismo archivo. Un intento de fix anterior (commit `01bb8bb`) solo cubría el camino PM2 (redirigiendo su output a `/dev/null`), pero el mecanismo real en uso es crontab del sistema, no PM2.

Solución: se quitó el `StreamHandler`, dejando solo el `FileHandler`. Verificado ejecutando el script con la misma redirección que usa el cron: una ejecución agrega exactamente las líneas esperadas, sin duplicar. Pusheado a GitHub (`d3481a7`).

Nota: el repo sí tiene git con remoto privado (a diferencia de lo que decía la documentación general del workspace). El cron corre por `crontab -l`, no por PM2 — el `ecosystem.config.js` existe pero no está cargado en PM2 actualmente.

Pendiente (no resuelto aún, decisión del usuario): `GMAIL_APP_PASSWORD` está hardcodeado en texto plano en `price_monitor.py` y presente en el historial de git desde el primer commit. El repo es privado, pero sigue siendo una credencial viva en control de versiones — recomendable moverla a `.env` (gitignored) y rotarla.