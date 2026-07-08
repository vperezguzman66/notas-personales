# Plan de Implementación: Aplicación de Centralización de Seguros Personales (Multiusuario)

Una aplicación web interactiva desarrollada con **Vite + React** para centralizar, visualizar y gestionar los seguros personales de múltiples personas (ej. miembros de la familia o perfiles individuales) en un solo lugar. La información se persistirá de forma local en el navegador (`localStorage`).

## Resoluciones del Usuario
- **Framework:** Vite + React.
- **Alcance:** Soporte para múltiples perfiles de usuario.
- **Alojamiento de Documentación:** `/Users/victor/Obsidian/nameVault/AppSeguros/` y `/Users/victor/Proyectos/gestor-seguros/docs/`.

## Cambios Propuestos

El desarrollo se realizará en la ruta local `/Users/victor/Proyectos/gestor-seguros`.

### Estructura de Componentes en React
1. **`App.jsx`:** Componente principal. Gestiona el estado de los perfiles, el perfil activo actual, y las vistas de la aplicación.
2. **`components/ProfileSwitcher.jsx`:** Permite crear, editar, eliminar y cambiar de perfil de usuario. Cada perfil tendrá un color y avatar descriptivo.
3. **`components/Dashboard.jsx`:** Muestra estadísticas generales:
   - Gasto mensual y anual del perfil activo (y consolidado general).
   - Contador de seguros vigentes, vencidos y por vencer.
   - Gráfico de gastos por categoría (Salud, Vehículo, Hogar, Fraude, etc.).
4. **`components/InsuranceCard.jsx`:** Tarjeta individual para cada seguro que muestra aseguradora, póliza, costo, vigencia y alertas visuales.
5. **`components/InsuranceForm.jsx`:** Formulario modal para agregar o editar pólizas de seguro.
6. **`components/EmergencyContacts.jsx`:** Lista rápida de teléfonos de asistencia de las principales aseguradoras chilenas (SOAP, Hogar, etc.).

### Archivos Nuevos

#### [NEW] [package.json](file:///Users/victor/Proyectos/gestor-seguros/package.json)
Configuración de dependencias de React y Vite.

#### [NEW] [src/main.jsx](file:///Users/victor/Proyectos/gestor-seguros/src/main.jsx)
Punto de entrada de la aplicación React.

#### [NEW] [src/App.jsx](file:///Users/victor/Proyectos/gestor-seguros/src/App.jsx)
Estado global y enrutamiento interno.

#### [NEW] [src/index.css](file:///Users/victor/Proyectos/gestor-seguros/src/index.css)
CSS Vanilla moderno:
- Variables CSS para colores HSL, bordes redondeados y efectos glassmorphism.
- Estilo premium responsivo con soporte para modo oscuro por defecto.
- Animaciones fluidas para el cambio de perfiles y carga de tarjetas.

#### [NEW] [docs/CHANGELOG.md](file:///Users/victor/Proyectos/gestor-seguros/docs/CHANGELOG.md)
Registro detallado de cambios realizados que se actualizará y sincronizará con Obsidian.

## Plan de Verificación

### Pruebas Manuales
1. **Gestión de Perfiles:** Crear perfiles para "Juan" y "María", verificar que se puede cambiar entre ellos y que cada uno mantiene sus propios seguros.
2. **Alertas de Vencimiento:** Registrar un seguro con vencimiento a menos de 30 días (ej. SOAP) y comprobar que se muestra la alerta en color amarillo o rojo según corresponda.
3. **Consistencia de Datos:** Agregar seguros en un perfil, recargar la página y verificar que se mantienen guardados.
4. **Cálculos:** Verificar que las estadísticas de gastos mensuales y anuales coincidan con los seguros ingresados.
