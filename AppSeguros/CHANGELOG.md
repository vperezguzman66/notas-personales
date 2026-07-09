# Changelog - Gestor de Seguros

Registro de cambios realizados en el desarrollo de la aplicación. Espejo de `docs/CHANGELOG.md` del repo.

## [Sin publicar]

### Nuevas categorías de seguro — 2026-07-08
Agregadas dos categorías a `CATEGORY_LABELS` (`src/utils/insurance.js`), a pedido del usuario: **Seguro de Vida** (`vida`) y **Responsabilidad Civil** (`rc`). Cada una con su propio color de badge (`--vida-color` dorado, `--rc-color` violeta en `src/styles/base.css`, mapeados en `CATEGORY_COLORS` de `InsuranceCard.jsx`). Al estar centralizadas en `CATEGORY_LABELS`/`CATEGORY_OPTIONS`, aparecen automáticamente en el filtro, el selector del formulario y el badge de la tarjeta sin tocar más de un archivo por punto de integración. Verificado con `oxlint`, `npm run build` y prueba end-to-end en navegador.

Se descartó agregar "Viaje" (no relevante para el usuario) y se dejó pendiente "Responsabilidad Civil Profesional" (médicos/odontólogos) para una futura iteración — ver [[roadmap_categorias_seguro]] en la memoria del proyecto.

### Modularización — 2026-07-08
Revisión de tamaño/estructura de archivos a pedido explícito ("no quiero programas con exceso de líneas"). `App.jsx` (311 líneas), `InsuranceForm.jsx` (256), `ProfileSwitcher.jsx` (163) e `index.css` (739) se dividieron en módulos más chicos y cohesivos. Verificado con `oxlint`, `npm run build` (CSS resultante idéntico en tamaño: 10.94 kB) y pruebas end-to-end en navegador (agregar/editar/borrar seguro, crear/borrar perfil).

- **Código muerto eliminado:** `src/App.css` (boilerplate del scaffold de Vite — clases `.counter`/`.hero`/`#next-steps`, nunca importado), `src/assets/react.svg`, `src/assets/vite.svg`. `src/assets/hero.png` y `public/icons.svg` también aparecen sin uso pero se dejaron para revisión manual (podrían ser para algo planeado).
- **`App.jsx`** (311 → ~150 líneas): datos semilla movidos a `src/data/seedData.js`; lógica de persistencia extraída a los hooks `src/hooks/usePersistedJSON.js` y `src/hooks/useActiveProfileId.js`; banner de error de storage extraído a `src/components/StorageErrorBanner.jsx`; buscador/filtro/grid de seguros extraído a `src/components/MainContent.jsx`.
- **`InsuranceForm.jsx`** (256 → 181 líneas): lista de aseguradoras movida a `src/data/insurers.js`; estado de campos, valores por defecto según tipo y validación extraídos al hook `src/hooks/useInsuranceFormFields.js`.
- **`ProfileSwitcher.jsx`** (163 → 70 líneas): modal de "Nuevo Perfil" extraído a `src/components/NewProfileModal.jsx`.
- **`index.css`** (739 → 10 líneas de `@import`): dividido en 8 partials bajo `src/styles/` (`base`, `layout`, `profile-switcher`, `dashboard`, `insurance-list`, `modal-form`, `emergency`, `animations`), en el mismo orden para no alterar el cascade.

### Fix UX: scroll interno en modales — 2026-07-08
`.modal-content` (`src/styles/modal-form.css`) no tenía límite de altura ni scroll propio; en viewports bajos (probado en 500×480) el contenido del formulario desbordaba la ventana y los botones "Guardar"/"Cancelar" quedaban fuera de la vista y sin forma de alcanzarlos. Se agregó `padding: 1.5rem` a `.modal-overlay` y `max-height: 100%` + `overflow-y: auto` a `.modal-content`. Verificado con `oxlint`, `npm run build` y prueba end-to-end en navegador con viewport reducido.

### Segunda revisión de código — 2026-07-08 (post-fix)
Revisión de correctitud sobre el estado actual del código (`/code-review`, effort high), después de corregir la primera tanda de hallazgos y subir el repo a GitHub. 10 hallazgos, detalle completo en [[revision_codigo_2026-07-08]]. Los 10 corregidos, en dos tandas.

- **Corregidos (correctitud):** `activeProfileId` ahora se valida contra los perfiles reales al iniciar y se autocorrige si queda huérfano; `loadJSON` valida la forma del valor parseado con un predicado (`Array.isArray`); nueva función `loadString`; `saveJSON`/`saveString` devuelven éxito/fracaso y `App.jsx` muestra un banner visible cuando falla el guardado; `formatDate` retorna "Fecha inválida" en vez del "Invalid Date" nativo en inglés; precio `0` ya no se muestra como campo vacío al editar (`?? ''` en vez de `|| ''`); la guarda de `handleDeleteProfile` ahora verifica el resultado real del filtro en vez de contar perfiles de antemano.
- **Corregidos (consistencia/limpieza):** `CATEGORY_LABELS`/`CATEGORY_OPTIONS` centralizados en `src/utils/insurance.js`; eliminado `formatUF` (código muerto en `Dashboard.jsx`); relectura redundante de `localStorage` resuelta como efecto colateral del fix de `activeProfileId`.

### Revisión de código — 2026-07-08
Revisión de correctitud del primer commit (`/code-review`). 10 hallazgos, detalle completo en [[revision_codigo_2026-07-08]]. Los 10 corregidos.

- **Bugs corregidos:** error de zona horaria en el cálculo de días restantes (afecta alertas de vencimiento), pagos "Pago Único" sumados como gasto recurrente, `JSON.parse` sin protección sobre `localStorage` (`src/utils/storage.js`), precio negativo sin validar, sin validar orden de fechas inicio/vencimiento, fecha de vencimiento inválida ya no se muestra como "Vigente", colisión de IDs por doble clic al guardar (IDs ahora vía `crypto.randomUUID`), `handleDeleteProfile` ahora tiene su propia guarda contra dejar cero perfiles, `localStorage.setItem` maneja `QuotaExceededError` sin tumbar la app.
- **Limpieza:** `formatCLP`, `formatDate`, `getDaysRemaining` y `generateId` centralizadas en `src/utils/insurance.js`.
- **Fix adicional detectado durante la corrección:** el mensaje de validación del formulario (`formError`) se calculaba pero nunca se renderizaba. Ahora se muestra en `InsuranceForm.jsx`.

## [0.1.0] - 2026-07-08

### Añadido
- **Estructura del Proyecto:** Configuración de React y Vite como base del desarrollo.
- **Diseño Visual Premium (`src/index.css`):**
  - Paleta de colores HSL refinada con modo oscuro por defecto.
  - Efecto de desenfoque de fondo (*glassmorphism*) en paneles y tarjetas.
  - Animaciones y micro-interacciones (hover, pulsaciones rojas en seguros por vencer).
- **Gestor de Perfiles (`components/ProfileSwitcher.jsx`):**
  - Soporte para múltiples personas en el mismo navegador.
  - Selección de colores e iconos (emojis) personalizados para cada perfil.
  - Persistencia total y sincronización del estado mediante `localStorage`.
- **Dashboard de Control (`components/Dashboard.jsx`):**
  - Cálculo de gasto mensual y anual en tiempo real.
  - Soporte para conversión y visualización combinada de CLP ($) y UF (Unidad de Fomento).
  - Panel indicador de vigencia: activos, advertencias (vencimiento < 30 días) y vencidos.
- **Tarjetas de Seguro (`components/InsuranceCard.jsx`):**
  - Tarjetas personalizadas por categoría (SOAP, Hogar, Auto, Salud, Fraude).
  - Alertas visuales dinámicas (semáforo de vencimiento).
- **Formulario de Registro (`components/InsuranceForm.jsx`):**
  - Autocompletado de aseguradoras populares en Chile (BCI, SURA, Consorcio, etc.).
  - Configuración automática de fechas y plazos típicos (ej. SOAP del 1 de abril al 31 de marzo).
- **Contactos de Emergencia (`components/EmergencyContacts.jsx`):**
  - Números rápidos de asistencia directa de aseguradoras chilenas.
- **SEO & Metadatos (`index.html`):**
  - Configuración del idioma en español, meta descripción para SEO y título descriptivo.
