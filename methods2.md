# 📐 Cheatsheet — Metodologías Teóricas (Examen BD3)

> Cubre: HEFESTO · Data Warehouse (modelos) · DDD para NoSQL · Arquitecturas Big Data · Metodología DDD

---

## 🗂️ Índice

1. [Metodología HEFESTO — Data Warehouse paso a paso](#1-metodología-hefesto--data-warehouse-paso-a-paso)
2. [Modelos de Datos en Data Warehouse](#2-modelos-de-datos-en-data-warehouse)
3. [ETL — Extracción, Transformación y Carga](#3-etl--extracción-transformación-y-carga)
4. [Metodología DDD para NoSQL](#4-metodología-ddd-para-nosql)
5. [Arquitecturas Big Data](#5-arquitecturas-big-data)

---

## 1. Metodología HEFESTO — Data Warehouse paso a paso

> HEFESTO es una metodología iterativa e incremental para construir Data Warehouses basada en los requerimientos del negocio.

### Visión general del proceso

```
FASE 1              FASE 2              FASE 3              FASE 4
Análisis de    →    Análisis de    →    Modelo lógico  →    Integración
requerimientos      OLTP               del DW              de datos (ETL)
```

---

### FASE 1 — Análisis de requerimientos

**Objetivo:** Entender qué necesita el negocio, qué preguntas quieren responder.

#### Paso 1.1 — Identificar preguntas del negocio

Reunirse con los usuarios y recolectar preguntas tipo:
- ¿Cuánto vendimos por mes / por región / por producto?
- ¿Cuáles son los clientes con mayor consumo?
- ¿Cómo evolucionó el stock en el último año?

#### Paso 1.2 — Identificar indicadores (hechos)

Los indicadores son las **medidas numéricas** que se quieren analizar.

```
Pregunta: "¿Cuánto vendimos por mes?"
                 ↓
Indicador: MONTO_VENTA, CANTIDAD_VENDIDA
```

| Tipo de indicador | Descripción | Ejemplo |
|---|---|---|
| **Aditivo** | Se puede sumar en todas las dimensiones | Monto venta, cantidad |
| **Semi-aditivo** | Solo se suma en algunas dimensiones | Saldo de cuenta |
| **No aditivo** | No tiene sentido sumar | Precio unitario, ratio |

#### Paso 1.3 — Identificar perspectivas (dimensiones)

Las perspectivas son los **ejes de análisis** (el "por qué" o "según qué").

```
Pregunta: "¿Cuánto vendimos POR MES, POR REGIÓN, POR PRODUCTO?"
                                    ↓
Dimensiones: TIEMPO, GEOGRAFÍA, PRODUCTO
```

#### Diagrama de Análisis de Requerimientos

```
┌─────────────────────────────────────────────────────────────┐
│                    ANÁLISIS DE REQUERIMIENTOS               │
│                                                             │
│  Pregunta del negocio                                       │
│  "¿Total de ventas por producto, mes y ciudad?"             │
│                          │                                  │
│          ┌───────────────┼───────────────┐                  │
│          ▼               ▼               ▼                  │
│   INDICADORES      PERSPECTIVAS    PERSPECTIVAS             │
│   (hechos)         (dimensiones)   (dimensiones)            │
│                                                             │
│   • Total_venta    • Producto      • Tiempo                 │
│   • Cantidad       • Ciudad        • Cliente                │
└─────────────────────────────────────────────────────────────┘
```

---

### FASE 2 — Análisis de los OLTP (fuentes de datos)

**Objetivo:** Mapear los indicadores y dimensiones a las tablas de la BD operacional.

#### Paso 2.1 — Conformar indicadores

Buscar en el OLTP de dónde viene cada indicador:

```
Indicador: MONTO_VENTA
  → Tabla OLTP: detalle_venta.precio_unit * detalle_venta.cantidad
  → Cálculo: SUM(precio_unit * cantidad)
```

#### Paso 2.2 — Establecer correspondencias (dimensiones → OLTP)

```
Dimensión TIEMPO     → columna fecha_venta de tabla VENTAS
Dimensión PRODUCTO   → tabla PRODUCTOS (id, nombre, categoria)
Dimensión CLIENTE    → tabla CLIENTES (id, nombre, ciudad)
```

#### Paso 2.3 — Nivel de granularidad

Define qué tan detallada es la información en la tabla de hechos:

| Granularidad | Descripción | Trade-off |
|---|---|---|
| **Alta** (fino) | Registro por transacción individual | Más espacio, más detalle |
| **Media** | Registro por día por producto | Balance |
| **Baja** (gruesa) | Registro por mes por categoría | Menos espacio, menos detalle |

> **Regla:** Siempre elegir la granularidad más fina posible. Se puede agregar después, no se puede desagregar.

#### Paso 2.4 — Modelo de datos fuente (diagrama ER simplificado)

```
CLIENTES ──────< VENTAS >────── PRODUCTOS
                   │
              DETALLE_VENTA
                   │
              PRODUCTOS
```

---

### FASE 3 — Modelo lógico del DW

**Objetivo:** Diseñar el esquema estrella o copo de nieve del Data Warehouse.

#### Paso 3.1 — Definir la tabla de HECHOS

La tabla de hechos contiene:
- Las **claves foráneas** a todas las dimensiones
- Los **indicadores** (métricas numéricas)
- Opcionalmente: métricas derivadas

```sql
-- Tabla de hechos resultante
CREATE TABLE FACT_VENTAS (
    -- FK a dimensiones
    id_tiempo      INT,
    id_producto    INT,
    id_cliente     INT,
    id_geografia   INT,
    -- indicadores
    monto_venta    DECIMAL(12,2),
    cantidad       INT,
    costo          DECIMAL(12,2),
    -- métricas derivadas
    utilidad       AS (monto_venta - costo)
);
```

#### Paso 3.2 — Definir las tablas de DIMENSIONES

Cada dimensión tiene:
- Una **surrogate key** (clave sustituta, entero autoincremental)
- Los **atributos descriptivos** del negocio
- La **clave natural** del OLTP (para referencia)

```sql
CREATE TABLE DIM_TIEMPO (
    id_tiempo     INT PRIMARY KEY,   -- surrogate key
    fecha         DATE,
    dia           INT,
    mes           INT,
    trimestre     INT,
    anio          INT,
    nombre_mes    VARCHAR(20),
    dia_semana    VARCHAR(15)
);

CREATE TABLE DIM_PRODUCTO (
    id_producto   INT PRIMARY KEY,
    cod_producto  VARCHAR(20),       -- clave natural del OLTP
    nombre        VARCHAR(100),
    categoria     VARCHAR(50),
    subcategoria  VARCHAR(50),
    marca         VARCHAR(50)
);
```

#### Paso 3.3 — Jerarquías en dimensiones

Las dimensiones pueden tener jerarquías para drill-down / roll-up:

```
DIM_TIEMPO:     Año → Trimestre → Mes → Semana → Día
DIM_GEOGRAFÍA:  País → Departamento → Ciudad → Barrio
DIM_PRODUCTO:   División → Categoría → Subcategoría → Producto
```

#### Paso 3.4 — Diagrama del modelo estrella (ver Sección 2)

---

### FASE 4 — Integración de datos (ETL)

**Objetivo:** Extraer datos del OLTP, transformarlos y cargarlos en el DW.

#### Paso 4.1 — Extracción

- Identificar fuentes (OLTP, archivos CSV, APIs)
- Definir estrategia: **full load** (carga completa) o **incremental** (solo cambios)
- Manejar cambios lentos en dimensiones (SCD)

#### Paso 4.2 — Transformación

| Transformación | Descripción | Ejemplo |
|---|---|---|
| **Limpieza** | Corregir nulos, duplicados, formatos | `NULL → 'Sin categoría'` |
| **Conversión** | Cambiar tipos de datos | `string '2024-01-15' → DATE` |
| **Integración** | Unir de múltiples fuentes | JOIN de 3 sistemas OLTP |
| **Derivación** | Calcular nuevos campos | `utilidad = venta - costo` |
| **Generación de surrogate keys** | Crear claves sustitutas | `SEQUENCE` o `IDENTITY` |

#### Paso 4.3 — Carga

| Estrategia | Cuándo usar |
|---|---|
| **Full refresh** | DW pequeño, datos cambian mucho |
| **Append** | Hechos históricos (ventas del día) |
| **Upsert** | Dimensiones que pueden cambiar (SCD Tipo 1) |
| **Insert + versión** | Dimensiones con historial (SCD Tipo 2) |

#### Slowly Changing Dimensions (SCD)

| Tipo | Estrategia | Efecto |
|---|---|---|
| **Tipo 0** | Ignorar cambio | Sin historial |
| **Tipo 1** | Sobreescribir el valor anterior | Sin historial, simple |
| **Tipo 2** | Nueva fila con fechas de vigencia | Con historial completo |
| **Tipo 3** | Agregar columna "valor anterior" | Solo último cambio |

```sql
-- SCD Tipo 2 en DIM_CLIENTE
id_cliente   | nombre     | ciudad    | vigente_desde | vigente_hasta | activo
1            | Ana López  | La Paz    | 2020-01-01    | 2023-05-10    | 0
2            | Ana López  | Cochabamba| 2023-05-11    | NULL          | 1
```

---

### Resumen HEFESTO en una página

```
REQUERIMIENTOS          FUENTES OLTP          MODELO LÓGICO DW         ETL
─────────────────────────────────────────────────────────────────────────────
1. Preguntas del    →  4. Conformar       →  7. Tabla de hechos    →  10. Extraer
   negocio             indicadores            (FK + métricas)          de OLTP

2. Indicadores      →  5. Mapear          →  8. Tablas de          →  11. Transformar
   (métricas)          dimensiones a          dimensiones               (limpiar, unir,
                        OLTP                   (SK + atributos)          calcular)

3. Perspectivas     →  6. Granularidad    →  9. Diagrama           →  12. Cargar
   (dimensiones)        del DW                estrella/snowflake        en DW
```

---

## 2. Modelos de Datos en Data Warehouse

### 2.1 Esquema Estrella (Star Schema)

**Estructura:** Una tabla de hechos central conectada directamente a tablas de dimensiones desnormalizadas.

```
                    ┌──────────────┐
                    │  DIM_TIEMPO  │
                    │  id_tiempo   │
                    │  fecha       │
                    │  mes         │
                    │  anio        │
                    └──────┬───────┘
                           │
  ┌──────────────┐   ┌─────┴──────────┐   ┌──────────────┐
  │ DIM_PRODUCTO │   │  FACT_VENTAS   │   │ DIM_CLIENTE  │
  │ id_producto  ├───┤  id_tiempo  FK │   │ id_cliente   │
  │ nombre       │   │  id_producto FK├───┤ nombre       │
  │ categoria    │   │  id_cliente  FK│   │ ciudad       │
  │ marca        │   │  id_geo     FK │   │ pais         │
  └──────────────┘   │  monto_venta   │   └──────────────┘
                     │  cantidad      │
                     └───────┬────────┘
                             │
                    ┌────────┴─────────┐
                    │  DIM_GEOGRAFIA   │
                    │  id_geo          │
                    │  ciudad          │
                    │  departamento    │
                    │  pais            │
                    └──────────────────┘
```

| Ventaja | Desventaja |
|---|---|
| Consultas simples y rápidas | Dimensiones desnormalizadas (redundancia) |
| Fácil de entender | Mayor espacio en disco |
| Optimizado para BI | Menos flexible para cambios |

---

### 2.2 Esquema Copo de Nieve (Snowflake Schema)

**Estructura:** Las dimensiones están normalizadas (subdivididas en tablas adicionales).

```
                    ┌──────────────┐
                    │  DIM_MES     │   ← subdimensión
                    │  id_mes      │
                    │  nombre_mes  │
                    │  trimestre   │
                    └──────┬───────┘
                           │ FK
                    ┌──────┴───────┐
                    │  DIM_TIEMPO  │
                    │  id_tiempo   │
                    │  fecha       │
                    │  id_mes   FK │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │       FACT_VENTAS       │
              │   id_tiempo          FK │
              │   id_producto        FK │
              │   monto_venta           │
              └────────────┬────────────┘
                           │
                    ┌──────┴───────┐
                    │ DIM_PRODUCTO │
                    │ id_producto  │
                    │ id_categoria │   → DIM_CATEGORIA
                    └──────────────┘      id_categoria
                                          nombre
                                          id_division → DIM_DIVISION
```

| Ventaja | Desventaja |
|---|---|
| Menor redundancia | Consultas más complejas (más JOINs) |
| Más fácil mantener dimensiones | Más lento para reportes |
| Refleja jerarquías con claridad | Más difícil para usuarios finales |

---

### 2.3 Esquema Constelación (Galaxy Schema / Fact Constellation)

**Estructura:** Múltiples tablas de hechos que comparten dimensiones.

```
  DIM_TIEMPO       DIM_PRODUCTO      DIM_CLIENTE
      │                 │                │
      ├─────────────────┤                │
      │          FACT_VENTAS             │
      │          (monto, cantidad)  ─────┘
      │
      ├─────────────────────────────┐
      │          FACT_INVENTARIO    │
      │          (stock, movimiento)│
                        │
                   DIM_ALMACEN
```

**Cuándo usar:** Cuando hay múltiples procesos de negocio que comparten dimensiones (ventas + inventario + compras).

---

### 2.4 Tabla de Hechos sin Hechos (Factless Fact Table)

Solo contiene FKs a dimensiones, sin métricas numéricas. Registra **eventos** o **coberturas**.

```sql
-- ¿Qué alumnos asistieron a qué clases?
CREATE TABLE FACT_ASISTENCIA (
    id_alumno   INT,
    id_clase    INT,
    id_tiempo   INT
    -- sin métricas: el hecho de que existe el registro ES el dato
);
```

---

### 2.5 Cuándo usar cada modelo

| Modelo | Usar cuando... |
|---|---|
| **Estrella** | Prioridad en performance de consultas, usuarios finales acceden directo |
| **Copo de Nieve** | Dimensiones muy grandes, espacio en disco es crítico |
| **Constelación** | Múltiples tablas de hechos que comparten dimensiones |
| **Factless** | Registrar asistencia, cobertura, elegibilidad (solo el hecho importa) |

---

## 3. ETL — Extracción, Transformación y Carga

### 3.1 Flujo completo del ETL

```
FUENTES                 ÁREA DE STAGING          DATA WAREHOUSE
────────────────────────────────────────────────────────────────
OLTP Base de datos  →                        
Archivos CSV/Excel  →   Staging Area     →   DIM_TIEMPO
APIs externas       →   (datos crudos    →   DIM_PRODUCTO
Archivos planos     →    sin transformar) →   DIM_CLIENTE
                                         →   FACT_VENTAS
        EXTRACCIÓN          TRANSFORMACIÓN        CARGA
        (E)                 (T)                   (L)
```

### 3.2 En Pentaho Data Integration (PDI)

#### Conceptos Pentaho

| Concepto | Descripción |
|---|---|
| **Transformation** | Flujo de procesamiento de datos (pasos conectados) |
| **Job** | Orquesta transformaciones y tareas en secuencia |
| **Step** | Unidad de procesamiento dentro de una transformation |
| **Hop** | Conector entre steps |
| **Kettle** | Nombre original de PDI |

#### Steps más usados en Pentaho

| Step | Uso |
|---|---|
| **Table Input** | Leer de BD |
| **CSV File Input** | Leer CSV |
| **Table Output** | Escribir en BD |
| **Dimension Lookup/Update** | Manejar dimensiones SCD Tipo 1 y 2 |
| **Combination Lookup/Update** | Buscar o insertar en tabla de hechos |
| **Calculator** | Calcular campos derivados |
| **String Operations** | Limpiar strings |
| **Filter Rows** | Filtrar registros |
| **Merge Join** | JOIN entre dos flujos |
| **Sort Rows** | Ordenar antes de agrupar |
| **Group By** | Agregar (SUM, COUNT, AVG) |
| **Add Sequence** | Generar surrogate keys |

#### Flujo típico de ETL en Pentaho

```
[Table Input OLTP]
       │
[Select Values]          ← seleccionar solo columnas necesarias
       │
[String Trim / Clean]    ← limpieza
       │
[Calculator]             ← campos derivados (ej: monto = qty * precio)
       │
[Dimension Lookup]       ← obtener SK de DIM_PRODUCTO
       │
[Dimension Lookup]       ← obtener SK de DIM_TIEMPO
       │
[Combination Lookup/Update] ← insertar en FACT_VENTAS
```

---

## 4. Metodología DDD para NoSQL

> DDD (Domain-Driven Design) aplicado a NoSQL se centra en modelar datos según el dominio del negocio y los patrones de acceso, no según reglas de normalización.

### 4.1 Conceptos clave de DDD

| Concepto | Descripción | En NoSQL |
|---|---|---|
| **Domain** | Área del negocio que se modela | Sistema de e-commerce |
| **Bounded Context** | Límite donde un modelo tiene significado específico | Contexto de "Órdenes" vs "Catálogo" |
| **Entity** | Objeto con identidad única que persiste | `Pedido`, `Cliente`, `Producto` |
| **Value Object** | Objeto sin identidad propia, se define por sus valores | `Dirección`, `Precio`, `Rango de fechas` |
| **Aggregate** | Grupo de entidades tratadas como unidad transaccional | `Pedido` + `LineaDetalle` |
| **Aggregate Root** | Entidad principal del aggregate, único punto de acceso | `Pedido` (controla a `LineaDetalle`) |
| **Repository** | Interfaz de acceso a datos para un aggregate | `PedidoRepository.findById()` |
| **Domain Event** | Evento que ocurre en el dominio | `PedidoConfirmado`, `PagoRecibido` |

---

### 4.2 Proceso DDD para diseñar un modelo NoSQL

#### Paso 1 — Event Storming (identificar eventos del dominio)

Técnica colaborativa: en post-its, listar todos los **eventos** del sistema.

```
[Usuario registrado] → [Producto agregado al carrito] → [Pedido creado]
       → [Pago procesado] → [Pedido enviado] → [Pedido entregado]
```

#### Paso 2 — Identificar Aggregates

Agrupar entidades que siempre se acceden juntas:

```
AGGREGATE: Pedido
├── Pedido (aggregate root)
│   ├── id_pedido
│   ├── fecha
│   ├── estado
│   └── cliente_info (value object embebido)
└── LineasDetalle[] (colección interna)
    ├── producto_id (referencia)
    ├── nombre_producto (desnormalizado)
    ├── cantidad
    └── precio_unitario
```

#### Paso 3 — Bounded Contexts y sus modelos

```
┌─────────────────────┐     ┌─────────────────────┐
│  CATÁLOGO Context   │     │   ÓRDENES Context    │
│                     │     │                      │
│  Producto:          │     │  Pedido:             │
│  - id               │ ──→ │  - producto_id (ref) │
│  - nombre           │     │  - nombre (copia)    │
│  - descripción      │     │  - precio (copia)    │
│  - precio           │     │    (desnormalizado   │
│  - inventario       │     │     al momento       │
└─────────────────────┘     │     de la compra)    │
                            └─────────────────────┘
```

> **Regla DDD en NoSQL:** Cada bounded context puede tener su propia colección/tabla. Es válido (y recomendable) duplicar datos entre contextos para evitar joins entre microservicios.

#### Paso 4 — Definir patrones de acceso (Query Patterns)

Listar todas las consultas necesarias antes de diseñar el esquema:

```
QP-01: Obtener pedido con todas sus líneas por id_pedido
QP-02: Listar pedidos de un cliente (últimos 30 días)
QP-03: Buscar productos por categoría y rango de precio
QP-04: Ver historial de estados de un pedido
QP-05: Contar pedidos por estado (dashboard)
```

#### Paso 5 — Diseñar el modelo según los query patterns

```javascript
// MongoDB — diseñado para QP-01 y QP-04 (pedido completo de una vez)
{
  "_id": ObjectId("..."),
  "cliente": {                          // value object embebido
    "id": "USR-123",
    "nombre": "Ana López",
    "email": "ana@mail.com"
  },
  "estado_actual": "enviado",
  "historial_estados": [                // QP-04: directo sin join
    { "estado": "creado",  "fecha": ISODate("2024-01-10") },
    { "estado": "pagado",  "fecha": ISODate("2024-01-10") },
    { "estado": "enviado", "fecha": ISODate("2024-01-11") }
  ],
  "lineas": [                           // aggregate completo embebido
    { "producto_id": "PROD-ABC", "nombre": "Laptop HP", "cantidad": 1, "precio": 1200.00 },
    { "producto_id": "PROD-XYZ", "nombre": "Mouse", "cantidad": 2, "precio": 25.00 }
  ],
  "total": 1250.00,
  "fecha_creacion": ISODate("2024-01-10")
}
```

#### Paso 6 — Context Map (cómo se relacionan los bounded contexts)

```
CATÁLOGO ──[Conformist]──→ ÓRDENES
ÓRDENES  ──[Anti-Corruption Layer]──→ PAGOS (sistema externo)
USUARIOS ──[Shared Kernel]──→ ÓRDENES
```

| Relación | Descripción |
|---|---|
| **Conformist** | Un contexto adopta el modelo del otro sin cambios |
| **Anti-Corruption Layer** | Adaptador que traduce modelos entre contextos |
| **Shared Kernel** | Dos contextos comparten una parte del modelo |
| **Customer/Supplier** | Un contexto provee, el otro consume |

---

### 4.3 Diagrama de Aggregates y Repositories

```
┌────────────────────────────────┐
│   << Aggregate >>              │
│         Pedido                 │  ← Aggregate Root
│  ─────────────────────────     │
│  + id: PedidoId                │
│  + cliente: ClienteInfo        │  ← Value Object
│  + lineas: LineaDetalle[]      │  ← Entidades internas
│  + estado: EstadoPedido        │  ← Value Object (enum)
│  + total(): Money              │
│  + confirmar(): DomainEvent    │  ← emite Domain Event
└───────────────┬────────────────┘
                │ usa
        ┌───────▼────────┐
        │ PedidoRepository│
        │ + findById()    │   ← interfaz
        │ + save()        │
        │ + findByCliente()│
        └────────────────┘
```

---

## 5. Arquitecturas Big Data

### 5.1 Arquitectura Lambda

**Concepto:** Combina procesamiento en batch (datos históricos) y en tiempo real (datos recientes).

```
                    ┌─────────────────────────────────────────┐
                    │           FUENTES DE DATOS              │
                    │   (IoT, logs, clicks, transacciones)    │
                    └────────────────┬────────────────────────┘
                                     │
                    ┌────────────────▼────────────────────────┐
                    │         CAPA DE INGESTA                  │
                    │    (Kafka, Kinesis, Flume)               │
                    └──────────┬───────────────┬──────────────┘
                               │               │
              ┌────────────────▼──┐    ┌───────▼────────────────┐
              │   BATCH LAYER     │    │    SPEED LAYER          │
              │  (datos históricos│    │  (datos recientes,      │
              │   Hadoop / Spark) │    │   últimas horas)        │
              │  Procesamiento    │    │   Storm / Spark         │
              │  lento pero       │    │   Streaming             │
              │  preciso          │    │   Rápido, aproximado    │
              └────────┬──────────┘    └──────────┬─────────────┘
                       │                          │
              ┌────────▼──────────┐    ┌──────────▼─────────────┐
              │  BATCH VIEWS      │    │  REAL-TIME VIEWS        │
              │  (HBase, HDFS)    │    │  (Redis, Cassandra)     │
              └────────┬──────────┘    └──────────┬─────────────┘
                       │                          │
              ┌────────▼──────────────────────────▼─────────────┐
              │              SERVING LAYER                       │
              │     Combina batch + realtime para responder      │
              │          consultas del usuario                   │
              └──────────────────────────────────────────────────┘
```

| Capa | Responsabilidad | Tecnologías |
|---|---|---|
| **Batch Layer** | Reprocesa todos los datos históricos periódicamente | Hadoop, Spark Batch |
| **Speed Layer** | Procesa solo datos recientes para baja latencia | Spark Streaming, Storm, Flink |
| **Serving Layer** | Une resultados batch + speed y responde queries | HBase, Cassandra, ElasticSearch |

**Problema de Lambda:** Mantener **dos sistemas paralelos** (batch + streaming) con la misma lógica es complejo y costoso.

---

### 5.2 Arquitectura Kappa

**Concepto:** Elimina el batch layer. **Todo** es un stream, incluso el reprocesamiento histórico.

```
                    ┌─────────────────────────────────────────┐
                    │           FUENTES DE DATOS              │
                    └────────────────┬────────────────────────┘
                                     │
                    ┌────────────────▼────────────────────────┐
                    │         LOG DE EVENTOS INMUTABLE        │
                    │              (Kafka)                    │
                    │    Retiene TODO el historial            │
                    └────────────────┬────────────────────────┘
                                     │
                    ┌────────────────▼────────────────────────┐
                    │       STREAM PROCESSING                 │
                    │   (Flink, Spark Streaming, Kafka        │
                    │    Streams)                             │
                    │   Para reprocesar: nueva versión        │
                    │   lee desde el inicio del log           │
                    └────────────────┬────────────────────────┘
                                     │
                    ┌────────────────▼────────────────────────┐
                    │           SERVING LAYER                 │
                    │   (Cassandra, Redis, ElasticSearch)     │
                    └─────────────────────────────────────────┘
```

| Aspecto | Lambda | Kappa |
|---|---|---|
| Capas | 3 (batch + speed + serving) | 2 (stream + serving) |
| Complejidad | Alta (dos codebases) | Media (un solo pipeline) |
| Latencia | Baja (speed layer) | Baja |
| Reprocesamiento | Batch layer | Replay del log Kafka |
| Cuándo usar | Necesitas exactitud batch + tiempo real | Solo tiempo real o reprocesamiento simple |

---

### 5.3 Componentes del Ecosistema Big Data

```
INGESTA           ALMACENAMIENTO    PROCESAMIENTO     VISUALIZACIÓN
─────────         ──────────────    ─────────────     ─────────────
Kafka             HDFS              Spark             Power BI
Flume             S3                Flink             Tableau
Sqoop             HBase             MapReduce         Grafana
NiFi              Cassandra         Hive              Kibana
Kinesis           MongoDB           Pig
                  Delta Lake        Kafka Streams
```

### 5.4 Características de los datos Big Data (las 5 V)

| V | Descripción | Ejemplo |
|---|---|---|
| **Volume** | Gran cantidad de datos | Petabytes de logs |
| **Velocity** | Alta velocidad de generación | Millones de eventos/seg |
| **Variety** | Múltiples formatos y fuentes | JSON, video, sensores, texto |
| **Veracity** | Calidad e incertidumbre de los datos | Datos incompletos, ruidosos |
| **Value** | El dato debe generar valor al negocio | Insights, predicciones |

---

### 5.5 Cuándo usar Lambda vs Kappa

```
¿Necesitas procesamiento histórico batch MUY preciso?
        │
       SÍ → Lambda
        │
        NO
        │
¿Puedes modelar todo como un stream?
        │
       SÍ → Kappa (más simple de mantener)
        │
        NO → Lambda o arquitectura híbrida
```

---

## 📝 Resumen para justificar en el examen

### HEFESTO — frase de una línea por fase

| Fase | Una línea |
|---|---|
| F1 — Requerimientos | "Identificar preguntas del negocio → indicadores y dimensiones" |
| F2 — OLTP | "Mapear indicadores y dimensiones a tablas fuente, definir granularidad" |
| F3 — Modelo lógico | "Diseñar esquema estrella: tabla hechos (FK + métricas) + tablas dimensiones (SK + atributos)" |
| F4 — ETL | "Extraer de OLTP, transformar (limpiar, calcular, generar SK), cargar en DW" |

### DDD — frase de una línea por paso

| Paso | Una línea |
|---|---|
| Event Storming | "Identificar todos los eventos del dominio con el equipo" |
| Aggregates | "Agrupar entidades que siempre se acceden juntas bajo un Aggregate Root" |
| Bounded Contexts | "Definir límites donde cada modelo tiene un significado específico" |
| Query Patterns | "Listar todas las consultas antes de diseñar el esquema (query-first)" |
| Modelo | "Diseñar el documento/tabla según los query patterns identificados" |

### Modelos DW — cuándo usar

| Modelo | Una línea |
|---|---|
| Estrella | "Performance máxima, dimensiones desnormalizadas, BI simple" |
| Copo de nieve | "Dimensiones normalizadas, más JOINs, menos redundancia" |
| Constelación | "Múltiples tablas de hechos compartiendo dimensiones" |
| Factless | "Registrar eventos sin métricas (asistencia, cobertura)" |

---

*Cheatsheet generado para examen BD3 — INF-262 UMSA*