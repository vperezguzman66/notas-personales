---
proyecto: "polizas-web"
ruta: "Proyectos/polizas-web"
cliente: "Propio"
stack: "Cloudflare Workers + D1 + R2 + Claude API (visión)"
estado: "En desarrollo — D1 creadas y migradas, R2 pendiente de habilitar"
ultimo_cambio: 2026-07-06
---

[[Varios]]

Agregador de seguros: el usuario sube una foto o PDF de cada póliza que ya tiene contratada (auto, SOAP, incendio, robo, viaje, hogar, salud, vida) y la app extrae los datos clave con IA, los organiza en un solo dashboard y avisa antes de que venzan. MVP fase 1 — solo agregación y visualización, sin vender seguros.

Repo en GitHub (privado): https://github.com/vperezguzman66/polizas-web

## Origen de la idea

Nació de una idea más grande de Victor: un "seguro integral" que unificara en una sola app todos los seguros de una persona (auto, SOAP, viaje, incendio, robo, etc.), como una especie de wallet de seguros. Al analizarlo, la decisión clave era el modelo:

- **Aseguradora propia** (emitir pólizas directamente) — requiere licencias por cada ramo ante la CMF y reservas técnicas, carísimo y lento.
- **Corredor de seguros** — intermediario autorizado por la CMF que vende pólizas de distintas aseguradoras. Mucho menos capital, pero igual requiere inscripción en el Registro de Intermediarios de Seguros de la CMF, examen y garantía de responsabilidad civil.
- **Agregador puro** (el camino elegido para el MVP) — ni vende ni intermedia, solo organiza pólizas que el usuario ya contrató en otro lado. No requiere ninguna licencia para empezar.

Victor pidió explícitamente **no incluir la inscripción como corredor en el roadmap de fase 2** — el corretaje queda fuera de este proyecto por ahora, la fase 2 (aún no planificada) sería más bien un comparador de precios de mercado para pólizas por vencer.

## Decisiones de alcance del MVP

Antes de armar la estructura se resolvieron tres preguntas de diseño:
1. **Alcance**: agregador primero (solo visualizar pólizas existentes), no venta activa desde el día uno.
2. **Plataforma**: web app / PWA (no app móvil nativa, más rápido de construir y probar).
3. **Captura de datos**: carga manual de foto/PDF + extracción automática con IA (no solo formulario manual).

## Arquitectura

Mismo patrón de auth por cookie + D1 que `busca-medicamentos-web` (PBKDF2 vía `crypto.subtle`, sesión `httpOnly`). Documentos originales van a R2, nunca a D1. La extracción usa la API de Claude (visión) con `output_config.format` de tipo `json_schema` para forzar una respuesta estructurada (tipo, aseguradora, número de póliza, vigencia, prima, cobertura) directamente desde la imagen o PDF subido — sin OCR tradicional.

Flujo: `POST /api/extract` (solo extrae, no persiste) → el usuario confirma o corrige los datos en el frontend → `POST /api/policies` crea el registro → `POST /api/policies/:id/documents` sube el archivo original a R2 y lo asocia.

## Estado (2026-07-06)

Scaffold completo (backend, frontend, tests, migraciones) y repo en GitHub. Al aprovisionar en Cloudflare:
- ✅ D1 creadas y con schema aplicado: `polizas-web-users` y `polizas-web-users-staging`.
- ⏳ **R2 pendiente** — la cuenta de Cloudflare no tenía R2 habilitado (`wrangler r2 bucket create` dio error 10042, "Please enable R2 through the Cloudflare Dashboard"). Victor decidió activarlo después; hasta entonces no se pueden crear los buckets `polizas-web-documents` / `-staging`.
- ⏳ Falta configurar el secret `ANTHROPIC_API_KEY` (prod y staging) para que funcione la extracción.

Sin desplegar todavía — no hay Worker corriendo en producción ni en staging.
