
[[Marcos - Diego]]

# Instructivo paso a paso — Motor de Precios Zarzar

> SAP Business One v10 · SQL Server · on-premise

> Ejecutar en orden. Cada fase termina con un punto de verificación ✔.

> **Regla de oro: todo se prueba primero en la base de TEST.**

---

## FASE 0 — Preparación (antes de tocar nada)

1. Confirmar que existe una **base de test** actualizada (copia reciente de la productiva). Si no existe, pedir al partner/TI que restaure un respaldo con otro nombre (ej. `SBO_ZARZAR_TEST`).
2. Respaldo de la base productiva vigente (aunque no se toque hoy).
3. Usuario B1 con perfil **superusuario** disponible.
4. Verificar versión y patch de B1: Ayuda → Acerca de. Anotarla.
5. Verificar que el **DTW instalado sea de la misma versión/patch** que B1 (DTW viene en el paquete de instalación del servidor B1, carpeta `...\SAP Business One\DataTransferWorkbench`).
6. Coordinar ventana horaria: la creación de campos de usuario **exige que todos los usuarios estén desconectados**.

✔ **Verificación**: tienes base de test, superusuario, DTW versión correcta y ventana coordinada.

---

## FASE 1 — Crear los campos de usuario (UDF) en el maestro de artículos

Ruta: **Herramientas → Herramientas de personalización → Campos definidos por el usuario - Gestión**

1. Expandir **Datos maestros → Artículos**. Seleccionar y pulsar **Añadir**.
2. Campo 1:

- - Título: `SubFamilia`
    - Descripción: `SubFamilia`
    - Tipo: **Alfanumérico**, Estructura: Regular, Longitud: `20`
    - Añadir. (B1 pedirá desconectar usuarios y reiniciará el formulario — normal.)

3. Repetir Añadir para el Campo 2:

- - Título: `Importado`
    - Descripción: `Importado (S/N)`
    - Tipo: **Alfanumérico**, Longitud: `1`
    - Marcar **Definir valores válidos**: agregar `S` = Sí, `N` = No
    - Marcar **Definir valor por defecto** = `N`
    - Añadir.

> Los campos quedan como `U_SubFamilia` y `U_Importado` en la tabla OITM. En el maestro de artículos aparecen en la ventana lateral de campos de usuario (se muestra/oculta con **Ctrl+Shift+U**).

✔ **Verificación**: abrir un artículo cualquiera, Ctrl+Shift+U, y ver ambos campos. `Importado` debe mostrar el dropdown S/N con N por defecto.

---

## FASE 2 — Crear la tabla de subfamilias (@SUBFAM)

### 2.1 Crear la tabla

Ruta: **Herramientas → Herramientas de personalización → Tablas definidas por el usuario - Definición**

1. En una fila nueva: Nombre tabla: `SUBFAM` · Descripción: `Subfamilias de artículos` · Tipo de objeto: **Sin objeto**.
2. Actualizar → OK.

### 2.2 Agregar el campo Familia a la tabla

Ruta: **Campos definidos por el usuario - Gestión → Tablas definidas por el usuario → SUBFAM → Añadir**

- Título: `Familia`
- Descripción: `Código grupo de artículos`
- Tipo: **Alfanumérico**, Longitud: `20`

> La tabla ya trae `Code` y `Name` por defecto: Code = código de la subfamilia, Name = nombre visible.

✔ **Verificación**: **Herramientas → Ventanas definidas por el usuario → SUBFAM** abre una grilla con columnas Code, Name, Familia. (Si no aparece: cerrar sesión y volver a entrar.)

---

## FASE 3 — Crear la tabla de reglas de precio (@PARAMPREC)

### 3.1 Crear la tabla

Igual que 2.1: Nombre `PARAMPREC` · Descripción `Parámetros motor de precios` · Tipo **Sin objeto**.

### 3.2 Agregar campos (Campos definidos por el usuario - Gestión → Tablas → PARAMPREC)

|Título|Tipo|Detalle|
|:--|:--|:--|
|`Familia`|Alfanumérico 20|opcional (vacío = cualquiera)|
|`SubFam`|Alfanumérico 20|opcional|
|`Importado`|Alfanumérico 1|valores válidos S/N, **sin** valor por defecto|
|`Factor`|**Unidades y totales → Cantidad**|⚠️ NO usar tipo "Numérico": es solo enteros; "Cantidad" permite decimales (1.40)|
|`ListaDestino`|Numérico|nº de la lista de precios a actualizar|
|`Vigencia`|Fecha|opcional|
||

✔ **Verificación**: Ventanas definidas por el usuario → PARAMPREC abre la grilla con todas las columnas. Probar ingresar `1.4` en Factor y que acepte el decimal.

---

## FASE 4 — Búsqueda formateada: subfamilias filtradas por familia

### 4.1 Guardar la consulta

1. **Herramientas → Consultas → Generador de consultas** (o Administrador de consultas → lápiz).
2. Pegar y ejecutar:

```sql
SELECT Code, Name FROM [@SUBFAM] WHERE U_Familia = $[OITM.ItmsGrpCod]
```

(Al ejecutarla directa dará error por la variable `$[...]` — es normal; **Guardar** igual.)

3. Guardar como: `FMS - SubFamilias por Familia`, en una categoría (ej. "Sistema").

### 4.2 Asignar al campo

1. Abrir **Datos maestros de artículo** y mostrar campos de usuario (Ctrl+Shift+U).
2. Hacer clic DENTRO del campo **SubFamilia**.
3. Presionar **Alt+Mayús+F2** (asignar búsqueda formateada).
4. Elegir **Buscar en consultas guardadas existentes** → seleccionar `FMS - SubFamilias por Familia`.
5. Marcar **Actualizar de forma regular** → campo: **Grupo de artículos**.
6. Marcar **Visualizar valores grabados**.
7. Actualizar.

> Si al usarla no filtra: reemplazar en la consulta `$[OITM.ItmsGrpCod]` por la variable de pantalla `$[$39.0.0]` (ítem 39 = combo Grupo de artículos en el formulario de artículo), guardar y probar de nuevo.

✔ **Verificación**: en un artículo, elegir una Familia, pararse en SubFamilia y presionar **Tab**: debe abrir la lista SOLO con las subfamilias de esa familia (fase 5 debe estar cargada para ver datos; puede hacerse esta verificación después de la fase 5).

---

## FASE 5 — Cargar los catálogos

### 5.1 Subfamilias

**Herramientas → Ventanas definidas por el usuario → SUBFAM**: cargar filas.

|Code|Name|Familia|
|:--|:--|:--|
|TORN|Tornillería|101|
|HMAN|Herramientas manuales|101|
||

> `Familia` = **código numérico** del grupo de artículos (`OITB.ItmsGrpCod`). Para verlos: Gestión → Definiciones → Inventario → Grupos de artículos, o `SELECT ItmsGrpCod, ItmsGrpNam FROM OITB`.

### 5.2 Reglas de precio

**Ventanas definidas por el usuario → PARAMPREC**. Ejemplo:

|Code|Name|Familia|SubFam|Importado|Factor|ListaDestino|
|:--|:--|:--|:--|:--|:--|:--|
|R001|Default general||||1.30|2|
|R002|Ferretería|101|||1.35|2|
|R003|Ferret. tornillería import.|101|TORN|S|1.55|2|
||

> Dejar **siempre una regla default** (sin Familia/SubFam/Importado) para que ningún artículo quede sin precio.

✔ **Verificación**: `SELECT * FROM [@SUBFAM]` y `SELECT * FROM [@PARAMPREC]` en SSMS devuelven lo cargado.

---

## FASE 6 — Clasificación masiva inicial de artículos (DTW #1)

_Solo si el catálogo ya existe y hay que asignarle SubFamilia/Importado en masa. Si son pocos artículos, hacerlo a mano y saltar a Fase 7._

### 6.1 Preparar archivo

Exportar artículos: `SELECT ItemCode, ItemName, ItmsGrpCod FROM OITM ORDER BY ItmsGrpCod` → clasificar en Excel → construir `Items_clasif.csv`:

|RecordKey|ItemCode|U_SubFamilia|U_Importado|
|:--|:--|:--|:--|
|1|A0001|TORN|S|
|2|A0002|HMAN|N|
||

- Guardar como **CSV** (separado por comas; si Excel genera ";", DTW permite elegir el separador al mapear)
- Sin filas vacías al final, sin totales, cabeceras exactas

### 6.2 Ejecutar DTW

1. Abrir Data Transfer Workbench → conectar: Server = servidor SQL, Tipo BD = MSSQL (versión correspondiente), Base = **la de TEST primero**, usuario `manager` (o superusuario).
2. Asistente: **Update** (NO Add).
3. Objeto de negocio: **Datos maestros → oItems**.
4. Asociar `Items_clasif.csv` a la plantilla **Items** (los campos U_ aparecen al final de la plantilla; mapear por nombre).
5. **Marcar "Simulation Run"** → Ejecutar → revisar log: 0 errores.
6. Desmarcar simulación → ejecutar real → revisar log.

✔ **Verificación**: consulta de control (nota SAP en Zarzar, sección 5) con `WHERE T2.Name IS NULL` → idealmente 0 filas sin clasificar.

---

## FASE 7 — Simulación de precios y aprobación

1. En SSMS, sobre la base de test, ejecutar la **consulta de simulación** (nota SAP en Zarzar, sección 8.3).
2. Clic derecho sobre los resultados → **Guardar resultados como** → Excel/CSV: `simulacion_precios_AAAAMMDD.xlsx`.
3. Revisar en Excel:

- - Ningún PrecioFinal en 0 o negativo
    - Columna ReglaAplicada coherente (spot-check de 10 artículos de distintas familias)
    - Filtrar diferencias grandes vs precio actual y validarlas con el responsable comercial

4. **Obtener aprobación formal** (correo o firma) antes de continuar.

✔ **Verificación**: Excel aprobado y archivado.

---

## FASE 8 — Respaldo de precios actuales (rollback)

Antes de pisar precios, exportar los vigentes de la lista destino:

```sql
SELECT I.ItemCode, I.PriceList, I.Price, I.Currency
FROM ITM1 I
WHERE I.PriceList = 2   -- lista destino
```

Guardar como `respaldo_precios_lista2_AAAAMMDD.csv`. **Este archivo ES el rollback**: si algo sale mal, se recarga con el mismo procedimiento DTW de la fase 9.

✔ **Verificación**: archivo de respaldo guardado y con datos.

---

## FASE 9 — Carga de precios (DTW #2)

### 9.1 Generar los 2 archivos

Ejecutar en SSMS la **consulta generadora** (nota SAP en Zarzar, sección 8.5) y armar:

`Items.csv`

|RecordKey|ItemCode|
|:--|:--|
|1|A0001|
||

`Items_Prices.csv`

|RecordKey|PriceList|Price|Currency|
|:--|:--|:--|:--|
|1|2|15990|CLP|
||

- RecordKey correlativo idéntico entre ambos archivos (misma fila = mismo artículo)
- Sin separador de miles; decimales según configuración regional de la máquina DTW

### 9.2 Ejecutar DTW

1. Conectar (TEST primero) → **Update** → objeto **oItems**.
2. Asociar `Items.csv` a la plantilla **Items** y `Items_Prices.csv` a la plantilla **ItemPrices**.
3. Verificar mapeo: ItemCode, y en el hijo PriceList / Price / Currency.
4. **Simulation Run** → 0 errores.
5. Ejecución real → revisar log (artículos OK vs error).
6. **Guardar el proyecto DTW (.dtw)**: los próximos ciclos serán solo regenerar CSVs y ejecutar.

### 9.3 Verificar en B1

1. Inventario → Listas de precios → abrir la lista destino → spot-check de 10 artículos contra el Excel aprobado.
2. Consulta cruzada:

```sql
SELECT T0.ItemCode, I.Price AS PrecioCargado,
       ROUND(T0.LastPurPrc * P.U_Factor, 0) AS PrecioEsperado
FROM OITM T0
INNER JOIN ITM1 I ON I.ItemCode = T0.ItemCode AND I.PriceList = 2
CROSS APPLY (
    SELECT TOP 1 R.U_Factor FROM [@PARAMPREC] R
    WHERE (R.U_Familia IS NULL OR R.U_Familia = CAST(T0.ItmsGrpCod AS NVARCHAR))
      AND (R.U_SubFam IS NULL OR R.U_SubFam = T0.U_SubFamilia)
      AND (R.U_Importado IS NULL OR R.U_Importado = T0.U_Importado)
    ORDER BY
      CASE WHEN R.U_Familia IS NOT NULL THEN 1 ELSE 0 END +
      CASE WHEN R.U_SubFam IS NOT NULL THEN 2 ELSE 0 END +
      CASE WHEN R.U_Importado IS NOT NULL THEN 1 ELSE 0 END DESC
) P
WHERE I.Price <> ROUND(T0.LastPurPrc * P.U_Factor, 0)
```

→ **0 filas = carga perfecta.**

✔ **Verificación**: 0 diferencias. Recién entonces, repetir fases 7-9 sobre la base **PRODUCTIVA** (con su propio respaldo de fase 8).

---

## FASE 10 — Cierre y operación recurrente

- Archivar por ciclo: simulación aprobada + CSVs cargados + log DTW + respaldo de precios. Esa carpeta es el historial de cambios de precios.
- Restringir por autorizaciones el acceso a las ventanas SUBFAM y PARAMPREC (un solo dueño del catálogo).
- Ciclo recurrente = fases 7 → 8 → 9 (con el proyecto .dtw guardado, ~30 min).

## Errores comunes de DTW y su causa

| Error                          | Causa probable                                                        |
| :----------------------------- | :-------------------------------------------------------------------- |
| "Item not found"               | ItemCode con espacios o que no existe; revisar el CSV                 |
| "Invalid price list"           | Nº de lista destino no existe en la compañía                          |
| No conecta a la compañía       | Versión de DTW ≠ versión/patch de B1                                  |
| Carga 0 registros sin error    | Modo Add en vez de Update, o RecordKey desalineado entre padre e hijo |
| Precios cargados ×1000 o /1000 | Separador decimal del CSV ≠ configuración regional de la máquina DTW  |


