
[[Marcos - Diego]]


> Notas de trabajo sobre SAP Business One v10, base de datos SQL Server, on-premise.

> Fecha: 2026-07-08 · Consultor: Víctor Pérez

---

## 1. SAP B1 v10 con SQL Server: qué funcionalidades tengo

La versión SQL tiene **toda la funcionalidad operativa y transaccional** del ERP. Lo que se pierde frente a HANA es la capa analítica avanzada.

### Disponible en SQL Server

**Módulos operativos completos:**

- Finanzas y contabilidad: plan de cuentas, asientos, multimoneda, centros de costo, presupuestos
- Ventas: oportunidades, cotizaciones, órdenes, facturación, devoluciones
- Compras: solicitudes, órdenes, recepciones, facturas de proveedor
- Socios de negocio con CRM básico
- Inventario: multialmacén, series/lotes, listas de precios, conteos
- Producción y MRP: listas de materiales, órdenes de fabricación
- Servicio: contratos, llamadas, tarjetas de equipo
- Gestión de proyectos, Bancos y conciliación, RRHH básico

**Herramientas técnicas:**

- Consultas SQL directas y Query Generator
- Alertas y procedimientos de aprobación
- UDF / UDT / UDO (campos, tablas y objetos de usuario)
- Transaction Notifications (`SP_TransactionNotification`)
- Crystal Reports
- SDK completo: DI API y UI API (.NET)
- Integration Framework (B1if)
- DTW para cargas masivas
- **Web Client** (Fiori en navegador) y **Service Layer** (API REST/OData): disponibles para SQL en feature packs recientes de v10 → _verificar nivel de patch instalado en Zarzar_

### Exclusivo de HANA (NO disponible)

- Pervasive Analytics: dashboards y KPIs nativos
- Pronósticos inteligentes
- Gestión avanzada de programación de entregas (ATP avanzado)
- Pronóstico de flujo de caja en tiempo real con gráficos
- Enterprise Search (búsqueda tipo Google)
- Análisis predictivo, búsqueda semántica, recomendaciones automáticas

> 💡 **Oportunidad de consultoría**: las brechas analíticas de SQL se cubren con Power BI conectado a la base, reportería Python, o integraciones vía Service Layer/DI API.

---

## 2. Listas de precios (funcionalidad completa en SQL)

### Gestión de listas

- Listas ilimitadas (mayorista, minorista, distribuidor…)
- **Listas derivadas por factor**: `Lista Distribuidor = Lista Base × 0,85` → se recalculan al actualizar la base
- Método de redondeo propio por lista
- Precios en moneda principal + 2 monedas adicionales por artículo
- Precios por unidad de medida (caja ≠ unidad, con grupos de UoM)
- Listas de precios brutos (impuesto incluido)
- Períodos de validez

### Precios especiales y descuentos

- **Precios especiales por socio de negocio**: precio pactado por cliente/artículo, con validez y descuentos escalonados por cantidad
- **Grupos de descuento**: por grupo de artículos, propiedades o fabricante; asignables a cliente o grupo de clientes
- Descuentos por período y volumen
- **Jerarquía de determinación**: 1º precio especial del cliente → 2º grupos de descuento → 3º lista asignada al socio

### Operación

- Actualización masiva por factor o porcentaje
- Cargas masivas vía DTW
- Autorizaciones por lista (vendedor no ve costos)
- UDFs sobre tablas de precios

### Tablas SQL del modelo de precios

|Tabla|Contenido|
|:--|:--|
|`OPLN`|Cabecera de listas de precios|
|`ITM1`|Precios por artículo/lista|
|`OSPP` / `SPP1` / `SPP2`|Precios especiales por cliente y escalas por cantidad|
|`OEDG` / `EDG1`|Grupos de descuento|
||

Escritura programática: objetos **PriceLists** y **SpecialPrices** del DI API, o entidades equivalentes vía Service Layer.

---

## 3. Último precio de compra (Last Purchase Price)

Lista de precios especial que **el sistema mantiene automáticamente**: registra el precio de la compra más reciente de cada artículo.

**Se actualiza con:**

- Factura de proveedores (caso más común)
- Entrada de mercancías con precio
- Cantidades iniciales de inventario con precio
- Costos de importación (landed costs)

**Características:**

- Registrado en moneda local (al tipo de cambio del documento), por UM base, considerando descuento de línea
- **NO es el costo del artículo** (el costo de valorización — promedio/FIFO/estándar — se calcula aparte)
- Es único por artículo a nivel compañía (no por bodega ni proveedor)

**Usos típicos:**

- Base para listas derivadas: `Precio Venta = Último precio compra × 1,4`
- Precio base para ganancia bruta en ventas (Gestión → Inicialización del sistema → Parámetros de documento)

```sql
SELECT ItemCode, ItemName, LastPurPrc AS UltimoPrecio,
       LastPurCur AS Moneda, LastPurDat AS FechaUltCompra
FROM OITM
WHERE LastPurPrc > 0
ORDER BY LastPurDat DESC
```

> 🔍 **Auditoría útil**: comparar `LastPurPrc` vs `AvgPrice` (costo promedio) para detectar alzas de proveedor no trasladadas al precio de venta.

---

## 4. Último precio determinado (Last Evaluated Price)

Precio calculado para cada artículo **la última vez que se ejecutó el Informe de simulación de valorización de inventario** (Inventario → Informes de inventario → Valorización de stock/simulación).

**Diferencia clave con el último precio de compra:**

- Último precio de compra → se actualiza automáticamente con cada transacción
- Último precio determinado → se actualiza **solo cuando alguien corre el informe** (dato estático, puede tener meses de antigüedad)

⚠️ **Advertencia**: NO usar como base de listas de venta ni decisiones de valorización. Depende del método elegido en la simulación, puede estar obsoleto o en cero. Su uso correcto es comparar escenarios de valorización (FIFO vs promedio) sin afectar contabilidad.

> 🚩 Bandera roja en clientes: listas de venta derivadas del "último precio determinado" = precios calculados sobre costos congelados.

---

## 5. Clasificación de artículos: Familia / SubFamilia / Origen

Los grupos de artículos (`OITB`) son **planos** — no hay jerarquía nativa. Diseño acordado para 3 clasificaciones:

### Arquitectura

1. **Familia** → Grupo de artículos estándar (`OITB`). Participa en informes, descuentos, determinación contable, MRP.
2. **SubFamilia** → UDT `@SUBFAM` + UDF `U_SubFamilia` en OITM, con Formatted Search dependiente.
3. **Origen** → UDF `U_Origen` con 4 valores válidos:

|Código|Valor|
|:--|:--|
|IMP|Importado|
|NIM|No Importado|
|FAB|Fabricación Propia|
|NAC|Compra Nacional|
||

> Nota: para artículos FAB conviene mantener coherencia con el campo nativo _Método de aprovisionamiento = Fabricar_ del maestro de artículos (pestaña Planificación). Definir con Zarzar la semántica exacta de NIM vs NAC para evitar clasificaciones ambiguas.

### UDT @SUBFAM

Columnas: `Code`, `Name`, `U_Familia` (referencia al código del grupo de artículos).

### Formatted Search del campo U_SubFamilia (Tab+F2 sobre el campo)

```sql
SELECT Code, Name FROM [@SUBFAM]
WHERE U_Familia = $[OITM.ItmsGrpCod]
```

- Configurar **Auto Refresh al cambiar el campo Grupo de artículos** (evita subfamilias huérfanas si cambian la familia después)

### Consulta de control / auditoría

```sql
SELECT T0.ItemCode, T0.ItemName,
       T1.ItmsGrpNam AS Familia,
       T2.Name AS SubFamilia,
       T0.U_Origen AS Origen
FROM OITM T0
INNER JOIN OITB T1 ON T0.ItmsGrpCod = T1.ItmsGrpCod
LEFT JOIN [@SUBFAM] T2 ON T0.U_SubFamilia = T2.Code
ORDER BY T1.ItmsGrpNam, T2.Name
```

→ Con `WHERE T2.Name IS NULL` detecta artículos sin subfamilia.

### Candado duro (opcional): SP_TransactionNotification

Rechaza crear/actualizar artículos sin subfamilia válida para su familia:

```sql
IF @object_type = '4' AND @transaction_type IN ('A','U')
BEGIN
  IF EXISTS (SELECT 1 FROM OITM T0
             LEFT JOIN [@SUBFAM] T1 ON T0.U_SubFamilia = T1.Code
                AND T1.U_Familia = CAST(T0.ItmsGrpCod AS NVARCHAR)
             WHERE T0.ItemCode = @list_of_cols_val_tab_del
               AND T1.Code IS NULL)
  BEGIN
    SET @error = 1
    SET @error_message = 'Debe asignar una SubFamilia válida para la Familia seleccionada'
  END
END
```

### Cargas masivas

- Clasificación inicial del catálogo existente: exportar → clasificar en Excel → cargar UDFs vía **DTW** (plantilla OITM incluye las columnas U_ una vez creados los campos)
- Automatización: DI API o Service Layer

---

## 6. Rutas de menú correctas (v10 español)

|Qué|Ruta|
|:--|:--|
|Crear tabla de usuario (UDT)|**Herramientas → Herramientas de personalización → Tablas definidas por el usuario - Definición**|
|Crear campos de usuario (UDF)|Herramientas → Herramientas de personalización → Campos definidos por el usuario - Gestión|
|**Mantener datos de la UDT** (agregar subfamilias)|**Herramientas → Ventanas definidas por el usuario → SUBFAM**|
|Propiedades de artículo|Gestión → Definiciones → Inventario → Propiedades de artículo|
|Grupos de artículos|Gestión → Definiciones → Inventario → Grupos de artículos|
|Autorizaciones|Gestión → Inicialización del sistema → Autorizaciones → Autorizaciones generales|
||

**Requisitos y trampas:**

- Crear UDT/UDF requiere **superusuario** (o autorización total en Herramientas de personalización)
- Crear UDFs exige que **todos los usuarios estén desconectados** → hacerlo fuera de horario
- La UDT debe ser tipo **"Sin objeto"** para aparecer en Ventanas definidas por el usuario
- Si la tabla recién creada no aparece en el menú → cerrar sesión y volver a entrar
- El "@" de `@SUBFAM` lo antepone el sistema; en SQL se referencia `[@SUBFAM]`
- El título amigable de los campos U_ se define en la descripción del campo al crearlo

---

## 7. Gobernanza y mantención del esquema

- **Dueño único del catálogo de subfamilias**: restringir por autorizaciones quién accede a la ventana SUBFAM (evita duplicados tipo "Tornillos/Tornilleria/TORNILLOS")
- **Flujo al crear artículo**: elegir Familia → Tab en SubFamilia → dropdown filtrado
- Los UDF no son obligatorios por sí solos → estrategia escalonada:

1. 1. Partir con Formatted Search + consulta de auditoría semanal
    2. Si siguen apareciendo artículos sin clasificar → instalar el candado en SP_TransactionNotification

---

## Pendientes / próximos pasos

- [ ] Verificar feature pack instalado (¿Web Client y Service Layer disponibles?)
- [ ] Confirmar si el catálogo existente requiere reclasificación masiva vía DTW
- [ ] Definir dueño del catálogo de subfamilias
- [ ] Evaluar auditoría LastPurPrc vs AvgPrice para revisión de márgenes

---

## 8. Motor de precios por parámetros (propuesta completa)

**Objetivo**: calcular el precio final de venta de cada artículo según su clasificación (Familia / SubFamilia / Origen) aplicando factores definidos en una tabla de parámetros, y volcarlo a una lista de precios destino de forma controlada.

### 8.1 Tabla de parámetros: UDT `@PARAMPREC`

|Campo|Tipo|Contenido|
|:--|:--|:--|
|`Code` / `Name`|Alfanumérico|Identificador de la regla|
|`U_Familia`|Alfanumérico|Código grupo de artículos (opcional)|
|`U_SubFam`|Alfanumérico|Código subfamilia (opcional)|
|`U_Origen`|Alfanumérico|IMP / NIM / FAB / NAC (opcional)|
|`U_Factor`|Numérico|Multiplicador sobre precio base (ej. 1.40)|
|`U_ListaDestino`|Numérico|Nº de lista de precios a actualizar|
|`U_Vigencia`|Fecha|Vigencia desde (opcional)|
||

Los campos opcionales (NULL) permiten reglas de distinta especificidad.

### 8.2 Lógica de resolución: la regla más específica gana

Prioridad (mayor a menor):

1. Familia + SubFamilia + Origen
2. Familia + SubFamilia
3. Familia + Origen
4. Familia
5. Regla por defecto (sin filtros)

La subfamilia pondera más que los demás criterios.

### 8.3 Consulta de simulación (validar ANTES de aplicar)

```sql
SELECT T0.ItemCode, T0.ItemName,
       T1.ItmsGrpNam AS Familia, T0.U_SubFamilia, T0.U_Origen,
       T0.LastPurPrc AS PrecioBase,
       P.U_Factor,
       ROUND(T0.LastPurPrc * P.U_Factor, 0) AS PrecioFinal,
       P.Code AS ReglaAplicada
FROM OITM T0
INNER JOIN OITB T1 ON T0.ItmsGrpCod = T1.ItmsGrpCod
CROSS APPLY (
    SELECT TOP 1 R.Code, R.U_Factor
    FROM [@PARAMPREC] R
    WHERE (R.U_Familia IS NULL OR R.U_Familia = CAST(T0.ItmsGrpCod AS NVARCHAR))
      AND (R.U_SubFam IS NULL OR R.U_SubFam = T0.U_SubFamilia)
      AND (R.U_Origen IS NULL OR R.U_Origen = T0.U_Origen)
    ORDER BY
      CASE WHEN R.U_Familia IS NOT NULL THEN 1 ELSE 0 END +
      CASE WHEN R.U_SubFam IS NOT NULL THEN 2 ELSE 0 END +
      CASE WHEN R.U_Origen IS NOT NULL THEN 1 ELSE 0 END DESC
) P
```

### 8.4 Aplicación de precios

⚠️ **NUNCA hacer** **`UPDATE ITM1`** **directo por SQL** — no soportado por SAP, rompe integridad y validaciones, puede invalidar soporte del partner.

**Vía elegida: DTW** (Data Transfer Workbench, modo actualización). Alternativas futuras si se quiere automatizar 100%: Service Layer o DI API.

### 8.5 Procedimiento DTW paso a paso

**Archivos a preparar (2 CSV/planillas, plantillas del objeto oItems):**

1. `Items.csv` (padre):

|RecordKey|ItemCode|
|:--|:--|
|1|A0001|
|2|A0002|
||

2. `Items_Prices.csv` (hijo, plantilla ItemPrices):

|RecordKey|PriceList|Price|Currency|
|:--|:--|:--|:--|
|1|2|15990|CLP|
|2|2|8990|CLP|
||

- `RecordKey` enlaza padre e hijo (correlativo simple)
- `PriceList` = número de la lista destino (visible en Inventario → Listas de precios)
- No usar separador de miles; decimal según configuración regional de la máquina DTW

**Consulta SQL que genera el contenido directo (exportar desde SSMS a CSV):**

```sql
-- Hijo Items_Prices: RecordKey, PriceList, Price, Currency
SELECT ROW_NUMBER() OVER (ORDER BY T0.ItemCode) AS RecordKey,
       P.U_ListaDestino AS PriceList,
       ROUND(T0.LastPurPrc * P.U_Factor, 0) AS Price,
       'CLP' AS Currency,
       T0.ItemCode                       -- misma fila sirve para armar Items.csv
FROM OITM T0
CROSS APPLY (
    SELECT TOP 1 R.U_Factor, R.U_ListaDestino
    FROM [@PARAMPREC] R
    WHERE (R.U_Familia IS NULL OR R.U_Familia = CAST(T0.ItmsGrpCod AS NVARCHAR))
      AND (R.U_SubFam IS NULL OR R.U_SubFam = T0.U_SubFamilia)
      AND (R.U_Origen IS NULL OR R.U_Origen = T0.U_Origen)
    ORDER BY
      CASE WHEN R.U_Familia IS NOT NULL THEN 1 ELSE 0 END +
      CASE WHEN R.U_SubFam IS NOT NULL THEN 2 ELSE 0 END +
      CASE WHEN R.U_Origen IS NOT NULL THEN 1 ELSE 0 END DESC
) P
WHERE T0.LastPurPrc > 0 AND T0.frozenFor = 'N'
```

**Ejecución en DTW:**

1. Conectarse a la base de la compañía (misma versión DTW que el cliente B1)
2. Tipo de operación: **Update** (no Add — actualiza artículos existentes)
3. Objeto de negocio: **oItems** → mapear Items.csv y Items_Prices.csv
4. Mapear campos (DTW los toma solos si las cabeceras coinciden con la plantilla)
5. **Simulation Run primero** (checkbox de simulación de DTW): valida sin grabar
6. Revisar log de errores → corregir → ejecutar real
7. Guardar el mapeo como proyecto DTW (.dtw) para reutilizarlo cada ciclo

**Precauciones:**

- ⚠️ Probar SIEMPRE primero en la base de test
- ⚠️ La actualización vía oItems solo modifica lo mapeado; no tocar otras columnas de la plantilla
- Partir con un lote de 5-10 artículos de prueba antes del catálogo completo
- Guardar los CSV de cada ciclo como respaldo/log de cambios (fecha en el nombre)

### 8.6 Flujo operativo (versión DTW)

1. Usuario mantiene reglas en la ventana de `@PARAMPREC`
2. Ejecutar consulta de simulación (sección 8.3) → exportar a Excel → **aprobación comercial**
3. Ejecutar consulta generadora (8.5) → exportar los 2 CSV
4. DTW en modo Update con Simulation Run → luego ejecución real
5. Archivar CSVs y log DTW del ciclo
6. Frecuencia sugerida: semanal o al cierre de cada ciclo de importaciones

### 8.7 Decisiones pendientes con Zarzar

- [ ] Precio base: ¿`LastPurPrc` (último precio compra, reactivo) o `AvgPrice` (costo promedio, amortiguado)?
- [ ] Lista(s) de precios destino
- [ ] Redondeo comercial (ej. a $10, a $90, a $990)
- [ ] Frecuencia de ejecución y responsable de aprobación
- [ ] ¿Artículos excluidos del motor (precios manuales)?

---

## 9. Tipo de Producto Mohican (factor en línea, SIN actualizar lista)

> Mohican = marca de la fábrica. Esta clasificación aplica un factor de cálculo **al momento de usar el precio** (cotización, factura, reporte), **nunca se escribe en la lista de precios**.

### 9.1 Arquitectura de dos etapas del modelo de precios

|Etapa|Clasificaciones|Factor|Persistencia|
|:--|:--|:--|:--|
|Motor de lista (DTW)|Familia / SubFamilia / Origen|@PARAMPREC|SÍ → actualiza lista destino|
|Factor en línea|Tipo Producto Mohican|@TIPOMOH|NO → se calcula al vuelo|
||

**Precio final = Precio de lista (etapa 1) × Factor Tipo Mohican (etapa 2)**

⚠️ Mantener las dimensiones separadas: el Tipo Mohican NO debe agregarse a @PARAMPREC, o el factor se aplicaría dos veces.

### 9.2 Estructuras

**UDT** **`@TIPOMOH`** (tipos y sus factores en un solo lugar):

|Code|Name|U_Factor|
|:--|:--|:--|
|REE|Reenvasado|(definir)|
|FAB|Fabricado|(definir)|
|NMO|No es Mohican|1.00|
|PRO|Promoción|(definir)|
||

**UDF** **`U_TipoMoh`** en OITM (alfanumérico 3), con búsqueda formateada que lee los valores desde @TIPOMOH:

```sql
SELECT Code, Name FROM [@TIPOMOH]
```

> Ventaja de UDT + FMS sobre "valores válidos": si mañana cambia un factor o se agrega un tipo, se edita la tabla y nada más.

### 9.3 Aplicación del factor en documentos de venta (FMS sobre el precio)

Búsqueda formateada asignada a la columna **Precio unitario** de las filas del documento de venta (cotización/orden/factura), con auto-refresh al cambiar **Número de artículo**:

```sql
SELECT I.Price * ISNULL(M.U_Factor, 1)
FROM ITM1 I
INNER JOIN OITM T0 ON T0.ItemCode = I.ItemCode
INNER JOIN OCRD C ON C.ListNum = I.PriceList
LEFT JOIN [@TIPOMOH] M ON M.Code = T0.U_TipoMoh
WHERE I.ItemCode = $[$38.1.0]
  AND C.CardCode = $[$4.0.0]
```

- `$[$38.1.0]` = columna ItemCode de la matriz de líneas; `$[$4.0.0]` = CardCode del documento
- `ISNULL(..., 1)`: artículo sin tipo asignado → factor neutro 1
- La FMS se asigna **por formulario**: repetir en cotización, orden de venta y factura según necesidad
- El usuario puede sobreescribir manualmente el precio después (evaluar si eso se permite o se bloquea por autorizaciones)

### 9.4 Alternativa/complemento: reporte de precios finales

Si el cliente prefiere consultar antes que automatizar el documento:

```sql
SELECT T0.ItemCode, T0.ItemName,
       T0.U_TipoMoh, M.Name AS TipoMohican,
       I.Price AS PrecioLista,
       ISNULL(M.U_Factor,1) AS FactorMoh,
       ROUND(I.Price * ISNULL(M.U_Factor,1), 0) AS PrecioFinal
FROM OITM T0
INNER JOIN ITM1 I ON I.ItemCode = T0.ItemCode AND I.PriceList = 2  -- lista destino
LEFT JOIN [@TIPOMOH] M ON M.Code = T0.U_TipoMoh
ORDER BY T0.ItemCode
```

Usable como consulta de usuario en B1, Crystal Report o base del futuro cotizador.

### 9.5 Pendientes con Zarzar

- [ ] Definir los factores de REE / FAB / PRO (¿NMO = 1.00?)
- [ ] ¿"Promoción" es un estado temporal? → si rota mucho, evaluar campo de vigencia en @TIPOMOH o manejarlo como precios especiales de B1
- [ ] ¿En qué documentos se aplica la FMS: solo cotización, o también orden y factura?
- [ ] ¿Se permite sobreescribir el precio calculado manualmente?
- [ ] Distinción FAB (Origen: Fabricación Propia) vs FAB (Tipo Mohican: Fabricado): confirmar si son el mismo concepto o dimensiones distintas