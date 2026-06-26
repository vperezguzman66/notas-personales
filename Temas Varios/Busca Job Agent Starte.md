[[Varios]]
[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#busca-job-agent-starter)

Agente modular en Python para buscar vacantes, filtrarlas por tu perfil, puntuarlas y enviarte alertas.

## Qué hace este starter

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#qu%C3%A9-hace-este-starter)

- Recolecta vacantes desde fuentes reales:
    - RemoteOK API
    - WeWorkRemotely RSS
    - GetOnBoard API (configurada para Chile por defecto)
    - Remotive API (filtrada por keywords de location)
    - LinkedIn Jobs guest endpoint (configurable por keywords/location)
    - Torre opportunities endpoint (filtrado local por keywords/location)
    - Greenhouse Jobs API (opcional por board token)
    - Lever Postings API (opcional por company token)
- Filtra y puntúa vacantes según:
    - Keywords objetivo
    - Keywords excluidas
    - Preferencia geográfica
    - Modo remoto
    - Salario mínimo
- Guarda resultados en SQLite (`data/jobs.db`)
- Muestra top vacantes en consola
- Envía alertas por Telegram (opcional)
- Permite ejecución única o diaria (scheduler)
- Permite seguimiento de pipeline por estado (`new`, `saved`, `applied`, `interview`, `rejected`)

## Estructura

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#estructura)

- `src/config.py`: carga de configuración desde `.env`
- `src/models.py`: modelos de datos (`JobPost`, `ScoredJob`)
- `src/sources/`: conectores de fuentes
- `src/services/`: collector, scorer, storage, notifier
- `src/jobs/scheduler.py`: ejecución programada
- `src/main.py`: CLI principal
- `tests/test_scorer.py`: tests iniciales

## Requisitos

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#requisitos)

- Python 3.11+

## Instalación

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#instalaci%C3%B3n)

1. Crear entorno virtual
2. Instalar dependencias de `requirements.txt`
3. Ajustar variables en `.env`

## Variables importantes (`.env`)

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#variables-importantes-env)

- `SEARCH_KEYWORDS`: palabras clave objetivo (CSV)
- `EXCLUDED_KEYWORDS`: palabras para descartar (CSV)
- `PREFERRED_COUNTRIES`: países/regiones preferidos (CSV)
- `REMOTE_ONLY`: `true/false`
- `MIN_SALARY_USD`: mínimo salarial esperado
- `RUN_HOUR`: hora diaria (UTC) para modo scheduler
- `TOP_K`: cantidad máxima a notificar
- `DB_PATH`: ubicación de SQLite
- `GETONBRD_COUNTRY_CODES`: países para GetOnBoard (CSV, por defecto `cl`)
- `GETONBRD_PAGES`: páginas a consultar en GetOnBoard
- `GETONBRD_PER_PAGE`: tamaño de página en GetOnBoard
- `REMOTIVE_LOCATION_KEYWORDS`: keywords para filtrar location en Remotive (CSV)
- `LINKEDIN_ENABLED`: habilita/deshabilita fuente LinkedIn
- `LINKEDIN_KEYWORDS`: keywords de búsqueda para LinkedIn (CSV)
- `LINKEDIN_LOCATION`: ubicación objetivo para LinkedIn (ej. `Chile`)
- `LINKEDIN_PAGES`: cantidad de páginas a consultar en LinkedIn guest endpoint
- `TORRE_ENABLED`: habilita/deshabilita fuente Torre
- `TORRE_KEYWORDS`: keywords para filtrar resultados Torre localmente (CSV)
- `TORRE_LOCATION_KEYWORDS`: filtro local por location en Torre (CSV)
- `TORRE_INCLUDE_REMOTE`: incluye vacantes remotas Torre aunque no calcen por location
- `TORRE_MAX_KEYWORDS`: máximo de keywords usadas para consultas múltiples en Torre
- `TORRE_ROTATE_KEYWORDS`: rota automáticamente la ventana de keywords en cada día/ejecución
- `GREENHOUSE_BOARDS`: boards de Greenhouse (CSV, opcional)
- `LEVER_COMPANIES`: compañías en Lever (CSV, opcional)
- `TELEGRAM_BOT_TOKEN` y `TELEGRAM_CHAT_ID` (opcionales)

## Uso

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#uso)

- Correr una vez:
    - `python -m src.main once`
- Correr diariamente:
    - `python -m src.main schedule`
- Marcar una vacante por URL:
    - `python -m src.main set-status --url "https://..." --status applied`
- Listar vacantes por estado:
    - `python -m src.main list-status --status applied --limit 20`
- Listar últimas vacantes con estado (sin filtro):
    - `python -m src.main list-status --limit 20`
- Flujo rápido diario (mover en lote por score):
    - `python -m src.main apply-today --from-status new --to-status saved --min-score 70 --limit 15`
    - Ejemplo para marcar como aplicadas: `python -m src.main apply-today --from-status saved --to-status applied --min-score 75 --limit 10`
- Revisión interactiva una por una (con atajos):
    - `python -m src.main review-today --from-status saved --min-score 70 --limit 10`
    - Por defecto abre cada URL en el navegador automáticamente.
    - Al revisar una vacante, guarda `last_reviewed_at` en la base.
    - Si está habilitado `--open-url`, también copia la URL al portapapeles.
    - Si no quieres apertura automática: `--no-open-url`
    - Atajos: `a=applied`, `s=saved`, `i=interview`, `r=rejected`, `n=skip`, `q=quit`
    - Vista previa sin cambios: `python -m src.main review-today --from-status new --min-score 65 --limit 10 --dry-run`

## Dashboard web (Streamlit)

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#dashboard-web-streamlit)

- Levantar dashboard:
    - `streamlit run src/dashboard/app.py`
- Qué verás:
    - Métricas generales (total, nuevas, revisadas hoy)
    - Pipeline por estado
    - Vista Kanban por estado con botones para mover vacantes
    - Filtros guardados (presets + personalizados)
    - Gráfica diaria de nuevas vs revisadas (ventana configurable)
    - Explorador con filtros por estado, score, búsqueda y location (ej. `Santiago, Chile`)
    - Modo de match para location: `flexible` (tokens) o `strict` (frase exacta)
    - Acciones masivas para mover múltiples vacantes de una vez

> Nota: el tablero usa una UX tipo Kanban con botones de mover (drag/drop nativo no viene en Streamlit core).
> 
> Los filtros personalizados se guardan en `data/dashboard_filters.json`.

El dashboard usa la misma base SQLite configurada en `DB_PATH` dentro de `.env`.

## Mini rutina diaria sugerida

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#mini-rutina-diaria-sugerida)

1. Ejecutar recolección: `python -m src.main once`
2. Preseleccionar buenas vacantes: `python -m src.main apply-today --from-status new --to-status saved --min-score 70 --limit 15`
3. Ver preselección: `python -m src.main list-status --status saved --limit 20`
4. Tras postular: `python -m src.main apply-today --from-status saved --to-status applied --min-score 0 --limit 10`
5. Para revisión manual fina: `python -m src.main review-today --from-status saved --min-score 0 --limit 10`

## Próximos pasos recomendados

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#pr%C3%B3ximos-pasos-recomendados)

- Crear una capa de deduplicación semántica adicional
- Integrar email además de Telegram
- Exponer dashboard web simple (FastAPI + HTMX o Streamlit)

## Nota legal

[](https://github.com/vperezguzman66/busca-job/blob/main/README.md#nota-legal)

Si agregas scraping de sitios sin API/RSS, respeta los términos de uso y `robots.txt`.

LinkedIn puede variar su estructura HTML con el tiempo; si eso ocurre, el parser del guest endpoint puede requerir ajustes.

Torre actualmente entrega un set limitado de resultados por request en su endpoint público, por lo que el conector realiza múltiples consultas (por keyword, configurable), rota la ventana de keywords para ampliar cobertura entre ejecuciones y luego aplica deduplicación/filtros locales para priorizar vacantes relevantes.