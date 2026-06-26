[[Solicitudes de Software]]

App para buscar texto en código público de **GitHub y GitLab**, agrupar los resultados por lenguaje/extensión y guardarlos como `.txt`. El frontend está hecho en **React + Vite** y habla con un **proxy backend en Express** que guarda y usa los tokens en el servidor, nunca en el navegador. Se puede ejecutar como web local o como **app de escritorio** con Electron para macOS, Windows y Linux.

## Tabla de contenidos

[](https://github.com/vperezguzman66/buscagit#tabla-de-contenidos)

- [Arquitectura](https://github.com/vperezguzman66/buscagit#arquitectura)
- [Puesta en marcha](https://github.com/vperezguzman66/buscagit#puesta-en-marcha)
- [Variables de entorno](https://github.com/vperezguzman66/buscagit#variables-de-entorno)
- [Cómo usar](https://github.com/vperezguzman66/buscagit#c%C3%B3mo-usar)
- [Scripts](https://github.com/vperezguzman66/buscagit#scripts)
- [Desarrollo y calidad](https://github.com/vperezguzman66/buscagit#desarrollo-y-calidad)
- [App de escritorio](https://github.com/vperezguzman66/buscagit#app-de-escritorio-electron)
- [Endpoints del proxy](https://github.com/vperezguzman66/buscagit#endpoints-del-proxy)
- [Estructura del proyecto](https://github.com/vperezguzman66/buscagit#estructura-del-proyecto)
- [GitLab](https://github.com/vperezguzman66/buscagit#buscar-en-gitlab)
- [Notas de la API de GitHub](https://github.com/vperezguzman66/buscagit#notas-sobre-la-api-de-github)
- [Solución de problemas](https://github.com/vperezguzman66/buscagit#soluci%C3%B3n-de-problemas)

## Arquitectura

[](https://github.com/vperezguzman66/buscagit#arquitectura)

```
            modo DEV (web)                         modo APP (escritorio)
  ┌──────────────────────────────┐      ┌────────────────────────────────────┐
  │ React → Vite (5173)          │      │  Electron (1 proceso)              │
  │   └─/api─► Express (8787)     │      │   React  ─/api─►  Express embebido │
  └───────────────┬──────────────┘      └─────────────────┬──────────────────┘
                  │  (mismo proxy Express)                 │
                  ▼                                        ▼
        api.github.com  ·  gitlab.com/api/v4   (token añadido por el proxy)
```

- El frontend siempre llama a `/api/*`; el proxy añade el token y reenvía a GitHub/GitLab. **El token nunca llega al navegador.**
- Las llamadas HTTP del frontend están centralizadas en `src/api/`, para que hooks y utilidades no repitan `fetch()` ni parsing de respuestas.
- **Precedencia del token** (de mayor a menor): cabecera `X-Override-Token` (token pegado en la UI, solo esa sesión) → token guardado en Ajustes → `GITHUB_TOKEN` / `GITLAB_TOKEN` del `.env` (modo dev).
- El proxy local rechaza orígenes no permitidos y, para escrituras, valida el `Content-Type` y el esquema JSON antes de aceptar datos.
- En modo dev el proxy escucha solo en `127.0.0.1`, no en todas las interfaces.
- En la app de escritorio, ese único proceso de Electron levanta el Express en memoria y también sirve el frontend ya construido.

## Puesta en marcha

[](https://github.com/vperezguzman66/buscagit#puesta-en-marcha)

1. Copia el ejemplo de entorno y pon tu token:
    
    ```shell
    cp .env.example .env
    # edita .env y rellena GITHUB_TOKEN=...
    ```
    

# opcional: define BUSCAGIT_ALLOWED_ORIGINS si cambias el origen web local

[](https://github.com/vperezguzman66/buscagit#opcional-define-buscagit_allowed_origins-si-cambias-el-origen-web-local)

# Token (sin permisos especiales para código público): [https://github.com/settings/tokens?type=beta](https://github.com/settings/tokens?type=beta)

[](https://github.com/vperezguzman66/buscagit#token-sin-permisos-especiales-para-c%C3%B3digo-p%C3%BAblico-httpsgithubcomsettingstokenstypebeta)

````

2. Instala dependencias y arranca frontend + backend juntos:

```bash
npm install
npm run dev
````

- Frontend: [http://localhost:5173](http://localhost:5173/)
- Proxy: [http://localhost:8787](http://localhost:8787/) (salud: `/api/health`)

## Variables de entorno

[](https://github.com/vperezguzman66/buscagit#variables-de-entorno)

|Variable|Qué hace|
|---|---|
|`GITHUB_TOKEN`|Token para GitHub en modo dev.|
|`GITLAB_TOKEN`|Token para GitLab en modo dev.|
|`PORT`|Puerto del proxy en dev (`8787` por defecto).|
|`BUSCAGIT_ALLOWED_ORIGINS`|Lista separada por comas de orígenes permitidos para el proxy local. Útil si cambias el puerto/origen del frontend.|

## Cómo usar

[](https://github.com/vperezguzman66/buscagit#c%C3%B3mo-usar)

1. **Configura tus tokens** (una vez):
    
    - En la app de escritorio: botón **⚙ Ajustes** → pega tu token de GitHub y/o GitLab → _Guardar_. Quedan cifrados en tu equipo.
    - En modo web/dev: ponlos en `.env` (`GITHUB_TOKEN`, `GITLAB_TOKEN`) o pégalo en el campo _Token_ del formulario para usarlo solo en esa sesión. El guardado persistente de Ajustes no está disponible sin cifrado nativo.
2. **Elige la fuente**: chip **GitHub** o **GitLab** en lo alto del formulario.
    
3. **Escribe el texto a buscar** (ej. `api_key`, `connection_string`, un dominio…).
    
4. **Acota la búsqueda según la fuente**:
    
    - **GitHub** → marca los **lenguajes** a incluir (Python, Go, …). Se hace una búsqueda por lenguaje y se agrupan los resultados.
    - **GitLab** → indica un **grupo** (`gitlab-org`) o **proyecto** (`gitlab-org/gitlab`); GitLab no busca código globalmente (ver más abajo). Los resultados se agrupan por **extensión de archivo**.
5. **Pulsa Buscar.** Verás el progreso y, al terminar, tarjetas por lenguaje con el número de URLs únicas. Cada URL enlaza al archivo en GitHub/GitLab.
    
6. **Guarda los resultados** como `.txt`:
    
    - _Guardar todo en reportes_ → un archivo con todas las URLs deduplicadas.
    - _Guardar en reportes_ (por tarjeta) → solo ese lenguaje.
    - Los GitLab llevan prefijo `gitlab_` para no pisar los de GitHub.
    - Carpeta: `reportes/` (modo dev) o `~/Documents/BuscaGit/reportes` (app).

> **Límites:** GitHub topa en **1000 resultados** por consulta y la búsqueda de código tiene un rate limit de ~30 req/min autenticado. La barra de la cabecera muestra las peticiones restantes.

## Scripts

[](https://github.com/vperezguzman66/buscagit#scripts)

|Comando|Qué hace|
|---|---|
|`npm run dev`|Levanta proxy + Vite a la vez (concurrently).|
|`npm run dev:server`|Solo el proxy Express.|
|`npm run dev:client`|Solo el frontend Vite.|
|`npm run build`|Build de producción del frontend.|
|`npm start`|Solo el proxy (para servir el build detrás de él).|
|`npm run test`|Ejecuta los tests base del backend con `node:test`.|
|`npm run lint`|Revisa el código con ESLint.|
|`npm run lint:fix`|Corrige automáticamente lo que pueda ESLint.|
|`npm run format`|Formatea el repo con Prettier.|
|`npm run format:check`|Comprueba que el formato cumple Prettier.|
|`npm run dist`|Empaqueta la app de escritorio para tu SO actual.|
|`npm run dist:mac`|Genera `.dmg` + `.zip` (macOS).|
|`npm run dist:win`|Genera instalador `.exe` NSIS (Windows).|
|`npm run dist:linux`|Genera `AppImage` + `.deb` (Linux).|

## Desarrollo y calidad

[](https://github.com/vperezguzman66/buscagit#desarrollo-y-calidad)

Antes de abrir PRs o preparar releases, estas son las comprobaciones recomendadas:

```shell
npm test
npm run lint
npm run format:check
```

- `npm test` valida el proxy con requests reales a un servidor efímero.
- `npm run lint` revisa React, hooks y JavaScript general.
- `npm run format:check` asegura que Prettier no detecta cambios pendientes.

## App de escritorio (Electron)

[](https://github.com/vperezguzman66/buscagit#app-de-escritorio-electron)

La app se puede empaquetar como ejecutable nativo para macOS, Windows y Linux con [electron-builder](https://www.electron.build/). Un único proceso de Electron arranca el proxy Express **en memoria** (puerto local efímero) y sirve el frontend; no hace falta abrir el navegador ni lanzar dos procesos.

- **Tokens**: en la app se configuran desde **⚙ Ajustes** y se guardan cifrados en el equipo (Keychain en macOS, DPAPI en Windows, libsecret en Linux mediante `safeStorage`). Si el cifrado nativo no está disponible, la persistencia se desactiva y la UI lo indica. No se usa `.env` ni van embebidos en el binario.
- **Reportes**: se guardan en `~/Documents/BuscaGit/reportes` (carpeta escribible).
- **Health**: `GET /api/health` solo devuelve `{ ok: true }` para comprobar que el proxy vive.

### Construir

[](https://github.com/vperezguzman66/buscagit#construir)

```shell
npm install
npm run dist        # tu SO actual -> carpeta release/
```

> **Compilación cruzada:** cada instalador se construye mejor en su propio SO. Desde macOS puedes generar el `.dmg` sin problema; para el `.exe` de Windows necesitas Wine y para los paquetes de Linux normalmente Docker. Lo más cómodo para los tres a la vez es CI (ver abajo). Los artefactos quedan en `release/`.

### Compilar los tres con CI (GitHub Actions)

[](https://github.com/vperezguzman66/buscagit#compilar-los-tres-con-ci-github-actions)

El repo incluye [`.github/workflows/build.yml`](https://github.com/vperezguzman66/buscagit/blob/main/.github/workflows/build.yml), que compila en paralelo en `macos`, `windows` y `ubuntu` (cada uno su instalador nativo, sin emulación):

- **En cada push a `main`, PR o ejecución manual** (_Actions → build → Run workflow_): compila y sube los instaladores como _artifacts_ del run.
- **Al hacer push de un tag `vX.Y.Z`**: además publica un _GitHub Release_ con los `.dmg`/`.zip`, `.exe` y `.AppImage`/`.deb` adjuntos.

```shell
git tag v0.1.0 && git push origin v0.1.0   # dispara build + release
```

> Requiere subir el proyecto a GitHub. No hacen falta secretos: las apps van sin firmar y el Release usa el `GITHUB_TOKEN` que provee Actions.

### Primera apertura (apps sin firmar)

[](https://github.com/vperezguzman66/buscagit#primera-apertura-apps-sin-firmar)

La app se genera **sin firmar** (firmar de verdad requiere certificados de pago de Apple/Microsoft). Por eso el sistema avisa la primera vez. Cómo abrirla:

- **macOS** — al venir de una descarga, macOS la pone en cuarentena. Tras moverla a _Aplicaciones_, quita la cuarentena una vez:
    
    ```shell
    xattr -dr com.apple.quarantine /Applications/BuscaGit.app
    ```
    
    (Alternativa por UI en Intel: clic derecho sobre la app → _Abrir_.) El binario es x64, así que en Apple Silicon corre bajo Rosetta sin problema.
    
- **Windows** — SmartScreen mostrará un aviso: _Más información_ → _Ejecutar de todas formas_.
    
- **Linux** — al `AppImage` dale permiso de ejecución: `chmod +x BuscaGit-*.AppImage`.
    

> Nota técnica: no se puede aplicar una firma _ad-hoc_ a mano porque Electron valida la integridad del `asar` cuando la app está firmada, y ese hash solo lo inyecta el pipeline de firma de electron-builder (con certificado). Una firma manual rompe el arranque. La firma real se habilita con `CSC_LINK`/`CSC_KEY_PASSWORD` (mac) y certificado de Windows cuando se disponga de ellos.

## Endpoints del proxy

[](https://github.com/vperezguzman66/buscagit#endpoints-del-proxy)

- `GET /api/search?q=<texto>&language=<lenguaje>&page=<n>` — busca código en GitHub.
- `GET /api/gitlab/search?q=<texto>&scopeType=group|project&scopeId=<ruta>&page=<n>` — busca código (blobs) en GitLab, acotado a un grupo o proyecto.
- `GET /api/rate_limit` — estado del rate limit de GitHub.
- `GET /api/settings` — devuelve `{ tokens: { github, gitlab }, secured }`.
- `POST /api/settings` — guarda o borra tokens (`{ githubToken?, gitlabToken? }`). Requiere JSON válido, solo acepta strings de hasta 4096 caracteres y solo está habilitado cuando hay cifrado disponible para persistirlos.
- `POST /api/save` — guarda reportes `.txt` en `reportes/`. Requiere `{ filename, content }` y el nombre debe ser seguro.
- `GET /api/health` — devuelve `{ ok: true }`.

## Estructura del proyecto

[](https://github.com/vperezguzman66/buscagit#estructura-del-proyecto)

```
buscagit/
├─ server/
│  ├─ app.js          Factory del proxy Express (ensambla middleware + rutas).
│  ├─ index.js        Entrada de DEV: createApp() + listen(8787). Lee .env.
│  ├─ lib/
│  │  ├─ origins.js   Orígenes permitidos y middleware de bloqueo.
│  │  ├─ token-store.js  Carga/guarda tokens y expone helpers de acceso.
│  │  └─ validation.js   Validaciones de JSON, nombres seguros y tokens.
│  ├─ routes/
│  │  ├─ github.js    Rutas de búsqueda/rate limit de GitHub.
│  │  ├─ gitlab.js    Rutas de búsqueda de GitLab.
│  │  ├─ health.js    GET /api/health.
│  │  └─ settings.js  GET/POST /api/settings y POST /api/save.
│  └─ services/
│     ├─ github.js    URLs y headers de GitHub.
│     └─ gitlab.js    URLs/headers, resolver de proyectos y concurrencia.
├─ electron/
│  └─ main.js         Proceso principal de Electron: levanta el Express en memoria,
│                     cifra tokens (safeStorage), abre la ventana, logging a archivo.
├─ src/                          Frontend React
│  ├─ App.jsx                    Contenedor principal: estado, orquestación y composición.
│  ├─ main.jsx                   Punto de montaje de React.
│  ├─ styles/                    Estilos modulares del tema oscuro.
│  │  ├─ base.css               Reset, variables, body, spinner y root.
│  │  ├─ layout.css             Header, rate limit y empty state.
│  │  ├─ forms.css              Formulario, chips y botones.
│  │  ├─ progress.css           Barra de progreso y tags de estado.
│  │  ├─ results.css            Tarjetas de resultados y lista de URLs.
│  │  ├─ banners.css            Banners de error, info y aviso.
│  │  └─ scrollbar.css          Scrollbar nativa estilizada.
│  ├─ components/
│  │  ├─ SearchForm.jsx          Formulario de búsqueda y selección de fuente/lenguajes.
│  │  ├─ ProgressPanel.jsx       Barra, pasos y botón de cancelar.
│  │  ├─ ResultsPanel.jsx        Encabezado de resultados + tarjetas por lenguaje.
│  │  ├─ EmptyState.jsx          Estado vacío inicial.
│  │  ├─ LanguageCard.jsx        Tarjeta plegable por lenguaje + guardar.
│  │  └─ SettingsModal.jsx       Modal de Ajustes (tokens).
│  ├─ constants/
│  │  └─ search.js               Lenguajes y colores compartidos por la UI.
│  ├─ api/
│  │  ├─ client.js               Cliente HTTP centralizado para el proxy local.
│  │  ├─ normalizers.js          Shapes canónicos para settings, búsquedas y guardado.
│  │  ├─ reports.js             Guardado de reportes vía `/api/save`.
│  │  ├─ search.js               Búsqueda y rate limit de GitHub/GitLab.
│  │  └─ settings.js            Lectura y guardado de configuración.
│  ├─ services/
│  │  ├─ reports/
│  │  │  └─ reportDownloads.js   Armado de nombres, deduplicado y guardado de reportes.
│  │  └─ search/
│  │     ├─ githubSearch.js      Estrategia GitHub: páginas, rate limit y abortos.
│  │     └─ gitlabSearch.js      Estrategia GitLab: blobs, concurrencia y agrupado.
│  ├─ hooks/
│  │  ├─ useSearchSession.js     Orquesta la búsqueda (rama GitHub y rama GitLab),
│  │  │                          estado, progreso y rate limit.
│  │  ├─ useSearchViewModel.js   Estado de la pantalla de búsqueda, acciones y
│  │  │                          métricas derivadas.
│  │  └─ useSettingsStatus.js    Carga y normaliza el estado de tokens/ajustes.
│  └─ utils/
│     └─ errors.js               Normaliza mensajes de error para la UI.
├─ build/                        Recursos de empaquetado (icon.svg fuente, icon.png).
├─ .github/workflows/build.yml   CI: compila los 3 instaladores y publica releases.
├─ tests/server.test.js          Tests base del proxy Express.
├─ eslint.config.js              Configuración de ESLint en modo flat.
├─ .prettierrc.json               Reglas de formato de Prettier.
├─ vite.config.js                Dev: proxy de /api → 8787.
└─ package.json                  Scripts + config de electron-builder.
```

**Flujo de una búsqueda:** `App.jsx` (submit) → `useSearchViewModel.handleSearch()` → `useSearchSession.search()` → `services/search/githubSearch.js` o `services/search/gitlabSearch.js` → `src/api/search.js` → `server/app.js` añade el token y llama a GitHub/GitLab → los resultados vuelven agrupados `{ lenguaje: { urls, totalCount } }` y se pintan como `LanguageCard`.

## Buscar en GitLab

[](https://github.com/vperezguzman66/buscagit#buscar-en-gitlab)

GitLab **no permite buscar código globalmente** como GitHub: la búsqueda de blobs solo funciona **acotada a un grupo o proyecto** (requiere Advanced/Exact code search, activado en gitlab.com). Por eso, al elegir la fuente _GitLab_ en la UI hay que indicar la ruta del grupo (`gitlab-org`) o del proyecto (`gitlab-org/gitlab`). Como GitLab no filtra por lenguaje, los resultados se **agrupan por extensión de archivo**.

1. Crea un Personal Access Token en [https://gitlab.com/-/user_settings/personal_access_tokens](https://gitlab.com/-/user_settings/personal_access_tokens) con scope **`read_api`**.
2. Añádelo en **⚙ Ajustes** (app) o a `.env` como `GITLAB_TOKEN=glpat-...` (o pégalo en la UI como override de sesión).

> **Token clásico vs fine-grained:** usa un token **clásico** con scope `read_api`. Los tokens _fine-grained_ (granulares) usan otro modelo de permisos donde `read_api` no existe y la cobertura del endpoint de búsqueda no está garantizada; si recibes `401/403`, cambia a un token clásico.

## Notas sobre la API de GitHub

[](https://github.com/vperezguzman66/buscagit#notas-sobre-la-api-de-github)

- La búsqueda de código exige token (autenticación obligatoria).
- Tope de **1000 resultados** por consulta (10 páginas de 100), aunque `total_count` informe muchos más.
- Rate limit de búsqueda de código: ~30 req/min autenticado.

## Solución de problemas

[](https://github.com/vperezguzman66/buscagit#soluci%C3%B3n-de-problemas)

- **`Origen no permitido` al abrir el frontend**: añade el origen correcto en `BUSCAGIT_ALLOWED_ORIGINS` o usa uno de los puertos locales documentados.
- **No aparecen tokens en Ajustes**: revisa si el proxy está arriba y si el `.env` o la configuración guardada se está leyendo correctamente.
- **`POST /api/save` devuelve 400**: el nombre debe terminar en `.txt` y no puede incluir rutas ni caracteres extraños.
- **`npm run lint` o `npm run format:check` fallan**: ejecuta `npm run format` y vuelve a comprobar. Para cambios grandes, revisa primero `src/hooks/useSearchSession.js` y `server/app.js`, que son los más sensibles.