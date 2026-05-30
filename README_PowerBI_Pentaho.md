# 📊 Cheatsheet — Power BI & Pentaho

> **Examen:** DataWarehouse · ETL · Visualización | Herramientas de extracción, transformación, carga y análisis de datos.

---

## Índice
1. [Pentaho — Visión General](#1-pentaho--visión-general)
2. [Pentaho Data Integration (Spoon/Kettle) — ETL paso a paso](#2-pentaho-data-integration-spoonkettle--etl-paso-a-paso)
3. [Transformaciones en Pentaho](#3-transformaciones-en-pentaho)
4. [Jobs en Pentaho](#4-jobs-en-pentaho)
5. [Power BI — Visión General](#5-power-bi--visión-general)
6. [Power BI — Flujo completo paso a paso](#6-power-bi--flujo-completo-paso-a-paso)
7. [Power Query (ETL en Power BI)](#7-power-query-etl-en-power-bi)
8. [Modelado de Datos en Power BI](#8-modelado-de-datos-en-power-bi)
9. [DAX — Lenguaje de Fórmulas](#9-dax--lenguaje-de-fórmulas)
10. [Visualizaciones y Reportes](#10-visualizaciones-y-reportes)
11. [Pentaho vs Power BI — Comparación](#11-pentaho-vs-power-bi--comparación)

---

## 1. Pentaho — Visión General

> **Definición:** Suite open-source de Business Intelligence y ETL. Integra datos desde múltiples fuentes, los transforma y los carga en DW o Data Marts.

### Componentes principales

| Componente | Nombre alternativo | Función |
|------------|-------------------|---------|
| **Pentaho Data Integration** | Kettle / Spoon | ETL: Extracción, Transformación y Carga |
| **Pentaho Report Designer** | PRD | Diseño de reportes |
| **Pentaho BI Server** | BA Server | Servidor de reportes y dashboards |
| **Pentaho Metadata Editor** | PME | Capa semántica sobre BD |
| **Schema Workbench** | PSW | Diseño de cubos OLAP (Mondrian) |

### Conceptos clave Pentaho ETL

| Concepto | Descripción |
|----------|-------------|
| **Transformación** | Flujo de datos en tiempo real. Procesa filas paso a paso entre pasos (Steps). |
| **Job** | Orquestador de tareas. Ejecuta transformaciones, scripts, FTP, emails en secuencia. |
| **Step** | Unidad mínima de una Transformación (ej: Table Input, Sort Rows, Merge Join). |
| **Entry** | Unidad mínima de un Job (ej: Start, Transformation, Execute SQL). |
| **Hop** | Conector entre Steps (Transformación) o Entries (Job). |
| **Repository** | Repositorio central para guardar transformaciones y jobs. |

---

## 2. Pentaho Data Integration (Spoon/Kettle) — ETL paso a paso

### Flujo ETL completo

```
[Fuente OLTP / CSV / API]
        │
        ▼
  ┌─────────────┐
  │  EXTRACCIÓN │  ← Table Input, CSV Input, Excel Input, REST Client
  └──────┬──────┘
         │
         ▼
  ┌───────────────────┐
  │  TRANSFORMACIÓN   │  ← Filter, Sort, Join, Lookup, Calculator, Data Validator
  └──────────┬────────┘
             │
             ▼
       ┌───────────┐
       │   CARGA   │  ← Table Output, Insert/Update, Bulk Load
       └───────────┘
              │
              ▼
     [Data Warehouse / Data Mart]
```

### Pasos para crear una Transformación en Spoon

| # | Paso | Detalle |
|---|------|---------|
| 01 | **Abrir Spoon** | Ejecutar `spoon.sh` (Linux) o `Spoon.bat` (Windows). Conectar al repositorio o trabajar en local. |
| 02 | **Nueva Transformación** | `File → New → Transformation`. Se abre el canvas. |
| 03 | **Configurar conexión a BD** | `Edit → Connection` → Tipo de BD, host, puerto, usuario, contraseña. Probar conexión. |
| 04 | **Agregar Step de Entrada** | Arrastrar desde el panel izquierdo: `Table Input` (SQL), `CSV Input`, `Excel Input`, `JSON Input`. |
| 05 | **Configurar el Step de Entrada** | Doble clic → escribir query SQL o seleccionar archivo. Preview para verificar datos. |
| 06 | **Agregar Steps de Transformación** | Conectar con Hop (Ctrl+arrastrar): `Filter Rows`, `Sort Rows`, `Merge Join`, `Lookup`, `Calculator`, `String Operations`, `Replace in String`, `Data Validator`. |
| 07 | **Agregar Step de Salida** | `Table Output` / `Insert / Update` / `Dimension Lookup/Update` (para SCD). |
| 08 | **Configurar Step de Salida** | Seleccionar tabla destino, mapear campos origen → destino. |
| 09 | **Ejecutar y monitorear** | `F9` o botón Play. Panel de métricas muestra filas leídas, escritas, errores. |
| 10 | **Guardar** | Guardar como `.ktr` (transformación). |

---

## 3. Transformaciones en Pentaho

### Steps más importantes

| Categoría | Step | Función |
|-----------|------|---------|
| **Input** | Table Input | Ejecuta SQL sobre BD configurada |
| **Input** | CSV File Input | Lee archivos CSV/TXT |
| **Input** | Microsoft Excel Input | Lee hojas de Excel |
| **Input** | JSON Input | Lee archivos o campos JSON |
| **Transform** | Filter Rows | Filtra filas por condición (como WHERE) |
| **Transform** | Sort Rows | Ordena filas por columnas |
| **Transform** | Select Values | Selecciona, renombra, reordena o elimina columnas |
| **Transform** | Calculator | Crea columnas calculadas (sumas, fechas, conversiones) |
| **Transform** | String Operations | Trim, upper, lower, substring, replace |
| **Transform** | Replace in String | Reemplaza texto en columnas string |
| **Transform** | Add Sequence | Agrega un número secuencial (para surrogate keys) |
| **Transform** | Value Mapper | Mapea valores: "M" → "Masculino" |
| **Lookup** | Database Lookup | Busca valor en otra tabla (tipo LEFT JOIN) |
| **Lookup** | Dimension Lookup/Update | Manejo de SCD Tipo 1 y Tipo 2 automático |
| **Join** | Merge Join | Join entre dos streams ordenados |
| **Join** | Stream Lookup | Lookup en memoria entre streams |
| **Output** | Table Output | INSERT en tabla destino |
| **Output** | Insert / Update | INSERT o UPDATE según si existe la fila |
| **Output** | Bulk Load | Carga masiva (MySQL, PostgreSQL) |
| **Output** | Microsoft Excel Writer | Escribe resultado en Excel |

### Manejo de SCD (Slowly Changing Dimensions) con Pentaho

| Tipo SCD | Step | Comportamiento |
|----------|------|---------------|
| **Tipo 1** (sobreescribir) | Insert/Update | Actualiza el registro existente, se pierde el historial |
| **Tipo 2** (historizar) | Dimension Lookup/Update | Cierra el registro anterior (`date_to`) y crea uno nuevo |
| **Tipo 3** (columna adicional) | Update + Calculator | Agrega columna `prev_value`, actualiza ambas columnas |

---

## 4. Jobs en Pentaho

> **Jobs** orquestan múltiples transformaciones y tareas en secuencia con manejo de errores y condiciones.

### Pasos para crear un Job

| # | Paso | Detalle |
|---|------|---------|
| 01 | **Nuevo Job** | `File → New → Job`. Canvas diferente al de transformaciones. |
| 02 | **Agregar Start** | Entry `Start` siempre es el punto de entrada. |
| 03 | **Agregar Transformation Entry** | Arrastra `Transformation` → doble clic → seleccionar `.ktr`. |
| 04 | **Conectar Entries** | Hop verde = éxito (`→`), Hop rojo = error (`✗`), Hop gris = incondicional. |
| 05 | **Agregar lógica de error** | Conectar hop de error a `Write to Log`, `Send Email`, o `Abort Job`. |
| 06 | **Agregar tareas adicionales** | `Execute SQL Script`, `FTP Get`, `Check if file exists`, `Set Variable`. |
| 07 | **Ejecutar** | `F9`. Cada entry muestra ✓ (éxito) o ✗ (error) en tiempo real. |
| 08 | **Programar ejecución** | Desde BA Server o con cron + `kitchen.sh` (CLI para jobs). |

### Entries más usadas en Jobs

| Entry | Función |
|-------|---------|
| **Start** | Punto de inicio obligatorio |
| **Transformation** | Ejecuta un archivo `.ktr` |
| **Job** | Ejecuta un sub-job `.kjb` |
| **Execute SQL Script** | Ejecuta SQL directamente (truncate, create, etc.) |
| **Check if file exists** | Valida existencia de archivo antes de procesar |
| **Write to Log** | Escribe mensaje en el log |
| **Send Email** | Notifica por correo al finalizar o en error |
| **Set Variable** | Define variables de entorno para usar en transformaciones |
| **Abort Job** | Detiene el job con error controlado |

---

## 5. Power BI — Visión General

> **Definición:** Herramienta de Microsoft para Business Intelligence. Conecta a múltiples fuentes, transforma datos con Power Query, modela con relaciones y publica reportes interactivos.

### Componentes principales

| Componente | Función |
|------------|---------|
| **Power BI Desktop** | Aplicación local para crear reportes (.pbix) |
| **Power Query (M)** | ETL visual: limpieza y transformación de datos |
| **Modelo de Datos** | Tablas, relaciones, jerarquías (como un DW pequeño) |
| **DAX** | Lenguaje de fórmulas para medidas y columnas calculadas |
| **Visualizaciones** | Gráficos, tablas, mapas, KPIs interactivos |
| **Power BI Service** | Plataforma web para publicar, compartir y programar actualizaciones |
| **Power BI Gateway** | Puente entre fuentes locales y Power BI Service |

### Capas de Power BI

```
[Fuentes de datos]
       │
       ▼
[Power Query]   ← ETL: conectar, limpiar, transformar (lenguaje M)
       │
       ▼
[Modelo de Datos] ← Tablas, relaciones, jerarquías, formato
       │
       ▼
[DAX]           ← Medidas, KPIs, columnas calculadas, tablas calculadas
       │
       ▼
[Visualizaciones] ← Reportes interactivos, dashboards
       │
       ▼
[Power BI Service] ← Publicación, sharing, refresh programado
```

---

## 6. Power BI — Flujo completo paso a paso

| # | Paso | Detalle |
|---|------|---------|
| 01 | **Obtener datos** | `Inicio → Obtener datos` → Elegir fuente: SQL Server, Excel, CSV, Web, API, SharePoint, etc. |
| 02 | **Transformar en Power Query** | Clic en `Transformar datos` → Se abre el Editor de Power Query. Limpiar, filtrar, dar tipo. |
| 03 | **Cargar datos al modelo** | `Cerrar y aplicar` → Los datos se cargan al modelo interno de Power BI. |
| 04 | **Crear relaciones** | Vista `Modelo` → Arrastrar campo clave entre tablas. Definir cardinalidad y dirección de filtro. |
| 05 | **Crear medidas DAX** | Vista `Datos` → `Nueva medida` → Escribir fórmula DAX (ej: `Ventas Total = SUM(Ventas[Monto])`). |
| 06 | **Construir visualizaciones** | Vista `Informe` → Seleccionar visual → Arrastrar campos a Eje, Valores, Leyenda. |
| 07 | **Agregar filtros e interacciones** | Panel Filtros, Segmentadores (Slicers), botones de navegación. |
| 08 | **Publicar** | `Inicio → Publicar` → Elegir área de trabajo en Power BI Service. |
| 09 | **Programar actualización** | En Power BI Service → Dataset → Programar actualización (requiere Gateway para fuentes locales). |

---

## 7. Power Query (ETL en Power BI)

> **Power Query** usa el lenguaje **M** internamente. Las transformaciones se aplican como pasos secuenciales registrados.

### Transformaciones más usadas

| Transformación | Dónde está | Qué hace |
|---------------|------------|----------|
| **Cambiar tipo** | Inicio / columna | Asigna tipo: Texto, Número, Fecha, Booleano |
| **Quitar filas vacías** | Inicio → Quitar filas | Elimina filas nulas o vacías |
| **Filtrar filas** | Flecha en encabezado | Filtra por valor o condición |
| **Dividir columna** | Transformar | Divide por delimitador o número de caracteres |
| **Combinar columnas** | Agregar columna | Concatena varias columnas en una |
| **Columna personalizada** | Agregar columna | Fórmula M personalizada: `= [Precio] * [Cantidad]` |
| **Columna condicional** | Agregar columna | IF/ELSE visual sin escribir M |
| **Agrupar por** | Transformar | Agrega: suma, conteo, promedio por grupo |
| **Combinar consultas (Merge)** | Inicio | JOIN entre dos tablas (Left, Inner, Full) |
| **Anexar consultas (Append)** | Inicio | UNION de dos tablas con mismas columnas |
| **Dinamizar / Anular dinamización** | Transformar | Pivot / Unpivot de columnas |
| **Reemplazar valores** | Transformar | Reemplaza texto o nulos por otro valor |
| **Promover encabezados** | Transformar | Usa primera fila como nombres de columnas |

### Pasos en Power Query — Panel Pasos Aplicados

```
Origen
  └→ Tipo cambiado
       └→ Filas filtradas
            └→ Columna agregada
                 └→ Columnas eliminadas
                      └→ [resultado final]
```
> Cada paso es reversible. Se puede editar o eliminar cualquier paso del historial.

---

## 8. Modelado de Datos en Power BI

> El modelo de Power BI actúa como un **Mini Data Warehouse** en memoria (motor VertiPaq columnar).

### Tipos de tablas

| Tipo | Función | Ejemplo |
|------|---------|---------|
| **Tabla de hechos** | Métricas, transacciones. Muchas filas. | `Ventas`, `Pedidos` |
| **Tabla de dimensiones** | Atributos descriptivos. Pocas filas. | `Clientes`, `Productos`, `Fechas` |
| **Tabla de fechas** | Dimensión de tiempo. Obligatoria para inteligencia de tiempo en DAX. | `DimFecha` |
| **Tabla calculada** | Creada con DAX, vive en el modelo. | `= CALENDAR(DATE(2020,1,1), TODAY())` |

### Tipos de relaciones

| Cardinalidad | Descripción | Uso típico |
|-------------|-------------|-----------|
| **Muchos a uno (*:1)** | Varios registros en hechos → un registro en dimensión | Hechos → Dimensión (estándar) |
| **Uno a uno (1:1)** | Un registro en cada tabla | Tablas de configuración |
| **Muchos a muchos (*:*)** | Requiere tabla puente | Relaciones complejas |

### Dirección del filtro cruzado

| Dirección | Comportamiento |
|-----------|---------------|
| **Única** | Dimensión filtra a Hechos (recomendado) |
| **Ambas** | Filtro en ambas direcciones (puede causar ambigüedad) |

### Buenas prácticas de modelado

- ✅ Usar esquema **estrella**: dimensiones directamente a hechos
- ✅ Siempre incluir una **tabla de fechas** marcada como tal
- ✅ Relaciones **Muchos a Uno** desde hechos a dimensiones
- ✅ Ocultar columnas FK en hechos (solo visibles en dimensiones)
- ❌ Evitar relaciones circulares
- ❌ Evitar muchos-a-muchos sin tabla puente

---

## 9. DAX — Lenguaje de Fórmulas

> **DAX** (Data Analysis Expressions): lenguaje para crear **Medidas**, **Columnas calculadas** y **Tablas calculadas**.

### Diferencia: Medida vs Columna calculada

| | **Medida** | **Columna calculada** |
|---|------------|----------------------|
| Se calcula | En tiempo de consulta (dinámico) | Al cargar datos (estático) |
| Contexto | Depende del filtro del visual | Fila por fila |
| Almacena datos | No (se calcula al vuelo) | Sí (ocupa memoria) |
| Uso típico | KPIs, totales, porcentajes | Categorías, flags, concatenaciones |

### Funciones DAX esenciales

| Categoría | Función | Ejemplo |
|-----------|---------|---------|
| **Agregación** | `SUM` | `= SUM(Ventas[Monto])` |
| **Agregación** | `AVERAGE` | `= AVERAGE(Ventas[Precio])` |
| **Agregación** | `COUNT` | `= COUNT(Ventas[ID])` |
| **Agregación** | `DISTINCTCOUNT` | `= DISTINCTCOUNT(Ventas[ClienteID])` |
| **Filtro** | `CALCULATE` | `= CALCULATE(SUM(Ventas[Monto]), Fechas[Año]=2024)` |
| **Filtro** | `FILTER` | `= FILTER(Ventas, Ventas[Monto] > 1000)` |
| **Filtro** | `ALL` | `= CALCULATE([Total], ALL(Fechas))` — elimina filtros |
| **Tiempo** | `TOTALYTD` | `= TOTALYTD(SUM(Ventas[Monto]), Fechas[Fecha])` |
| **Tiempo** | `SAMEPERIODLASTYEAR` | `= CALCULATE([Total], SAMEPERIODLASTYEAR(Fechas[Fecha]))` |
| **Tiempo** | `DATEADD` | `= CALCULATE([Total], DATEADD(Fechas[Fecha], -1, YEAR))` |
| **Lógica** | `IF` | `= IF([Ventas Total] > 10000, "Alto", "Bajo")` |
| **Texto** | `CONCATENATE` | `= CONCATENATE([Nombre], " " & [Apellido])` |
| **Tabla** | `RELATED` | `= RELATED(Productos[Categoría])` — trae valor de tabla relacionada |
| **Tabla** | `SUMMARIZE` | Agrupa tabla por columnas |
| **Tabla** | `CALENDAR` | `= CALENDAR(DATE(2020,1,1), TODAY())` — tabla de fechas |

### Medidas calculadas comunes

```dax
-- Ventas totales
Ventas Total = SUM(Ventas[Monto])

-- % del total general
% del Total = DIVIDE([Ventas Total], CALCULATE([Ventas Total], ALL(Ventas)))

-- Ventas año anterior
Ventas Año Anterior = CALCULATE([Ventas Total], SAMEPERIODLASTYEAR(Fechas[Fecha]))

-- Variación vs año anterior
Variación YoY = [Ventas Total] - [Ventas Año Anterior]

-- Ventas acumuladas en el año
Ventas YTD = TOTALYTD([Ventas Total], Fechas[Fecha])
```

---

## 10. Visualizaciones y Reportes

### Visualizaciones principales en Power BI

| Visual | Úsalo para... |
|--------|--------------|
| **Gráfico de barras/columnas** | Comparar categorías |
| **Gráfico de líneas** | Tendencias en el tiempo |
| **Gráfico de áreas** | Tendencias con énfasis en volumen |
| **Gráfico circular / anillo** | Proporciones (máx. 5-6 categorías) |
| **Mapa** | Datos geográficos |
| **Tarjeta (Card)** | Un KPI o valor único |
| **Tarjeta de KPI** | Valor actual vs objetivo vs tendencia |
| **Tabla / Matriz** | Datos tabulares detallados / pivotados |
| **Treemap** | Jerarquías proporcionales |
| **Dispersión** | Correlación entre dos medidas |
| **Segmentador (Slicer)** | Filtro visual interactivo |
| **Cascada (Waterfall)** | Contribuciones positivas/negativas a un total |

### Filtros en Power BI

| Tipo | Alcance |
|------|---------|
| **Filtro de visual** | Solo afecta ese visual |
| **Filtro de página** | Afecta todos los visuales de la página |
| **Filtro de informe** | Afecta todas las páginas |
| **Segmentador (Slicer)** | El usuario elige el filtro interactivamente |
| **Filtro de obtención de detalles** | Filtra al navegar entre páginas (drill-through) |

### Buenas prácticas de reportes

- ✅ Máximo **5-7 visuales** por página para no saturar
- ✅ Incluir **segmentadores** de Fecha y categorías clave
- ✅ Usar **jerarquías** para drill-down (Año → Trimestre → Mes)
- ✅ Consistencia de colores por categoría en todo el reporte
- ✅ Títulos descriptivos en cada visual
- ❌ Evitar gráficos circulares con más de 5 categorías
- ❌ Evitar demasiados colores diferentes

---

## 11. Pentaho vs Power BI — Comparación

| Criterio | Pentaho | Power BI |
|---------|---------|---------|
| **Función principal** | ETL + BI Server | BI + Visualización + ETL básico |
| **ETL** | Muy potente (Kettle/Spoon) | Power Query (bueno para transformaciones simples-medias) |
| **Visualización** | Reportes estáticos/programados | Dashboards interactivos |
| **Lenguaje de fórmulas** | Groovy/JavaScript en Steps | DAX |
| **Lenguaje ETL** | Interfaz gráfica + Groovy | Power Query (M) |
| **Licencia** | Open-source (Community) / Pago | Freemium (Desktop gratis, Service pago) |
| **Escalabilidad ETL** | Alta (Big Data, Hadoop, NoSQL) | Media |
| **Curva de aprendizaje** | Media-alta | Media |
| **Integración DW** | Nativa (carga directa a DW) | Conexión + importación/DirectQuery |
| **Sharing** | BA Server | Power BI Service (nube Microsoft) |
| **Mejor para** | Procesos ETL complejos, DW corporativos | Reportes de negocio, self-service BI |

### Cuándo usar cada uno en un proyecto DW

```
PENTAHO → Proceso ETL            POWER BI → Análisis y Reportes
    │                                  │
    ├─ Cargas nocturnas                ├─ Dashboards ejecutivos
    ├─ Múltiples fuentes heterogéneas  ├─ Reportes de autoservicio
    ├─ Transformaciones complejas      ├─ KPIs interactivos
    ├─ Manejo de SCD                   ├─ Drill-down temporal
    └─ Carga al DW / Data Mart         └─ Publicación en nube
```

> ⚡ **En un stack típico:** Pentaho hace el ETL y carga el DW → Power BI se conecta al DW y construye los reportes.

---

*Cheatsheet generado para examen de DataWarehouse · Power BI · Pentaho ETL*
