# Changelog - Gestor de Seguros

Registro de cambios realizados en el desarrollo de la aplicación.

## [Sin publicar]

### Revisión de código — 2026-07-08
Revisión de correctitud del primer commit (`/code-review`). 10 hallazgos, ver [[revision_codigo_2026-07-08]] para el detalle completo. Ninguno corregido aún.

- **Bugs confirmados a corregir:** error de zona horaria en el cálculo de días restantes (afecta alertas de vencimiento), pagos "Pago Único" sumados como gasto recurrente, `JSON.parse` sin protección sobre `localStorage`, precio negativo sin validar, sin validar orden de fechas inicio/vencimiento.
- **Bugs plausibles:** fecha de vencimiento inválida se muestra como "Vigente", colisión de IDs por doble clic al guardar, `handleDeleteProfile` depende solo de una guarda de UI, `localStorage.setItem` sin manejo de `QuotaExceededError`.
- **Limpieza:** `formatCLP` y `getDaysRemaining` duplicadas entre `InsuranceCard.jsx` y `Dashboard.jsx`.

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
