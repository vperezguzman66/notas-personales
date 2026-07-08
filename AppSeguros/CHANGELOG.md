# Changelog - Gestor de Seguros

Registro de cambios realizados en el desarrollo de la aplicación.

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
