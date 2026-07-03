[[Marcos - Diego]]

## Estado (2026-07-02)

Los 10 bugs detectados por code review (2026-06-28) y el `README.md` con arquitectura/instalación/endpoints ya están commiteados y pusheados a `origin/main` (`b7c791e`). No queda pendiente de sincronización local.

**Sin pendientes abiertos actualmente** para este proyecto.

# ML Bot — Respuestas automáticas para Mercado Libre

[](https://github.com/vperezguzman66/ml-bot#ml-bot--respuestas-autom%C3%A1ticas-para-mercado-libre)

Bot que responde automáticamente las preguntas de compradores en Mercado Libre usando keywords predefinidos y/o IA (Claude Haiku).

---

## Arquitectura

[](https://github.com/vperezguzman66/ml-bot#arquitectura)

```
ml-bot/
├── backend/
│   ├── main.py            # FastAPI — endpoints y lógica central
│   ├── config.py          # Carga de .env y flags ML_CONFIGURED / ANTHROPIC_CONFIGURED
│   ├── db.py              # SQLite — tablas questions, keywords, config
│   ├── ml_client.py       # Wrapper API de Mercado Libre (dry-run si sin credenciales)
│   ├── keyword_matcher.py # Matching por palabra clave con normalización de acentos
│   └── ai_responder.py    # Generación de respuesta con Claude Haiku
├── frontend/
│   ├── index.html         # Panel admin
│   ├── style.css
│   └── app.js
├── .env.example
├── requirements.txt
└── run.sh
```

## Flujo de procesamiento

[](https://github.com/vperezguzman66/ml-bot#flujo-de-procesamiento)

```
Pregunta llega (webhook ML o polling manual)
    │
    ▼
keyword_matcher.find_match(texto)
    ├─ Match → respuesta predefinida          (answered_by = "keyword")
    └─ Sin match
           ├─ ANTHROPIC_API_KEY configurada → Claude Haiku  (answered_by = "ai")
           └─ Sin API key → mensaje genérico de fallback     (answered_by = "fallback")
    │
    ▼
ml_client.post_answer()
    ├─ ML configurado → publica en Mercado Libre
    └─ Sin credenciales → dry-run (solo loguea)
    │
    ▼
db.mark_answered() → status = "answered"
```

## Instalación y arranque

[](https://github.com/vperezguzman66/ml-bot#instalaci%C3%B3n-y-arranque)

```shell
# Clonar e instalar dependencias
git clone https://github.com/vperezguzman66/ml-bot.git
cd ml-bot

# Configurar entorno
cp .env.example .env
# Editar .env con credenciales ML y/o Anthropic

# Arrancar (usa ~/.venv o crea uno local)
bash run.sh
# → http://127.0.0.1:8001
# → Docs Swagger: http://127.0.0.1:8001/docs
```

## Variables de entorno

[](https://github.com/vperezguzman66/ml-bot#variables-de-entorno)

```dotenv
# Mercado Libre (obligatorio para producción)
ML_CLIENT_ID=
ML_CLIENT_SECRET=
ML_REDIRECT_URI=http://localhost:8001/auth/callback
ML_ACCESS_TOKEN=      # se obtiene con el flujo OAuth
ML_REFRESH_TOKEN=
ML_USER_ID=

# Claude / Anthropic (opcional — sin esto usa respuesta genérica)
ANTHROPIC_API_KEY=

# App
ADMIN_SECRET=cambia_esta_clave
```

Sin credenciales en `.env` el bot funciona en **modo simulación**: `POST /api/simulate` prueba el flujo completo sin llamar a ML ni a la IA.

## API endpoints

[](https://github.com/vperezguzman66/ml-bot#api-endpoints)

|Método|Ruta|Auth|Descripción|
|---|---|---|---|
|`GET`|`/`|—|Panel admin|
|`GET`|`/api/status`|—|Estado de ML y Anthropic|
|`POST`|`/webhook/ml`|—|Receptor de notificaciones ML|
|`POST`|`/api/poll`|admin|Polling manual de preguntas|
|`POST`|`/api/simulate`|admin|Simular pregunta sin credenciales|
|`GET`|`/api/questions`|admin|Lista de preguntas|
|`GET/POST`|`/api/keywords`|admin|CRUD de keywords|
|`PUT/DELETE`|`/api/keywords/{id}`|admin|Actualizar / eliminar keyword|
|`GET/PUT`|`/api/config`|admin|Contexto del negocio para la IA|
|`GET`|`/auth/callback`|—|Callback OAuth ML|

**Auth admin:** header `x-admin-secret: <ADMIN_SECRET>`.

## Puesta en producción

[](https://github.com/vperezguzman66/ml-bot#puesta-en-producci%C3%B3n)

1. Crear app en [ML Developers](https://developers.mercadolibre.cl/) → obtener `ML_CLIENT_ID` y `ML_CLIENT_SECRET`.
2. Agregar como redirect URI: `https://tudominio.com/auth/callback`.
3. Rellenar `.env` y arrancar el servidor.
4. Navegar a `GET /api/status` → abrir `ml_auth_url` en el browser → autorizar → copiar tokens del JSON al `.env`.
5. En la app ML: Notificaciones → URL `https://tudominio.com/webhook/ml` → topic `questions`.

Si no hay dominio público, usar polling periódico:

```shell
# Cada 5 minutos
*/5 * * * * curl -s -X POST http://localhost:8001/api/poll -H "x-admin-secret: <clave>"
```