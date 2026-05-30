# ⚙️ Cheatsheet — Pentaho PDI + Power BI (Examen BD3)

> Cubre: Pentaho Data Integration (ETL) · Power BI (modelado, DAX, visualización)

---

## 🗂️ Índice

1. [Pentaho Data Integration — Conceptos base](#1-pentaho-data-integration--conceptos-base)
2. [Steps más importantes y cuándo usarlos](#2-steps-más-importantes-y-cuándo-usarlos)
3. [Flujos ETL típicos paso a paso](#3-flujos-etl-típicos-paso-a-paso)
4. [Jobs en Pentaho](#4-jobs-en-pentaho)
5. [Variables y configuración](#5-variables-y-configuración)
6. [Power BI — Conceptos base](#6-power-bi--conceptos-base)
7. [Power Query — Transformación de datos](#7-power-query--transformación-de-datos)
8. [Modelado en Power BI](#8-modelado-en-power-bi)
9. [DAX — Data Analysis Expressions](#9-dax--data-analysis-expressions)
10. [Visualizaciones y buenas prácticas](#10-visualizaciones-y-buenas-prácticas)
11. [Flujo completo: OLTP → Pentaho → DW → Power BI](#11-flujo-completo-oltp--pentaho--dw--power-bi)

---

## 1. Pentaho Data Integration — Conceptos base

### Jerarquía de objetos en PDI

```
PDI (Spoon)
├── Job (.kjb)                    ← orquesta el proceso completo
│   ├── Start
│   ├── Transformation (.ktr)     ← procesa y mueve datos
│   ├── Transformation (.ktr)
│   └── Send mail / Log / etc.
└── Transformation (.ktr)
    ├── Step → Step → Step        ← cada step es una operación
    └── Hops (flechas de conexión)
```

### Diferencia clave: Transformation vs Job

| | **Transformation** | **Job** |
|---|---|---|
| Archivo | `.ktr` | `.kjb` |
| Procesa | Filas de datos (streams) | Tareas y control de flujo |
| Flujo | Paralelo por defecto | Secuencial |
| Contiene | Steps conectados por hops | Entries (transformation, SQL, mail...) |
| Uso | ETL real (leer, transformar, escribir) | Orquestar varias transformations |

### Tipos de hops en Transformation

| Hop | Color | Significado |
|---|---|---|
| **Main stream** | Negro | Flujo principal de datos |
| **Error** | Rojo | Filas con error van por aquí |
| **Info** | Azul | Datos de referencia (lookup) |

---

## 2. Steps más importantes y cuándo usarlos

### Input (Lectura)

| Step | Cuándo usar |
|---|---|
| **Table Input** | Leer de cualquier BD con SQL personalizado |
| **CSV File Input** | Leer archivos CSV/TSV |
| **Excel Input** | Leer archivos .xls / .xlsx |
| **Get System Info** | Obtener fecha actual, nombre del servidor, etc. |
| **Generate Rows** | Crear filas sintéticas (pruebas o secuencias) |

```
Configuración Table Input:
- Connection: seleccionar conexión BD
- SQL: SELECT id, nombre, precio FROM productos WHERE activo = 1
- Replace variables in script: ✓ (si usa ${variables})
- Limit size: 0 (sin límite)
```

### Output (Escritura)

| Step | Cuándo usar |
|---|---|
| **Table Output** | Insertar filas en una tabla BD |
| **Insert / Update** | Insertar o actualizar según clave (upsert) |
| **Delete** | Eliminar filas por clave |
| **Dimension Lookup/Update** | Cargar dimensiones con SCD Tipo 1 o 2 |
| **Combination Lookup/Update** | Buscar/insertar en tabla de hechos |

### Transformación

| Step | Cuándo usar |
|---|---|
| **Select Values** | Seleccionar, renombrar, reordenar, cambiar tipo de columnas |
| **Calculator** | Calcular campos (suma, resta, fecha, concatenar) |
| **String Operations** | Trim, upper, lower, replace, pad |
| **Replace in String** | Reemplazar texto dentro de un campo |
| **Add Constants** | Agregar columnas con valor fijo |
| **Add Sequence** | Generar surrogate keys (1, 2, 3...) |
| **Filter Rows** | Bifurcar flujo según condición (como IF/WHERE) |
| **Switch / Case** | Enrutar filas a distintos destinos según valor |
| **If field value is null** | Reemplazar nulos |

### Lookup y Join

| Step | Cuándo usar |
|---|---|
| **Database Lookup** | Buscar valor en BD por clave (lento, row by row) |
| **Stream Lookup** | JOIN entre dos streams en memoria (rápido, sin BD) |
| **Merge Join** | JOIN formal entre dos inputs ordenados |
| **Merge Rows (diff)** | Comparar dos versiones del mismo dataset |

### Agregación y ordenamiento

| Step | Cuándo usar |
|---|---|
| **Sort Rows** | Ordenar antes de agrupar (REQUERIDO antes de Group By) |
| **Group By** | SUM, COUNT, AVG, MIN, MAX por grupo |
| **Unique Rows** | Eliminar duplicados (requiere Sort previo) |
| **Row Denormaliser** | Pivotar filas a columnas |
| **Row Normaliser** | Columnas a filas (unpivot) |

### Flujo y control

| Step | Cuándo usar |
|---|---|
| **Dummy** | Paso vacío (para debug o branch vacío) |
| **Abort** | Abortar transformation con error |
| **Write to Log** | Log de mensajes durante ejecución |
| **Delay Row** | Esperar N ms entre filas |

---

## 3. Flujos ETL típicos paso a paso

### 3.1 Cargar una Dimensión (SCD Tipo 1 — sobreescribir)

```
[Table Input]                   ← leer de OLTP
    │  SELECT id_prod, nombre, categoria, precio FROM productos
    │
[Select Values]                 ← renombrar/seleccionar campos
    │  id_prod → producto_nk (natural key)
    │
[Dimension Lookup/Update]       ← buscar o insertar en DIM_PRODUCTO
    │  Lookup field: producto_nk
    │  Technical key: id_producto (surrogate key, retorna al stream)
    │  Update fields: nombre, categoria, precio
    │  Type of update: Punch through (SCD Tipo 1)
    │
[Dummy o siguiente step]        ← continúa con SK disponible
```

**Configuración Dimension Lookup/Update para SCD Tipo 1:**
- Connection: conexión al DW
- Schema/Table: `DIM_PRODUCTO`
- Key fields: `producto_nk = id_prod` (campo OLTP → campo DIM)
- Technical key field: `id_producto` (autoincremental de la DIM)
- Fields to update: `nombre, categoria, precio`
- Enable the cache: ✓ (mejora performance)
- Preload cache: ✓

---

### 3.2 Cargar una Dimensión (SCD Tipo 2 — historial)

```
[Table Input OLTP]
    │
[Select Values]
    │
[Dimension Lookup/Update]
    │  Type of update: Insert (new version)
    │  Version field: version_num
    │  Date range start field: fecha_inicio
    │  Date range end field: fecha_fin
    │  Min date: 1900-01-01
    │  Max date: 9999-12-31
```

El step automáticamente:
1. Busca si existe por `natural_key`
2. Si no existe → inserta con `fecha_inicio = hoy`, `fecha_fin = max`
3. Si existe y hay cambio → cierra registro anterior (`fecha_fin = hoy`) e inserta nuevo

---

### 3.3 Cargar la Tabla de Hechos

```
[Table Input OLTP]
    │  SELECT v.id_venta, v.fecha, v.id_cliente, v.id_producto,
    │         d.cantidad, d.precio_unit, d.cantidad * d.precio_unit AS monto
    │  FROM ventas v JOIN detalle_venta d ON v.id_venta = d.id_venta
    │
[Select Values]                 ← asegurar tipos correctos
    │
[Database Lookup → DIM_TIEMPO]  ← fecha → id_tiempo
    │  Input: fecha
    │  Lookup: DIM_TIEMPO WHERE fecha = fecha → id_tiempo
    │
[Database Lookup → DIM_CLIENTE]
    │  Input: id_cliente
    │  Lookup: DIM_CLIENTE WHERE cliente_nk = id_cliente → id_cliente_sk
    │
[Database Lookup → DIM_PRODUCTO]
    │  Input: id_producto
    │  Lookup: DIM_PRODUCTO WHERE producto_nk = id_producto → id_producto_sk
    │
[Select Values]                 ← seleccionar solo SK e indicadores
    │  id_tiempo, id_cliente_sk, id_producto_sk, cantidad, monto
    │
[Combination Lookup/Update]     ← insertar en FACT_VENTAS si no existe
    │  Table: FACT_VENTAS
    │  Key fields: id_tiempo, id_cliente_sk, id_producto_sk
    │  (si ya existe la combinación, no duplica)
```

---

### 3.4 Transformaciones comunes

#### Generar DIM_TIEMPO completa

```
[Generate Rows]                 ← genera 1 fila
    │  limit: 1
    │
[Calculator]                    ← fecha_inicio = '2020-01-01'
    │
[Add Sequence]                  ← id_tiempo autoincremental
    │
[Calculator]                    ← extraer dia, mes, año, trimestre
    │  dia = DATE_WORKING_DAY(fecha, 0)
    │  mes = MONTH_OF_YEAR_0(fecha)
    │
[Table Output → DIM_TIEMPO]
```

> En la práctica, DIM_TIEMPO se genera con un script SQL o con un Excel precargado que se lee con **Excel Input**.

#### Limpiar campos nulos

```
[Table Input]
    │
[If field value is null]        ← campo: ciudad, reemplazar con: 'Sin ciudad'
    │
[String Operations]             ← trim, upper a campos de texto
    │
[Filter Rows]                   ← descartar filas con id_cliente NULL
    │  condición: id_cliente IS NOT NULL
    │  hop TRUE → continúa | hop FALSE → Dummy (descarte)
```

---

## 4. Jobs en Pentaho

### Entries más usadas en Jobs

| Entry | Uso |
|---|---|
| **Start** | Punto de inicio (obligatorio) |
| **Transformation** | Ejecutar un .ktr |
| **Job** | Ejecutar otro .kjb (jobs anidados) |
| **SQL** | Ejecutar SQL (truncate, create, etc.) |
| **Table Exists** | Verificar si una tabla existe |
| **File Exists** | Verificar si un archivo existe |
| **Send Mail** | Notificar por email |
| **Write to Log** | Registrar mensaje en log |
| **Abort Job** | Detener con error |
| **Success** | Fin exitoso |

### Flujo típico de un Job completo

```
[Start]
   │ (OK)
[SQL: TRUNCATE staging tables]
   │ (OK)
[Transformation: extracción OLTP → staging]
   │ (OK)                    │ (ERROR)
[Transformation: cargar DIM_TIEMPO]    [Send Mail: error ETL]
   │ (OK)                              │
[Transformation: cargar DIM_PRODUCTO]  [Abort Job]
   │ (OK)
[Transformation: cargar DIM_CLIENTE]
   │ (OK)
[Transformation: cargar FACT_VENTAS]
   │ (OK)
[Write to Log: ETL completado exitosamente]
   │
[Success]
```

### Hops en Jobs

| Hop | Color | Condición |
|---|---|---|
| **Unconditional** | Negro sólido | Siempre pasa (éxito o error) |
| **Follow when true** | Verde | Solo si el entry anterior fue exitoso |
| **Follow when false** | Rojo | Solo si el entry anterior falló |

---

## 5. Variables y configuración

### Variables en PDI

```bash
# En kettle.properties (home del usuario / .kettle/kettle.properties)
DB_HOST=localhost
DB_PORT=3306
DB_NAME=datawarehouse
DB_USER=etl_user
FECHA_CARGA=2024-01-01
```

### Usar variables en steps

```
Table Input SQL:
  SELECT * FROM ventas WHERE fecha >= '${FECHA_CARGA}'

Database connection:
  Host: ${DB_HOST}
  Port: ${DB_PORT}
  Database: ${DB_NAME}
```

### Pasar parámetros a una Transformation desde un Job

En el Job entry "Transformation":
- Pestaña **Parameters** → agregar nombre/valor
- En la Transformation → pestaña **Parameters** → declarar el parámetro

---

## 6. Power BI — Conceptos base

### Flujo de trabajo en Power BI Desktop

```
OBTENER DATOS    →    POWER QUERY     →    MODELO         →    REPORTES
(conectar fuente)     (transformar)         (relaciones,        (visualizaciones,
                                            DAX medidas)        filtros, slicers)

Get Data              Transform             Model View          Report View
└── SQL Server        └── Filtrar cols      └── Relaciones      └── Gráficos
└── Excel             └── Cambiar tipos     └── Medidas DAX     └── Tablas
└── CSV               └── Columnas calc.    └── Jerarquías      └── KPI cards
└── Web               └── Merge/Append                          └── Slicers
```

### Tres vistas principales

| Vista | Para qué |
|---|---|
| **Report View** | Crear páginas, arrastrar visualizaciones, configurar filtros |
| **Data View** | Ver los datos de las tablas, revisar columnas calculadas |
| **Model View** | Ver y configurar relaciones entre tablas |

---

## 7. Power Query — Transformación de datos

### Operaciones esenciales (interfaz + M)

#### Tipos de datos — siempre verificar al cargar

```
Texto          → abc
Número entero  → 123
Decimal        → 1.5
Fecha          → 📅
Fecha/Hora     → 📅⏰
Booleano       → True/False
```

#### Transformaciones más usadas

| Operación | Dónde | Para qué |
|---|---|---|
| **Remove Columns** | Home o click derecho | Eliminar columnas innecesarias |
| **Rename Columns** | Doble click en cabecera | Nombres descriptivos |
| **Change Type** | Home → Transform | Asegurar tipos correctos |
| **Filter Rows** | Flecha en cabecera | Excluir nulos, filtrar por valor |
| **Remove Duplicates** | Home | Limpiar duplicados |
| **Replace Values** | Transform | Reemplazar nulos o textos |
| **Split Column** | Transform | Separar por delimitador |
| **Merge Columns** | Transform | Concatenar columnas |
| **Add Column** | Add Column → Custom | Columna calculada en M |
| **Merge Queries** | Home | JOIN entre tablas |
| **Append Queries** | Home | UNION entre tablas |
| **Group By** | Transform | Agregar (SUM, COUNT...) |
| **Pivot / Unpivot** | Transform | Cambiar forma de la tabla |

#### Fórmula M para columna personalizada

```m
// En Add Column → Custom Column
// Ejemplo: extraer año de una fecha
Date.Year([fecha_venta])

// Concatenar nombre completo
[nombres] & " " & [apellidos]

// Calcular utilidad
[precio_venta] - [costo]

// If-Then-Else
if [estado] = "A" then "Activo" else "Inactivo"

// Reemplazar nulo
if [ciudad] = null then "Sin ciudad" else [ciudad]
```

#### Merge Queries (equivale a JOIN)

```
Home → Merge Queries
- Left table:  FACT_VENTAS  | columna: id_producto
- Right table: DIM_PRODUCTO | columna: id_producto
- Join kind: Left Outer (más común)
→ Expande la columna resultante seleccionando campos necesarios
```

---

## 8. Modelado en Power BI

### Tipos de relaciones

| Cardinalidad | Símbolo | Cuándo ocurre |
|---|---|---|
| **Muchos a uno (*:1)** | `* ─── 1` | FACT → DIM (lo más común) |
| **Uno a uno (1:1)** | `1 ─── 1` | Tablas divididas del mismo objeto |
| **Uno a muchos (1:*)** | `1 ─── *` | Igual que *:1 pero al revés |
| **Muchos a muchos (*:*)** | `* ─── *` | Evitar — usar tabla bridge |

### Dirección del filtro cruzado

| Dirección | Comportamiento |
|---|---|
| **Single** | La DIM filtra a la FACT (estándar, recomendado) |
| **Both** | Filtro va en ambas direcciones (usar con precaución) |

### Modelo estrella en Power BI (lo que debes tener)

```
DIM_TIEMPO (1) ────* FACT_VENTAS *──── (1) DIM_PRODUCTO
                          │
                    (1) DIM_CLIENTE
                          │
                    (1) DIM_GEOGRAFIA
```

**Reglas de buen modelado:**

```
✅ Las FK en FACT_VENTAS siempre al lado muchos (*)
✅ Las PK en dimensiones siempre al lado uno (1)
✅ No circular references
✅ Una sola tabla de fechas (DIM_TIEMPO) marcada como "Date Table"
✅ Nombres de tablas y columnas descriptivos (sin prefijos técnicos)
✅ Ocultar las FK de la FACT al usuario (solo las DIM son visibles)
```

### Marcar tabla de fechas

```
Model View → click derecho en DIM_TIEMPO
→ Mark as date table
→ Seleccionar columna de tipo Date
```

Esto activa funciones de inteligencia de tiempo (YTD, MTD, etc.) en DAX.

---

## 9. DAX — Data Analysis Expressions

### Dos tipos de cálculos DAX

| Tipo | Dónde | Evalúa en |
|---|---|---|
| **Columna calculada** | Table tools → New Column | Contexto de fila (row by row) |
| **Medida** | Table tools → New Measure | Contexto de filtro (según visual) |

> **Regla:** Siempre preferir **medidas** sobre columnas calculadas para métricas. Las medidas son dinámicas y no ocupan memoria estática.

---

### Medidas básicas

```dax
-- Suma simple
Total Ventas = SUM(FACT_VENTAS[monto_venta])

-- Conteo de filas
Cantidad Pedidos = COUNTROWS(FACT_VENTAS)

-- Conteo de valores distintos
Clientes Únicos = DISTINCTCOUNT(FACT_VENTAS[id_cliente])

-- Promedio
Ticket Promedio = AVERAGE(FACT_VENTAS[monto_venta])

-- Mínimo / Máximo
Venta Mínima = MIN(FACT_VENTAS[monto_venta])
Venta Máxima = MAX(FACT_VENTAS[monto_venta])

-- División con control de error
Margen % = DIVIDE([Utilidad], [Total Ventas], 0)
-- tercer argumento = valor si denominador es 0
```

---

### CALCULATE — la función más importante

> `CALCULATE` evalúa una expresión en un contexto de filtro modificado.

```dax
-- Sintaxis
CALCULATE(<expresión>, <filtro1>, <filtro2>, ...)

-- Ventas solo de categoría "Electrónico"
Ventas Electrónico = 
    CALCULATE(
        SUM(FACT_VENTAS[monto_venta]),
        DIM_PRODUCTO[categoria] = "Electrónico"
    )

-- Ventas del año 2024
Ventas 2024 = 
    CALCULATE(
        [Total Ventas],
        DIM_TIEMPO[anio] = 2024
    )

-- Ventas sin filtro de región (ALL elimina filtro)
Ventas Total Global = 
    CALCULATE([Total Ventas], ALL(DIM_GEOGRAFIA))

-- % del total
% del Total = 
    DIVIDE(
        [Total Ventas],
        CALCULATE([Total Ventas], ALL(DIM_PRODUCTO))
    )
```

---

### Inteligencia de tiempo (requiere tabla de fechas marcada)

```dax
-- Año hasta la fecha (YTD)
Ventas YTD = TOTALYTD([Total Ventas], DIM_TIEMPO[fecha])

-- Mes hasta la fecha (MTD)
Ventas MTD = TOTALMTD([Total Ventas], DIM_TIEMPO[fecha])

-- Trimestre hasta la fecha (QTD)
Ventas QTD = TOTALQTD([Total Ventas], DIM_TIEMPO[fecha])

-- Mismo período del año anterior
Ventas Año Anterior = 
    CALCULATE([Total Ventas], SAMEPERIODLASTYEAR(DIM_TIEMPO[fecha]))

-- Variación vs año anterior
Variación YoY = [Total Ventas] - [Ventas Año Anterior]

-- Variación % vs año anterior
Variación YoY % = 
    DIVIDE([Variación YoY], [Ventas Año Anterior], 0)

-- Acumulado desde inicio (Running Total)
Ventas Acumuladas = 
    CALCULATE(
        [Total Ventas],
        DATESYTD(DIM_TIEMPO[fecha])
    )
```

---

### Funciones de filtro

```dax
-- ALL: elimina todos los filtros de una tabla o columna
Ventas Sin Filtro Producto = CALCULATE([Total Ventas], ALL(DIM_PRODUCTO))

-- ALLEXCEPT: mantiene filtros excepto los indicados
Ventas Por Año = 
    CALCULATE([Total Ventas], ALLEXCEPT(DIM_TIEMPO, DIM_TIEMPO[anio]))

-- FILTER: filtro dinámico con condición compleja
Ventas Productos Caros = 
    CALCULATE(
        [Total Ventas],
        FILTER(DIM_PRODUCTO, DIM_PRODUCTO[precio] > 500)
    )

-- VALUES: lista de valores únicos del contexto actual
Año Seleccionado = SELECTEDVALUE(DIM_TIEMPO[anio], "Todos")
```

---

### Columnas calculadas (cuando sí tiene sentido)

```dax
-- En DIM_TIEMPO: nombre del mes con número para ordenar
Mes Nombre = FORMAT(DIM_TIEMPO[fecha], "MMMM")
Mes Numero = MONTH(DIM_TIEMPO[fecha])

-- En FACT_VENTAS: clasificar por monto
Segmento Venta = 
    IF(FACT_VENTAS[monto_venta] > 1000, "Alto",
       IF(FACT_VENTAS[monto_venta] > 500, "Medio", "Bajo"))

-- Concatenar en dimensión
Nombre Completo = DIM_CLIENTE[nombre] & " " & DIM_CLIENTE[apellido]
```

---

### Tabla de medidas (buena práctica)

Crear una tabla vacía llamada `_Medidas` y colocar todas las medidas ahí:

```dax
-- En Power BI Desktop: Enter Data → tabla vacía llamada "_Medidas"
-- Luego mover todas las medidas a esa tabla
-- Ventaja: las medidas aparecen juntas, separadas de las tablas de datos
```

---

## 10. Visualizaciones y buenas prácticas

### Visualizaciones más usadas

| Visual | Cuándo usar |
|---|---|
| **Card** | Un solo KPI (Total Ventas, Clientes, etc.) |
| **Bar Chart** | Comparar categorías (ventas por producto) |
| **Column Chart** | Comparar categorías temporales (ventas por mes) |
| **Line Chart** | Tendencia en el tiempo |
| **Combo Chart** | Comparar dos métricas con escalas distintas |
| **Pie / Donut** | Proporción de pocas categorías (máx 5-6) |
| **Matrix** | Tabla dinámica con totales (como pivot) |
| **Table** | Datos detallados con formato |
| **Slicer** | Filtro interactivo para el usuario |
| **Map** | Datos geográficos |
| **Treemap** | Jerarquía y proporción |
| **Scatter Plot** | Correlación entre dos métricas |
| **Waterfall** | Variaciones acumuladas (bridge) |
| **Gauge** | Progreso hacia una meta |

### Slicers — filtros interactivos

```
Tipos de slicer:
- List      → lista de valores con checkbox
- Dropdown  → desplegable (ahorra espacio)
- Between   → rango numérico o de fechas
- Relative Date → "últimos 30 días", "este año", etc.

Configurar: Visual → Format → Slicer settings → Style
```

### Jerarquías para drill-down

```
Model View → DIM_TIEMPO → Nueva jerarquía
Agregar en orden: Año → Trimestre → Mes → Día

En el visual: los botones ↓ y ↑ navegan la jerarquía
```

### Formato de números en medidas

```dax
-- Con formato en la medida (mejor práctica)
Total Ventas = 
    FORMAT(SUM(FACT_VENTAS[monto_venta]), "#,##0.00")

-- O configurar en Format pane del visual:
-- Measure Tools → Format → Moneda, Porcentaje, etc.
```

### Páginas del reporte — estructura recomendada para DW

```
Página 1: Resumen Ejecutivo
  └── Cards: Total Ventas, Clientes, Productos, Margen
  └── Line: Ventas por mes (YTD vs año anterior)
  └── Slicer: Año, Categoría

Página 2: Análisis de Ventas
  └── Bar: Top 10 productos
  └── Column: Ventas por mes
  └── Matrix: Ventas por categoría × región
  └── Slicer: Período, Región

Página 3: Análisis de Clientes
  └── Map: distribución geográfica
  └── Bar: Top clientes
  └── Scatter: Frecuencia vs Ticket promedio

Página 4: Detalle
  └── Table con drill-through
```

### Drill-through

```
Página destino (ej: Detalle Producto):
  Report View → Visualizations → Add drill-through fields
  → Agregar DIM_PRODUCTO[nombre]

En cualquier visual de otra página:
  Click derecho sobre un producto → Drill through → Detalle Producto
```

---

## 11. Flujo completo: OLTP → Pentaho → DW → Power BI

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         FLUJO COMPLETO                                   │
│                                                                          │
│  OLTP (SQL Server / MySQL)                                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                         │
│  │  Ventas    │  │ Productos  │  │  Clientes  │                         │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                        │
│        └───────────────┴───────────────┘                                │
│                         │                                                │
│                    PENTAHO PDI                                           │
│  ┌──────────────────────▼────────────────────────────┐                  │
│  │  Job: ETL_Completo.kjb                            │                  │
│  │  ┌─────────────────────────────────────────────┐  │                  │
│  │  │  Transformation: carga_dim_tiempo.ktr        │  │                  │
│  │  │  Transformation: carga_dim_producto.ktr      │  │                  │
│  │  │  Transformation: carga_dim_cliente.ktr       │  │                  │
│  │  │  Transformation: carga_fact_ventas.ktr       │  │                  │
│  │  └─────────────────────────────────────────────┘  │                  │
│  └──────────────────────┬────────────────────────────┘                  │
│                         │                                                │
│                  DATA WAREHOUSE                                          │
│  ┌──────────────────────▼────────────────────────────┐                  │
│  │  DIM_TIEMPO   DIM_PRODUCTO   DIM_CLIENTE          │                  │
│  │                    FACT_VENTAS                    │                  │
│  └──────────────────────┬────────────────────────────┘                  │
│                         │                                                │
│                      POWER BI                                            │
│  ┌──────────────────────▼────────────────────────────┐                  │
│  │  Get Data → SQL Server → Import                   │                  │
│  │  Power Query → validar tipos y transformar        │                  │
│  │  Model View → verificar relaciones *:1             │                  │
│  │  DAX → crear medidas                              │                  │
│  │  Report → visualizaciones e interactividad        │                  │
│  └──────────────────────────────────────────────────┘                  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 📝 Resumen ultra-rápido para el examen

### Pentaho — orden de carga obligatorio

```
1° DIM_TIEMPO         (no depende de nada)
2° Dimensiones simples (DIM_PRODUCTO, DIM_CLIENTE, DIM_GEO...)
3° FACT_VENTAS         (depende de todas las dimensiones)
```

> **Jamás cargar la tabla de hechos antes que las dimensiones** — los lookups de SK fallarán.

### DAX — las 5 funciones que más salen

| Función | Para qué |
|---|---|
| `SUM` | Suma de una columna numérica |
| `CALCULATE` | Modificar el contexto de filtro |
| `DIVIDE` | División sin error si denominador = 0 |
| `TOTALYTD` | Acumulado del año hasta la fecha |
| `SAMEPERIODLASTYEAR` | Comparar con el mismo período del año anterior |

### Power BI — checklist antes de publicar

```
□ Tipos de datos correctos en Power Query
□ Relaciones *:1 sin circulares en Model View
□ DIM_TIEMPO marcada como Date Table
□ Medidas en tabla _Medidas
□ Nombres descriptivos en tablas y columnas
□ FK ocultas en la FACT (solo visibles para el modelo)
□ Slicers configurados (año, categoría mínimo)
□ Formato de números aplicado (moneda, porcentaje)
```

---

*Cheatsheet generado para examen BD3 — INF-262 UMSA*
