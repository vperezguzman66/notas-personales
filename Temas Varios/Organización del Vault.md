[[Varios]]

Notas sobre el trabajo de organización del propio vault de Obsidian (no es un proyecto de código, es meta-documentación sobre el vault y las herramientas que lo acompañan).

## 2026-07-03 / 2026-07-04

- **Frontmatter unificado en 20 notas de proyecto**: se agregó el esquema `proyecto` / `ruta` / `cliente` / `stack` / `estado` / `ultimo_cambio` (opcional) a todas las notas que representan un proyecto de código, incluyendo las 7 que faltaban (ProvDent, ProvDent-limpio, ProvDent-limpio-historial, Protadult, Control de Finanzas, LimpiaPC, Limpieza de Mac).
- **`Proyectos.base`** (raíz del vault): Obsidian Base que tabula automáticamente todas las notas con la propiedad `proyecto` (filtro `not proyecto.isEmpty()`), agrupadas por cliente. Dos vistas: "Todos los proyectos" (agrupada por `cliente`) y "Por último cambio" (agrupada por `ultimo_cambio`). No requiere mantenimiento manual — cualquier nota nueva con ese frontmatter aparece sola.
- **`Mapa de Proyectos.canvas`** (raíz del vault): mapa visual de los 20 proyectos agrupados por cliente (Marcos/Diego, René Medina, VP Services, Temas Varios), con 7 conexiones documentando relaciones reales entre proyectos (ej. ProvDent → ProvDent-limpio "reinicio limpio", Busca Medicamentos → Busca Medicamentos Web "reescritura para producción"). Requirió dos iteraciones de layout: la primera versión tenía grupos muy pegados con líneas cruzadas; se separaron los grupos (160px de gap) y se re-enrutaron las conexiones del trío ProvDent (de lado-a-lado a arriba/abajo) porque las etiquetas de texto quedaban tapadas detrás de las tarjetas vecinas cuando las columnas estaban muy juntas.
- **Obsidian CLI habilitado** (Settings → General → Advanced → Command line interface). Permite automatizar el vault desde la terminal (`obsidian open`, `obsidian eval`, `obsidian dev:screenshot`), útil para revisar cambios en el canvas/notas sin depender de capturas de pantalla del sistema (que no funcionan en este Mac — ver [[Organización del Vault#Limitación de macOS|abajo]]).
- **Prefix de npm arreglado**: `npm config set prefix ~/.npm-global` + PATH en `~/.zprofile`, para que `npm install -g <paquete>` no falle por permisos de `/usr/local`. Verificado instalando `defuddle` globalmente.

### Limitación de macOS

Este MacBook Air (2015, macOS 12.6 Monterey) no soporta `ScreenCaptureKit`, por lo que la herramienta de captura de pantalla de `computer-use` falla ahí. Workaround: usar `obsidian dev:screenshot` (vía Obsidian CLI) para revisar visualmente cambios en el vault — no depende de ScreenCaptureKit. La misma limitación de hardware ya bloqueaba `wrangler dev` (ver nota de `busca-medicamentos-web`); no es solucionable sin actualizar el equipo, decisión que Victor ya descartó tomar.
