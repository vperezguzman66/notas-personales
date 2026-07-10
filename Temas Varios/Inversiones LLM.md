[[Varios]]

Apuntes personales sobre el mundo de las inversiones ligadas a IA/LLM. Dos ángulos distintos que conviene no mezclar:

1. **Invertir EN el sector IA/LLM** — comprar acciones/ETFs de empresas que fabrican o explotan esta tecnología.
2. **Usar LLMs COMO herramienta** para investigar, analizar y apoyar decisiones de inversión (en cualquier sector, no solo IA).

Estado (actualizado 2026-07-10): sin fondos para invertir y sin intención de invertir por ahora — el **Ángulo 1 queda en pausa**. El foco actual es 100% el **Ángulo 2**: construir la herramienta. Esta nota documenta el diseño; el código vive en `Proyectos/inversiones-llm/` (fuera de este vault), con su propio `CLAUDE.md` y `README.md`, y en GitHub (privado): [github.com/vperezguzman66/inversiones-llm](https://github.com/vperezguzman66/inversiones-llm).

**Progreso del código (al 2026-07-10):**
- RAG funcionando de punta a punta: `chunk.js`, `embeddings.js` (Voyage AI), `db.js` (SQLite), `ingest.js`, `retrieve.js`.
- Esquema de "propuesta de orden" implementado y probado: `propose` → `proposals` → `decide`, auditable.
- Corpus con 11 documentos reales: Nvidia, Microsoft, Alphabet, AMD, Broadcom/TSM, Meta, Amazon/Trainium, Apple (estrategia "capex-lite", contraste directo con el resto), comparación de ETFs de IA, y dos riesgos transversales documentados aparte (financiamiento circular Nvidia/OpenAI/Oracle/Broadcom ~$800B, restricción energética de centros de datos) — todos con research de julio 2026, frontmatter `doc_type`/`ticker`/`date`.
- **Prueba de la salvaguarda "sin research, no inventar" (2026-07-10):** se probó `/analizar-inversion` con AAPL *antes* de tener documento propio — el RAG devolvió solo coincidencias tangenciales (score máx. 0.52) y la propuesta lo dijo explícitamente en vez de rellenar con conocimiento genérico, quedando en "mantener" con la invalidación "agregar research real antes de decidir". Después de agregar el documento de Apple, la misma consulta pasó a 0.63 de score — confirma que la salvaguarda funciona y que agregar research mejora el sistema de forma medible.
- Dos bugs reales encontrados y corregidos al correr el ingest completo: `embeddings.js` no toleraba el rate limit de Voyage AI (3 RPM sin método de pago agregado — ahora reintenta automáticamente); `ingest.js` no era idempotente (reprocesar un archivo duplicaba sus chunks — ahora los reemplaza).
- 19 tests pasando. Sin capa de ejecución todavía (deliberado).
- **Skill `/analizar-inversion <ticker>`** (`.claude/skills/`) que encadena todo el flujo: retrieval del RAG → datos vivos de TradingView MCP → síntesis → propuesta → guardar pending → esperar aprobación → registrar decisión.
- **Flujo completo verificado con datos reales (2026-07-10):** análisis de NVDA combinando 5 chunks del RAG (riesgos de TSM, competencia AMD/TPU) con datos vivos de TradingView (precio $210.01, RSI 68.90 cerca de sobrecompra tras un rebote, EMA 204.43, rango de 100 barras -2.71% — tendencia de mediano plazo aún negativa pese al rebote). Propuesta "mantener" guardada, mostrada, y **aprobada explícitamente** con nota. El circuito completo (RAG + datos vivos + propuesta + aprobación humana + auditoría) funciona de punta a punta.
- **Dashboard web local** (`npm run web`, `http://127.0.0.1:4100`, solo local — nunca expuesto a la red): pestañas de propuestas pendientes (aprobar/rechazar con botones), decididas, corpus indexado, y búsqueda semántica. Requirió un refactor previo (lógica de retrieval y propuestas movida a `search.js`/`proposals-store.js`, reutilizada por CLI y web). Deliberadamente **sin botón de "analizar"** — eso exigiría una API key propia de Anthropic y el MCP de TradingView no es alcanzable desde un servidor web normal; el análisis sigue siendo exclusivo del skill dentro de Claude Code. Probado en el navegador de punta a punta, incluyendo aprobar una propuesta real con un click.

**Documento de conceptos:** `Proyectos/inversiones-llm/docs/como-funciona.md` explica cada pieza (RAG, embeddings, chunking, similitud coseno, propuesta estructurada, skill, MCP, dashboard) con definiciones claras — para entender el sistema, no solo operarlo.

### Secuencia de uso

Setup (`npm install` + `.env` con `VOYAGE_API_KEY`) es una sola vez. Después, el ciclo normal:

1. **Ingest** — `npm run ingest`, cada vez que cambia algo en `corpus/` (no en cada análisis).
2. **Analizar** — `/analizar-inversion <ticker>` dentro de una sesión de Claude Code abierta en el proyecto (requiere TradingView Desktop abierto para los datos vivos).
3. **Propuesta guardada** — automático dentro del paso 2, queda como `pending`.
4. **Decidir** — aprobar/rechazar por el dashboard web (`npm run web`) o por CLI (`npm run decide`).

El ciclo vuelve al paso 2 para el siguiente ticker.

## Ángulo 1 — Invertir en el sector IA

### Empresas líderes (consenso julio 2026)

- **Nvidia (NVDA)** — domina ~90% del mercado de GPUs para entrenar/ejecutar modelos. Es la apuesta más directa y también la más cara/consensuada (todo el mundo ya la conoce, lo cual infla la valoración).
- **Microsoft (MSFT)** y **Alphabet/Google (GOOGL)** — exposición a IA vía su negocio de nube (Azure, Google Cloud), que es donde se entrenan y sirven los modelos de terceros. Más diversificado que Nvidia porque tienen otros negocios de respaldo.
- **Broadcom (AVGO)** y **Taiwan Semiconductor (TSM)** — infraestructura/fabricación de chips, un nivel más abajo en la cadena que Nvidia (TSM literalmente fabrica los chips de Nvidia).
- **AMD** — competidor de Nvidia en chips de IA, con la ventaja de estar más diversificado (PCs, servidores, centros de datos).

Idea recurrente en las fuentes: no apostar todo a una sola acción — diversificar entre subsectores (chips, nube, software, infraestructura, aplicaciones).

### ETFs como alternativa diversificada

En vez de elegir acciones individuales, existen ETFs temáticos:

| ETF | Foco | Retorno ~1 año | Expense ratio |
|---|---|---|---|
| **AIQ** (Global X AI & Technology) | 85 empresas: chips, hardware, software IA | +40.6% | 0.68% |
| **BOTZ** (Global X Robotics & AI) | 68 empresas: robótica industrial, automatización, autónomos | +16.3% | 0.68% |
| **ROBO** (Robo Global Robotics & Automation) | 78 empresas, ponderación equilibrada (máx 2.5% por posición) | — | — |

AIQ tuvo mejor retorno reciente pero valoraciones más altas (P/E ~24.6, P/B ~4.5) — más concentrado en el "boom" actual. BOTZ/ROBO son más defensivos por diversificación y menor concentración por posición.

**Nota:** estas cifras son de julio 2026, van a quedar desactualizadas rápido — antes de decidir, revisar cotización y rendimiento actual, no confiar en esta tabla como dato vivo.

## Ángulo 2 — Usar LLMs como herramienta para invertir

### Panorama de modelos/herramientas (julio 2026)

- **Claude (Fable 5 / Sonnet 5)** — mejor en razonamiento profundo y análisis narrativo largo; el mejor puntaje en el Hebbia Finance Benchmark; menor tasa de alucinación que la competencia (más propenso a decir "no sé" que inventar un número).
- **ChatGPT (GPT-5.5)** — buena cobertura general, pero mayor tasa de alucinación de datos financieros que Claude en varios estudios.
- **Gemini 3.1 Pro** — fuerte en integración con Google Workspace.
- **Microsoft Copilot** — fuerte en integración M365 y gobernanza empresarial.
- **FinRobot** (open source, [GitHub](https://github.com/ai4finance-foundation/finrobot)) — plataforma que combina LLMs con cálculos deterministas (DCF, WACC, comparables, Monte Carlo) — el LLM razona y redacta, pero los números salen de código verificable, no de la generación del modelo. Patrón interesante: usar el LLM para síntesis/explicación, nunca para el cálculo en sí.

Patrón que se repite en 2026: combinar un modelo "premium" de razonamiento (Claude Fable 5, GPT-5.5) para el análisis narrativo con un modelo más barato para procesamiento masivo/rutinario.

### Flujo propio posible: Claude + TradingView MCP

Ya tengo el MCP de TradingView conectado en Claude Code (`mcp__tradingview__*`), que permite leer gráficos en vivo, indicadores, y hasta Pine Script. Un flujo razonable para research (no para automatizar compras):

1. **Lectura de contexto** — `chart_get_state` + `data_get_study_values` para ver de un vistazo qué indicadores están activos y sus valores actuales (RSI, MACD, medias móviles) sobre un ticker.
2. **Snapshot de precio** — `quote_get` para el precio en tiempo real, `data_get_ohlcv` (con `summary=true`) para el histórico resumido.
3. **Pedir a Claude que sintetice**, no que decida — usar el LLM para explicar qué está pasando ("¿por qué se ve así este RSI?", "resume la tendencia de las últimas 4 semanas") en vez de pedirle una señal de compra/venta directa.
4. **Registrar la tesis por escrito** (en esta misma nota o en una nueva por posición) — fecha, precio de entrada considerado, razón, y qué invalidaría la tesis. Esto es más valioso que cualquier respuesta puntual del LLM, porque permite revisar después si el razonamiento era sólido o no.

Esto es exploración — por ahora no hay una posición ni una automatización real montada.

### Especializar un modelo existente en inversiones (para planificar implementación)

Pregunta de fondo: en vez de un flujo genérico, ¿se puede tomar un modelo ya entrenado (Claude, GPT, Llama) y hacerlo "experto" en inversiones? Hay tres rutas, que tocan partes distintas del pipeline y no son intercambiables:

**1. Prompting / contexto** — no toca el modelo. Instrucciones y ejemplos en cada llamada (system prompt, formato deseado). Es lo que ya hago hoy con Claude. Cero infraestructura, pero no hay memoria persistente entre conversaciones — cada sesión "olvida" el contexto de dominio salvo que se lo vuelva a dar.

**2. RAG (Retrieval-Augmented Generation)** — tampoco toca los pesos. Se arma una base de conocimiento propia (informes, tesis registradas, notas como esta) y antes de cada consulta el sistema busca los documentos relevantes y los inyecta como contexto al modelo. El modelo sigue siendo el mismo; solo "lee" documentos propios antes de responder.

**3. Fine-tuning** — la única que modifica los pesos del modelo, reentrenándolo con ejemplos del dominio.
- **Obstáculo concreto con Claude:** Anthropic no ofrece fine-tuning público de sus modelos — solo Claude 3 Haiku, y únicamente vía Amazon Bedrock (us-west-2). Es una decisión deliberada de seguridad (evitar degradar las propiedades de Constitutional AI), no un límite técnico temporal. Fine-tuning real requeriría un modelo de pesos abiertos (Llama, Mistral, DeepSeek), no Claude.
- **Costo real (2026):** con LoRA/QLoRA, sorprendentemente barato — $50-500 por corrida, un GPU de consumo (RTX 4090, 24GB VRAM) alcanza para modelos de 7B, 500-1000 ejemplos de calidad son suficientes, un GPU corre la fine-tune en una noche.
- **El problema no es el costo, es la finalidad:** el fine-tuning hornea el conocimiento en los pesos del modelo. En finanzas, donde los datos (precios, informes, filings) cambian constantemente, eso significa que el modelo queda desactualizado apenas cambian los datos — habría que reentrenar cada vez. Por eso el consenso de la industria en 2026 es usar fine-tuning solo para fijar **formato/tono/comportamiento consistente**, nunca para inyectar hechos que cambian.

**Recomendación práctica para mi caso** (sin infraestructura GPU propia, ya usando Claude vía API/Claude Code): el camino realista es **prompting + RAG**, no fine-tuning — se puede construir con lo que ya tengo, sin rentar GPUs ni gestionar pesos de modelo. Fine-tuning quedaría como paso posterior, solo si RAG no resuelve algo específico de formato/comportamiento.

### Arquitectura RAG en detalle

**Principio de diseño: separar conocimiento estable de datos vivos.** No deben vivir en el mismo store.

- **Conocimiento estable (esto va al RAG)** — tesis de inversión propias, informes/filings leídos, notas como esta. Cambia poco, vale la pena indexarlo y buscarlo por similitud semántica.
- **Datos vivos (esto sigue viniendo del MCP de TradingView, no del RAG)** — precio, indicadores, OHLCV. Cambia constantemente; indexarlo en un vector store sería guardar datos que quedan obsoletos en minutos.

**Pipeline de ingesta** (offline, se corre cada vez que se agrega un documento nuevo):

```
Documentos  →  Chunking + Embeddings  →  Vector store
(tesis,         (~500 palabras por        (vectores +
informes)        chunk, con solape)        metadata)
```

**Pipeline de consulta** (en tiempo real, cada vez que se pregunta algo):

```
Pregunta  →  Retrieval (top-k más similares)  →  Claude responde
(lenguaje      (busca en el vector store,          (con los chunks
 natural)       filtra por metadata si aplica)      recuperados como contexto)
```

**Decisiones técnicas concretas** (partiendo de cero, sin infraestructura dedicada — no sobre-construir):

| Decisión | Opción recomendada | Por qué |
|---|---|---|
| Embeddings | API de Voyage AI — modelo `voyage-3-large` (recomendación oficial de Anthropic para Claude, top del ranking MTEB) | Se integra bien con Claude, no requiere GPU propia. Voyage AI también ofrece un modelo específico para finanzas si el corpus crece — evaluarlo más adelante, no bloquea el arranque con el modelo general |
| Vector store | Array en memoria con cosine similarity, persistido en JSON o SQLite | Con decenas/cientos de documentos personales, un índice vectorial dedicado (Vectorize, Pinecone) es over-engineering |
| Chunking | ~500 palabras por chunk, con solapamiento de ~50 palabras | Suficiente granularidad sin fragmentar el razonamiento de una tesis completa |
| Metadata por chunk | fuente, fecha, ticker/tema relacionado, tipo (tesis propia vs. informe externo) | Permite filtrar antes de rankear — ej. "solo mis tesis", "solo informes de NVDA" |

**Control de calidad antes de confiar en las respuestas:** antes de usar el flujo para decisiones reales, hacer preguntas de prueba y verificar manualmente que los chunks recuperados sean los relevantes (grounding check) — si el retrieval trae basura, la respuesta de Claude hereda ese error aunque el modelo razone bien.

**Esqueleto de implementación:**
- [ ] Reunir corpus inicial: tesis de inversión propias, notas como esta, informes/filings relevantes.
- [ ] Script de ingesta: lee documentos → chunking → embeddings (Voyage AI) → guarda en JSON/SQLite con metadata.
- [ ] Script de consulta: embed de la pregunta → cosine similarity contra el store → top-k chunks → arma el prompt → llama a Claude.
- [ ] Probar el grounding check con preguntas de prueba antes de confiar en el flujo.
- [ ] Integrar con el flujo Claude + TradingView MCP ya documentado más arriba: RAG aporta el conocimiento estable, TradingView aporta los datos vivos, en la misma conversación.
- [ ] Probar el flujo completo con research real (no con dinero) antes de tomar cualquier decisión con el output.

### Del research assistant al agente de decisión (con aprobación humana)

El objetivo real no es solo un asistente de research — es una IA que **proponga** decisiones de inversión concretas, con un humano (yo) aprobando antes de cualquier ejecución. Esto añade dos capas sobre el RAG + TradingView ya diseñados:

```
Análisis          →    Propuesta           →    Ejecución
(RAG + datos vivos)     (con tu aprobación)      (fase futura)
```

**Análisis** — sin cambios: el RAG + MCP de TradingView ya documentados arriba.

**Propuesta** — la pieza nueva. Claude no debe responder en texto libre ("yo compraría NVDA"); debe producir una estructura fija y siempre completa:
- Ticker
- Acción: comprar / vender / mantener
- Cantidad sugerida
- Precio límite sugerido
- Razón, citando qué chunks del RAG y qué datos vivos la sustentan
- Qué invalidaría la tesis (condición de salida)

Esa estructura es lo que reviso y apruebo o rechazo — nunca una recomendación vaga en prosa.

**Ejecución — deliberadamente sin diseñar todavía.** Conectar esto a un bróker real (Interactive Brokers, Alpaca, etc.) implica elegir bróker con API, gestionar credenciales con el mismo cuidado que ya aplico en otros proyectos (nunca tokens en el cliente, como en `buscagit`), y decidir permisos (¿puede vender también, o solo comprar?). No tiene sentido diseñar esto en detalle sin capital ni bróker elegido — es una conversación aparte cuando llegue el momento. Mientras tanto, el sistema corre en **modo simulación**: registra qué habría propuesto, sin ejecutar nada — mismo patrón que ya uso en [ml-bot](../../../Proyectos/ml-bot) (modo simulación cuando no hay credenciales reales).

**Auditoría de decisiones.** Cada propuesta y cada aprobación/rechazo debe quedar en un log — mismo patrón que `audit_log` en [Ferreteria](../../../Ferreteria). No es por cumplimiento regulatorio (esto es una herramienta personal, no un producto para terceros); es para poder revisar después si el razonamiento de la IA era bueno, y detectar sesgos o errores recurrentes antes de confiarle capital real.

**Esqueleto de implementación (agente de decisión) — completado:**
- [x] Definir el esquema fijo de la "propuesta de orden" como salida estructurada (no texto libre) — implementado en `src/proposal-schema.js`.
- [x] Construir el store de propuestas + aprobaciones/rechazos (auditable), separado del vector store del RAG — tabla `proposals` en la misma SQLite, con `status`, `decision_note`, `decided_at`.
- [ ] Correr en modo simulación durante un tiempo — sin bróker conectado — y revisar periódicamente si las propuestas tenían sentido.
- [ ] Solo cuando exista capital real: diseñar la capa de ejecución como su propio proyecto (elección de bróker, seguridad de credenciales, permisos).

**Esquema final de la propuesta** (validado en código, ver `CLAUDE.md` del proyecto para el detalle):

| Campo | Regla |
|---|---|
| `ticker` | obligatorio |
| `action` | `comprar` / `vender` / `mantener` |
| `quantity`, `limit_price` | obligatorios y > 0 si se compra/vende; se ignoran si es mantener |
| `reasoning` | obligatorio, debe citar qué lo sustenta |
| `invalidation` | obligatorio, condición que invalida la tesis |
| `sources` | ids de chunks del RAG citados, para trazabilidad |

Flujo verificado de punta a punta con datos reales: `npm run propose` (guarda con status `pending`) → `npm run proposals -- pending` (revisar) → `npm run decide -- <id> approved "nota"` (queda auditado, con timestamp; no se puede redecidir una propuesta ya cerrada).

### Infraestructura y herramientas de desarrollo

**Modo de ejecución decidido: interactivo** — yo invoco el análisis dentro de Claude Code, no corre solo en segundo plano. Esto simplifica mucho la infraestructura: nada queda corriendo cuando no lo uso.

**Infraestructura:**
- Sin servidor, sin hosting, sin Docker, sin PM2 — todo corre a demanda, local, en Claude Code.
- TradingView Desktop debe estar abierto y logueado para los datos vivos (ya conectado vía MCP).
- Carpeta de proyecto local: documentos del corpus, vector store (SQLite), log de propuestas/aprobaciones (auditoría).

**Cuentas/servicios externos:**
- **Voyage AI** — cuenta + API key, solo para embeddings. Es la única cuenta nueva a crear.
- **No hace falta API key propia de Anthropic** — el razonamiento de la "Propuesta" lo hace la sesión de Claude Code misma, no un script separado orquestando llamadas a la API.
- Bróker: nada todavía (fase futura).

**Herramientas de desarrollo:**
- Node.js (ya instalado, v24) — mismo lenguaje que Ferreteria/ml-bot.
- `better-sqlite3` — mismo patrón que Ferreteria, para el vector store y el log de auditoría.
- `fetch` nativo (o SDK de Voyage) para la API de embeddings.
- `node --test` para tests.
- `CLAUDE.md` propio del proyecto, como los demás proyectos del workspace.
- Opcional: slash command/skill de Claude Code (ej. `/analizar-inversion <ticker>`) que encadene retrieval → MCP de TradingView → propuesta estructurada, para no repetir el flujo a mano.

## Riesgos y limitaciones (por qué no confiar ciegamente)

- **Tasa de alucinación de datos financieros:** un estudio de UPenn encontró que ChatGPT daba datos financieros incorrectos ~35% de las veces al preguntar por empresas públicas; otro reporte (NIH) llega hasta 47% en ciertos escenarios. Claude tiende a tener tasas más bajas, pero **ningún modelo está exento**.
- **Números no reproducibles:** los LLMs son probabilísticos — el mismo prompt puede dar respuestas ligeramente distintas. Para finanzas, donde cada cifra debería ser auditable y trazable, esto es un problema estructural, no un bug puntual.
- **Sin datos en tiempo real (según el modelo/setup):** el conocimiento base de un LLM tiene fecha de corte; sin una herramienta conectada a datos en vivo (como el MCP de TradingView), no sabe el precio de hoy ni catalizadores recientes.
- **No es un asesor financiero licenciado:** no puede evaluar tolerancia al riesgo personal, situación tributaria ni objetivos de largo plazo con responsabilidad fiduciaria — sus respuestas son reconocimiento de patrones, no asesoría regulada.
- **Regla práctica:** usar el LLM para *entender* y *sintetizar* información (leer un balance, explicar un indicador, resumir noticias), nunca como fuente única de una cifra específica sin verificarla en la fuente original (10-K, cotización real, etc.), y nunca como el que toma la decisión final de compra/venta.

## Próximos pasos

**Ángulo 1 (invertir) — en pausa**, sin fondos disponibles. Retomar recién cuando exista capital separado del fondo de emergencia y gastos fijos:
- [ ] (Bloqueado) Definir monto y horizonte de tiempo antes de mirar tickers concretos.
- [ ] (Bloqueado) Decidir: ¿acciones individuales, ETF diversificado, o ambos?

**Ángulo 2 (construir la herramienta) — foco actual:**
- [ ] Arrancar por el RAG (ver esqueleto de implementación arriba) — es la base de todo lo demás.
- [ ] Definir el esquema de "propuesta de orden" y correr en modo simulación (ver esqueleto del agente de decisión arriba).
- [ ] Revisar cifras de este documento antes de actuar — quedan desactualizadas rápido.

## Fuentes

- [Las 10 mejores inversiones en IA en 2026](https://www.lisdatasolutions.com/es/blog/las-10-mejores-inversiones-en-inteligencia-artificial-en-2026/)
- [Invertir en IA: diez oportunidades para 2026](https://blog.activotrade.com/invertir-en-ia-diez-oportunidades-para-2026)
- [Las mejores acciones de IA para 2026: Nvidia, Broadcom y TSM](https://neuron.expert/news/prediction-these-3-artificial-intelligence-ai-stocks-will-be-big-winners-again-in-2026/15867/es/)
- [AIQ vs. BOTZ — TipRanks](https://www.tipranks.com/news/aiq-vs-botz-which-ai-etf-could-deliver-bigger-returns-in-2026)
- [The Best AI and Robotics ETFs to Buy in 2026 — Kiplinger](https://www.kiplinger.com/investing/etfs/601112/top-artificial-intelligence-ai-etfs)
- [Best LLMs for finance teams in 2026 — Aleph](https://www.getaleph.com/answers/best-llms-finance-teams-2026)
- [FinRobot — AI4Finance Foundation (GitHub)](https://github.com/ai4finance-foundation/finrobot)
- [Using AI for financial advice? Proceed with caution](https://stockpil.com/ai-financial-advice-caution)
- [Personal finance and AI — Euronews](https://www.euronews.com/business/2025/10/30/personal-finance-and-ai-should-you-trust-chatgpts-investment-advice)
- [Claude fine-tuning: guía completa (callsphere.ai)](https://callsphere.ai/blog/vw8g-anthropic-claude-fine-tuning-patterns-bedrock-2026)
- [Fine-tuning Claude 3 Haiku en Amazon Bedrock — AWS](https://aws.amazon.com/blogs/aws/fine-tuning-for-anthropics-claude-3-haiku-model-in-amazon-bedrock-is-now-generally-available/)
- [How to Fine-Tune LLMs in 2026: Costs, GPUs, and Code — Spheron](https://www.spheron.network/blog/how-to-fine-tune-llm-2026/)
- [RAG vs Fine-Tuning: A 2026 Decision Framework — Zartis](https://www.zartis.com/rag-vs-fine-tuning-a-2026-decision-framework/)
- [Embeddings — Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/embeddings)
- [Claude Cookbook — Voyage AI embeddings (GitHub)](https://github.com/anthropics/claude-cookbooks/blob/main/third_party/VoyageAI/how_to_create_embeddings.md)
