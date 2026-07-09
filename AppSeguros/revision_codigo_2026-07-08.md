# Revisión de Código — 2026-07-08

Revisión de correctitud (`/code-review`, effort high) sobre el primer commit del proyecto `/Users/victor/Proyectos/gestor-seguros`. Se generaron 8 ángulos de búsqueda independientes y se verificó cada hallazgo contra el código real antes de reportarlo. 10 hallazgos sobrevivieron la verificación, ordenados de más a menos severo.

## Bugs de correctitud

### 1. Error de zona horaria en `getDaysRemaining` — CONFIRMADO
📍 `src/components/InsuranceCard.jsx:47` (duplicado en `src/components/Dashboard.jsx:24`)

`new Date('2027-03-31')` se interpreta como medianoche UTC; luego `end.setHours(0,0,0,0)` reinterpreta ese instante en hora local (UTC-3/-4 en Chile), retrocediéndolo a la medianoche local del 30 de marzo. Una póliza SOAP que vence el 31 de marzo se trata como si venciera el 30 — la insignia de vencimiento cambia a "Vence Hoy"/"Vencido" un día antes. El mismo bug está duplicado en `Dashboard.jsx` y afecta los contadores de la barra lateral (Vigentes/Vencen pronto/Vencidos).

Ya existe la corrección en el mismo archivo: `formatDate` (línea 76) agrega `'T00:00:00'` justamente para evitar este problema, pero `getDaysRemaining` no la usa.

### 2. Pagos únicos tratados como gasto recurrente — CONFIRMADO
📍 `src/components/Dashboard.jsx:41`

`calculateCosts` solo trata especialmente `paymentFrequency === 'anual'`; un pago "Pago Único" (opción real del formulario, `InsuranceForm.jsx:208`) cae en la rama por defecto y se suma como si fuera mensual. Un cargo único de $50.000 CLP se muestra como $50.000/mes y $600.000/año **para siempre** en "Resumen de Gastos".

### 3. `JSON.parse` sin protección sobre localStorage — CONFIRMADO
📍 `src/App.jsx:72` (también líneas 79 y 85)

No hay try/catch ni error boundary en ningún lugar del árbol. Si `gestor_seguros_profiles` o `gestor_seguros_insurances` contiene JSON corrupto (edición manual, escritura parcial por un crash, futuro cambio de esquema), `JSON.parse` lanza excepción dentro del inicializador de `useState` → pantalla blanca sin recuperación.

### 4. Fecha de vencimiento inválida se reporta como "Vigente" — PLAUSIBLE
📍 `src/components/InsuranceCard.jsx:56`

Si `endDate` está vacío o mal formado, `getDaysRemaining` devuelve `NaN`. Ninguna comparación (`NaN<0`, `NaN===0`, etc.) es verdadera, cae al `else` final y muestra "✓ Vigente" — justo lo opuesto de lo que necesita una app de alertas de vencimiento.

### 5. Precio negativo sin validación — CONFIRMADO
📍 `src/components/InsuranceForm.jsx:88`

El input de precio es `type="number" step="any"` sin `min`. `parseFloat('-500') || 0` deja pasar el valor negativo tal cual, corrompiendo silenciosamente los totales de costos del Dashboard.

### 6. Sin validar que `startDate` sea anterior a `endDate` — CONFIRMADO
📍 `src/components/InsuranceForm.jsx:78`

`handleSubmit` solo valida `insurer` y `endDate`. Se puede guardar con fecha de inicio vacía o posterior a la de vencimiento, sin error.

### 7. Colisión de IDs por doble clic en "Guardar" — PLAUSIBLE
📍 `src/components/InsuranceForm.jsx:81` (mismo patrón en `ProfileSwitcher.jsx:31`)

El ID se genera con `Date.now().toString()` sin deshabilitar el botón tras el envío. Un doble clic rápido puede generar dos registros con el mismo ID — borrar uno los borra ambos, y las `key={ins.id}` colisionan en el render.

### 8. `handleDeleteProfile` sin protección propia contra dejar cero perfiles — PLAUSIBLE
📍 `src/App.jsx:139`

La única protección es a nivel de UI (`ProfileSwitcher.jsx:60` oculta el botón si `profiles.length <= 1`). Si algo más invoca `onDeleteProfile` con un solo perfil, `activeProfileId` queda apuntando a un perfil inexistente y la app sustituye silenciosamente el perfil demo "Juan".

### 9. `localStorage.setItem` sin manejo de errores — PLAUSIBLE
📍 `src/App.jsx:95`

Si se excede la cuota de almacenamiento, `setItem` lanza `QuotaExceededError` dentro del `useEffect`. El estado en memoria ya muestra el cambio como "guardado" (UI optimista), pero no se persiste — al refrescar, la última edición desaparece sin aviso.

## Limpieza

### 10. `formatCLP` y `getDaysRemaining` duplicadas literalmente — CONFIRMADO
📍 `src/components/InsuranceCard.jsx:36` (y `Dashboard.jsx`)

Copiadas y pegadas entre los dos archivos. El bug de zona horaria (#1) o cualquier cambio futuro de formato requiere corregir dos lugares; si se corrige solo uno, la tarjeta y el panel lateral quedarán en desacuerdo sobre el estado de la misma póliza.

## Estado

Los 10 hallazgos fueron corregidos (2026-07-08). Resumen de la corrección:

- **#1, #10:** `formatCLP`, `formatDate`, `getDaysRemaining` y `generateId` centralizadas en `src/utils/insurance.js`; `getDaysRemaining` ahora parsea con `T00:00:00` antes de `setHours`, eliminando el corrimiento de zona horaria.
- **#2:** `calculateCosts` en `Dashboard.jsx` ignora explícitamente `paymentFrequency === 'único'`.
- **#3, #9:** nuevas envolturas `loadJSON`/`saveJSON`/`saveString` en `src/utils/storage.js`; `App.jsx` las usa para los tres campos persistidos en `localStorage`, con try/catch en lectura y escritura.
- **#4:** `getDaysRemaining` devuelve `NaN` si la fecha está vacía o mal formada; `InsuranceCard.jsx` y `Dashboard.jsx` chequean `Number.isNaN` explícitamente y muestran "Fecha inválida" en vez de "Vigente".
- **#5, #6:** `InsuranceForm.jsx` valida precio > 0, fechas de inicio/vencimiento presentes y `startDate <= endDate` antes de guardar.
- **#7:** `generateId()` usa `crypto.randomUUID()` (con fallback), reemplazando `Date.now().toString()` en `InsuranceForm.jsx` y `ProfileSwitcher.jsx`.
- **#8:** `handleDeleteProfile` en `App.jsx` verifica `profiles.length <= 1` como guarda propia, sin depender de que `ProfileSwitcher` oculte el botón.

**Hallazgo adicional detectado durante la corrección (no en la lista original):** el estado `formError` de `InsuranceForm.jsx` se calculaba en `handleSubmit` pero nunca se renderizaba en el JSX — las validaciones de #5/#6 bloqueaban el guardado sin mostrar ningún mensaje al usuario. Corregido agregando el bloque de error antes de `form-actions`.

---

## Segunda revisión de código — 2026-07-08 (post-fix, tras push a GitHub)

Tras corregir los 10 hallazgos anteriores y subir el repo a `github.com/vperezguzman66/gestor-seguros` (privado), se ejecutó una segunda revisión (`/code-review`, effort high) sobre el estado actual completo del código (un solo commit raíz, sin diff previo). 8 ángulos de búsqueda independientes, verificación 1-voto por hallazgo. 10 nuevos hallazgos sobrevivieron, ordenados de más a menos severo. Ninguno corregido aún.

### Bugs de correctitud

#### 1. `activeProfileId` huérfano no se valida contra `profiles` — CONFIRMADO
📍 `src/App.jsx:75`

El inicializador de `activeProfileId` confía en el valor guardado en `localStorage` sin comprobar que ese perfil siga existiendo en `profiles`. Si el id guardado ya no existe (perfil borrado desde otra pestaña, o storage editado a mano), `activeProfile` cae a `profiles[0]` para mostrarse (línea 105), pero `activeProfileInsurances` (línea 108) y todo lo derivado (lista filtrada, contadores del Dashboard) siguen filtrando por el id obsoleto y quedan vacíos — ninguna pestaña de `ProfileSwitcher` aparece activa. Si el usuario crea un seguro nuevo en ese estado, `InsuranceForm` lo guarda con ese `activeProfileId` huérfano: el registro persiste en `localStorage` pero nunca volverá a coincidir con el filtro de ningún perfil real — pérdida de datos silenciosa y permanente.

#### 2. `loadJSON` no valida la forma del valor parseado, solo que no lance excepción — CONFIRMADO
📍 `src/utils/storage.js:4`

Si `localStorage.getItem('gestor_seguros_profiles')` devuelve el string `"null"` o `"{}"`, `JSON.parse` no lanza excepción, así que `loadJSON` retorna `null` o `{}` en vez de un array. `App.jsx` llama luego `profiles.find(...)` (línea 105) o `ProfileSwitcher.jsx` llama `profiles.map(...)` (línea 50) sobre algo que no es array, y la app cae en pantalla blanca — exactamente lo que el comentario de cabecera de `storage.js` dice que evita, porque esa garantía solo cubre errores de parseo, no de forma.

#### 3. Lectura directa de `localStorage.getItem` sin pasar por el wrapper seguro — CONFIRMADO
📍 `src/App.jsx:76`

`storage.js` expone `loadJSON`/`saveJSON`/`saveString` justo para que un acceso a `localStorage` que lance excepción nunca tumbe la app, y el lado de escritura ya usa `saveString` (línea 97), pero la lectura inicial de `activeProfileId` llama `localStorage.getItem` directo. En cualquier entorno donde `getItem` lance de forma síncrona (modo privado de Safari en casos límite, storage deshabilitado por política de navegador/empresa, iframe sandboxed), esto revienta sin capturar dentro del inicializador de `useState`, antes de que React pueda renderizar — reproduciendo el mismo modo de falla (pantalla blanca) que la corrección de `JSON.parse` (hallazgo #3 de la primera revisión) buscaba eliminar, solo que por otro disparador. Tampoco existe un `loadString` equivalente a `saveString`.

#### 4. `saveJSON`/`saveString` solo hacen `console.error` en fallos de escritura, nunca avisan al usuario — CONFIRMADO
📍 `src/utils/storage.js:15`

El bug original reportado en la primera revisión (#9) era: ante `QuotaExceededError`, la UI ya muestra el cambio como guardado (estado optimista en memoria) pero no persiste, y desaparece sin aviso al refrescar. La corrección actual evita que la app se caiga, pero no resuelve el síntoma que el usuario realmente sufre — `console.error` es invisible para un usuario final, y no existe ningún `toast`/`alert`/estado de error en ningún componente (confirmado por grep). Un usuario que llena la cuota de `localStorage` sigue perdiendo su seguro o perfil nuevo al recargar, sin ninguna indicación de que algo falló.

#### 5. `formatDate` no tiene guarda para fecha inválida, a diferencia del badge de vencimiento adyacente — CONFIRMADO
📍 `src/components/InsuranceCard.jsx:114`

Para un `endDate` mal formado pero no vacío (alcanzable vía `localStorage` corrupto/editado a mano — el mismo escenario que motivó el fix de NaN), `getDaysRemaining` retorna `NaN` correctamente y el badge muestra "⚠️ Fecha inválida" (línea 40), pero el campo adyacente en la línea 114 llama `formatDate(endDate)`, que no tiene guarda de `NaN`/fecha inválida en `src/utils/insurance.js` y renderiza el string crudo en inglés "Invalid Date" justo al lado del badge en español — una inconsistencia que el fix de NaN de la primera revisión no cubrió porque solo tocó los call sites de `getDaysRemaining`.

#### 6. `setPrice(insuranceToEdit.price || '')` trata un precio legítimo de 0 como vacío — PLAUSIBLE
📍 `src/components/InsuranceForm.jsx:44`

Si algún registro llega a tener `price: 0` (p. ej. una póliza pagada por el empleador, ingresada vía edición directa de datos, importación, o una futura flexibilización de la regla `price > 0` al guardar), abrir ese registro para editar muestra el campo de precio vacío en vez de `0`. Como un campo vacío ahora falla la validación `price > 0` agregada en este mismo archivo, el usuario queda forzado a escribir un valor distinto de cero solo para volver a guardar un registro que ya era válido.

#### 7. La guarda de `handleDeleteProfile` cuenta perfiles totales, no si borrar ese id específico deja el array vacío — PLAUSIBLE
📍 `src/App.jsx:131`

La guarda `if (profiles.length <= 1) return;` asume que todos los ids de perfil son únicos. Si `profiles` llegara a tener dos entradas con el mismo id (exactamente lo que el esquema pre-fix `Date.now().toString()` en `ProfileSwitcher` pudo haber producido antes de introducir `crypto.randomUUID`, y nada en el código migra/deduplica datos existentes de `localStorage` al cargar), `profiles.length` sería 2, la guarda pasaría, y `profiles.filter(p => p.id !== profileId)` (línea 133) eliminaría ambas entradas coincidentes, dejando `remainingProfiles.length === 0`. La línea 142, `setActiveProfileId(remainingProfiles[0].id)`, lanzaría entonces sobre `undefined` — el mismo estado de "cero perfiles" que la guarda debía prevenir.

### Consistencia

#### 8. Las etiquetas de tipo de seguro están triplicadas y ya desincronizadas — CONFIRMADO
📍 `src/App.jsx:219`

El dropdown de filtro en `App.jsx` tiene hardcodeado `<option value="soap">SOAP</option>` y `<option value="otro">Otros</option>`, mientras que los mismos tipos se etiquetan "SOAP (Auto Obligatorio)" y "Otro Seguro" en `CATEGORY_LABELS` de `InsuranceCard.jsx` (usado en cada badge de tarjeta) y se duplican otra vez en la lista de `<option>` de `InsuranceForm.jsx`. Un usuario que filtra por "SOAP" en el dropdown ve tarjetas con la insignia "SOAP (Auto Obligatorio)" — la etiqueta ya no coincide entre el control de filtro y la tarjeta que filtra, porque no existe una única fuente compartida (p. ej. `CATEGORY_LABELS` exportado desde `src/utils/insurance.js`) para este mapeo tipo→etiqueta.

### Limpieza

#### 9. `formatUF` está definida y se recrea en cada render pero nunca se llama — CONFIRMADO
📍 `src/components/Dashboard.jsx:8`

Código muerto: un grep sobre el archivo solo encuentra la definición, ningún call site — cada render asigna un closure sin uso.

#### 10. Relectura redundante de `localStorage` al montar — CONFIRMADO
📍 `src/App.jsx:78`

El inicializador de `activeProfileId` vuelve a llamar `loadJSON('gestor_seguros_profiles', DEFAULT_PROFILES)`, la misma clave que el inicializador de `profiles` (líneas 71-73) ya calculó una línea antes, duplicando el trabajo de `JSON.parse` en cada montaje y creando dos caminos de código que podrían divergir silenciosamente si alguna vez usan fallbacks distintos.

## Estado (segunda revisión)

Corregidos #1–#5 (2026-07-08). Verificado con `npm run build` y en navegador (Chrome DevTools) inyectando datos corruptos en `localStorage`. Pendientes: #6, #7 (plausibles, baja probabilidad hoy) y #8–#10 (consistencia/limpieza).

- **#1:** `App.jsx` ahora valida `activeProfileId` contra `profiles` reales al inicializar (en vez de confiar ciegamente en `localStorage`), y un `useEffect` se autocorrige al primer perfil válido si `activeProfileId` alguna vez queda huérfano. Probado inyectando un id inexistente y recargando: la app se autocorrigió a "Juan" en vez de quedar con el dashboard/lista vacíos.
- **#2:** `loadJSON` (`src/utils/storage.js`) ahora acepta un validador de forma (`isValid`); `App.jsx` le pasa `Array.isArray` para `profiles`/`insurances`, así que un JSON válido pero mal formado (`"null"`, `"{}"`) cae al fallback en vez de romper `.find`/`.map` más adelante.
- **#3:** nueva función `loadString` en `storage.js` (con su propio try/catch); `App.jsx` la usa para leer `activeProfileId` en vez de `localStorage.getItem` directo.
- **#4:** `saveJSON`/`saveString` ahora retornan `true`/`false`; `App.jsx` mantiene un estado `storageError` que muestra un banner visible ("No se pudieron guardar los últimos cambios...") cuando cualquier guardado falla, en vez de fallar en silencio.
- **#5:** `formatDate` (`src/utils/insurance.js`) retorna `'Fecha inválida'` si la fecha no parsea, en vez de dejar pasar el `"Invalid Date"` nativo en inglés. Probado inyectando un `endDate` corrupto: el campo "Vencimiento" ahora coincide en idioma y mensaje con el badge de la tarjeta.
