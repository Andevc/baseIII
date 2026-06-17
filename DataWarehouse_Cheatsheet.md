# 🏛️ Cheatsheet — Data Warehouse
> Modelado, Metodologías, HEFESTO y Código | BD III · UMSA Informática

---

## 1. ¿Qué es un Data Warehouse?

Un **Data Warehouse (DW)** es una base de datos **analítica y centralizada** que integra datos históricos provenientes de múltiples fuentes operacionales (OLTP), optimizada para **consultas y análisis**, no para transacciones.

> ⚡ **No es una BD transaccional.** No se actualiza fila por fila en tiempo real. Se carga en lotes (ETL) y se usa para reportes, dashboards y minería de datos.

### Características clave (definición de Inmon)

| Característica | Significado |
|---|---|
| **Orientado a temas** | Organizado por áreas de negocio (ventas, compras, RRHH), no por aplicaciones |
| **Integrado** | Datos de múltiples fuentes unificados en un formato consistente |
| **Variante en el tiempo** | Guarda historia; cada registro tiene una dimensión temporal |
| **No volátil** | Los datos no se modifican ni eliminan; solo se agregan |

### DW vs BD Operacional (OLTP vs OLAP)

| | OLTP | OLAP / DW |
|---|---|---|
| **Propósito** | Operaciones del día a día | Análisis e informes |
| **Consultas** | Cortas, muchas por segundo | Largas, pocas y complejas |
| **Datos** | Actuales | Históricos |
| **Diseño** | Normalizado (3FN) | Desnormalizado (estrella/copo) |
| **Usuarios** | Cajeros, empleados | Analistas, gerentes |
| **Ejemplo** | Sistema de ventas | Reporte de ventas por región y año |

---

## 2. Arquitectura de un Data Warehouse

```
[Fuentes OLTP]          [Área de Staging]        [Data Warehouse]       [Presentación]
  SQL Server    ──┐
  PostgreSQL    ──┤   ┌──────────────┐    ┌──────────────────────┐    ┌────────────┐
  MySQL         ──┼──►│  ETL / ELT   │───►│  Data Warehouse      │───►│ Power BI   │
  Excel/CSV     ──┤   │  (Pentaho,   │    │  (Esquema estrella,  │    │ Tableau    │
  APIs          ──┘   │   SSIS, etc) │    │   copo de nieve)     │    │ Excel      │
                      └──────────────┘    └──────────────────────┘    └────────────┘
                           Limpieza,              Hechos +
                           integración,           Dimensiones
                           transformación
```

### Componentes

- **Fuentes:** Sistemas OLTP, archivos planos, APIs externas.
- **Staging Area:** Zona temporal donde llegan los datos antes de transformarse. Es una copia cruda.
- **ETL:** Extract, Transform, Load — el proceso de mover y limpiar datos.
- **Data Warehouse:** El almacén analítico central con el modelo dimensional.
- **Data Mart:** Subconjunto del DW orientado a un área específica (ej: DM de Ventas).
- **Capa de presentación:** Herramientas de BI que consultan el DW.

---

## 3. Modelo Dimensional — Conceptos Base

El **modelo dimensional** organiza los datos en dos tipos de tablas:

### Tabla de Hechos (Fact Table)

Contiene las **métricas numéricas** del negocio (lo que se mide).

- Cada fila = un evento o transacción.
- Contiene **claves foráneas** a las dimensiones.
- Contiene **medidas** (cantidades, montos, conteos).

```
Ejemplo — Tabla de Hechos: FACT_VENTAS
┌─────────────────────────────────────────────────────────────┐
│ id_tiempo (FK) │ id_producto (FK) │ id_cliente (FK) │ id_tienda (FK) │
│ cantidad_vendida│ monto_total     │ descuento       │ costo          │
└─────────────────────────────────────────────────────────────┘
```

### Tabla de Dimensión (Dimension Table)

Contiene los **atributos descriptivos** que dan contexto a los hechos (cómo se analiza).

```
Ejemplo — DIM_TIEMPO         Ejemplo — DIM_PRODUCTO
┌──────────────────┐         ┌────────────────────────────┐
│ id_tiempo (PK)   │         │ id_producto (PK)            │
│ fecha            │         │ nombre_producto             │
│ dia              │         │ categoria                   │
│ mes              │         │ subcategoria                │
│ trimestre        │         │ marca                       │
│ anio             │         │ proveedor                   │
│ dia_semana       │         │ precio_unitario             │
└──────────────────┘         └────────────────────────────┘
```

### Tipos de Medidas

| Tipo | Descripción | Ejemplo |
|---|---|---|
| **Aditivas** | Se pueden sumar en todas las dimensiones | Ventas totales, cantidad |
| **Semi-aditivas** | Solo se suman en algunas dimensiones | Saldo de cuenta (no sumar en tiempo) |
| **No aditivas** | No tiene sentido sumarlas | Precio unitario, porcentaje |

---

## 4. Esquemas de Modelado

### 4.1 Esquema Estrella (Star Schema)

La tabla de hechos en el centro, rodeada de dimensiones **desnormalizadas** (planas, sin jerarquías separadas).

```
                    DIM_TIEMPO
                        │
         DIM_CLIENTE ───┼─── FACT_VENTAS ───── DIM_PRODUCTO
                        │
                    DIM_TIENDA
```

```sql
-- Estructura estrella
CREATE TABLE DIM_TIEMPO (
    id_tiempo   INT PRIMARY KEY,
    fecha       DATE,
    dia         INT,
    mes         INT,
    trimestre   INT,
    anio        INT,
    dia_semana  VARCHAR(20)
);

CREATE TABLE DIM_PRODUCTO (
    id_producto INT PRIMARY KEY,
    nombre      VARCHAR(100),
    categoria   VARCHAR(50),   -- Todo en la misma tabla (desnormalizado)
    subcategoria VARCHAR(50),
    marca       VARCHAR(50)
);

CREATE TABLE FACT_VENTAS (
    id_tiempo     INT REFERENCES DIM_TIEMPO,
    id_producto   INT REFERENCES DIM_PRODUCTO,
    id_cliente    INT REFERENCES DIM_CLIENTE,
    id_tienda     INT REFERENCES DIM_TIENDA,
    cantidad      INT,
    monto_total   DECIMAL(12,2),
    descuento     DECIMAL(5,2),
    PRIMARY KEY (id_tiempo, id_producto, id_cliente, id_tienda)
);
```

**✅ Ventajas:** Consultas simples y rápidas, fácil de entender.
**❌ Desventajas:** Redundancia de datos en dimensiones.

---

### 4.2 Esquema Copo de Nieve (Snowflake Schema)

Las dimensiones están **normalizadas**: las jerarquías se separan en tablas propias.

```
DIM_SUBCATEGORIA ──► DIM_CATEGORIA
                          │
DIM_CIUDAD ──► DIM_REGION ─┐
                            │
DIM_CLIENTE ───────────────┤
                            │
              FACT_VENTAS ──┼── DIM_PRODUCTO ──► DIM_SUBCATEGORIA
                            │
              DIM_TIENDA ───┘
```

```sql
-- Copo de nieve: categoría separada de producto
CREATE TABLE DIM_CATEGORIA (
    id_categoria INT PRIMARY KEY,
    nombre       VARCHAR(50)
);

CREATE TABLE DIM_SUBCATEGORIA (
    id_subcategoria INT PRIMARY KEY,
    nombre          VARCHAR(50),
    id_categoria    INT REFERENCES DIM_CATEGORIA  -- jerarquía separada
);

CREATE TABLE DIM_PRODUCTO (
    id_producto     INT PRIMARY KEY,
    nombre          VARCHAR(100),
    marca           VARCHAR(50),
    id_subcategoria INT REFERENCES DIM_SUBCATEGORIA  -- FK a subcategoría
);
```

**✅ Ventajas:** Menos redundancia, más fácil actualizar jerarquías.
**❌ Desventajas:** Consultas con más JOINs, más lento para análisis.

---

### 4.3 Esquema Constelación (Galaxy / Fact Constellation)

**Varias tablas de hechos** que comparten dimensiones. Es el modelo más completo y real en empresas.

```
                    DIM_TIEMPO
                   ╱           ╲
     DIM_PRODUCTO ─── FACT_VENTAS    FACT_COMPRAS ─── DIM_PROVEEDOR
                   ╲           ╱
                    DIM_TIENDA
                   (dimensión compartida)
```

```sql
-- Dos tablas de hechos compartiendo DIM_TIEMPO y DIM_PRODUCTO
CREATE TABLE FACT_VENTAS (
    id_tiempo   INT REFERENCES DIM_TIEMPO,
    id_producto INT REFERENCES DIM_PRODUCTO,
    id_cliente  INT REFERENCES DIM_CLIENTE,
    cantidad    INT,
    monto       DECIMAL(12,2)
);

CREATE TABLE FACT_COMPRAS (
    id_tiempo    INT REFERENCES DIM_TIEMPO,     -- misma dimensión
    id_producto  INT REFERENCES DIM_PRODUCTO,   -- misma dimensión
    id_proveedor INT REFERENCES DIM_PROVEEDOR,  -- dimensión exclusiva
    cantidad     INT,
    costo        DECIMAL(12,2)
);
```

**✅ Ventajas:** Análisis cruzado entre procesos del negocio.
**❌ Desventajas:** Mayor complejidad de diseño.

---

### Comparación de Esquemas

| | Estrella | Copo de Nieve | Constelación |
|---|---|---|---|
| **Dimensiones** | Desnormalizadas | Normalizadas | Mixtas |
| **Tablas de hechos** | 1 | 1 | 2 o más |
| **Velocidad de consulta** | ⚡⚡⚡ | ⚡⚡ | ⚡⚡ |
| **Redundancia** | Alta | Baja | Media |
| **Complejidad** | Baja | Media | Alta |
| **Uso recomendado** | Reportes simples | Jerarquías complejas | Múltiples procesos |

---

## 5. Dimensiones Especiales

### Dimensión Tiempo

La dimensión más importante. Siempre debe pre-poblarse con todos los días del rango.

```sql
-- Poblar DIM_TIEMPO para 5 años (script SQL Server)
DECLARE @fecha DATE = '2020-01-01';
DECLARE @fin   DATE = '2024-12-31';

WHILE @fecha <= @fin
BEGIN
    INSERT INTO DIM_TIEMPO VALUES (
        CONVERT(INT, FORMAT(@fecha, 'yyyyMMdd')),  -- id_tiempo = 20231215
        @fecha,
        DAY(@fecha),
        MONTH(@fecha),
        DATEPART(QUARTER, @fecha),
        YEAR(@fecha),
        DATENAME(WEEKDAY, @fecha),
        CASE WHEN DATENAME(WEEKDAY, @fecha) IN ('Saturday','Sunday') THEN 0 ELSE 1 END
    );
    SET @fecha = DATEADD(DAY, 1, @fecha);
END
```

### Slowly Changing Dimensions (SCD)

Las dimensiones cambian con el tiempo. ¿Cómo manejar esos cambios?

| Tipo | Comportamiento | Ejemplo |
|---|---|---|
| **SCD Tipo 1** | Sobreescribir — se pierde el histórico | Corrección de error tipográfico |
| **SCD Tipo 2** | Agregar nueva fila con fechas de vigencia — se conserva el histórico | Cliente cambia de ciudad |
| **SCD Tipo 3** | Agregar columna "valor anterior" — solo guarda el cambio previo | Cambio de nombre de categoría |

```sql
-- SCD Tipo 2: agregar nueva fila cuando cambia el cliente
ALTER TABLE DIM_CLIENTE ADD
    fecha_inicio DATE,
    fecha_fin    DATE,
    es_actual    BIT;

-- Cuando el cliente cambia de ciudad:
-- 1. Cerrar el registro actual
UPDATE DIM_CLIENTE
SET fecha_fin = GETDATE(), es_actual = 0
WHERE id_cliente = 101 AND es_actual = 1;

-- 2. Insertar nuevo registro
INSERT INTO DIM_CLIENTE VALUES (
    102, 'Juan Perez', 'Santa Cruz',  -- nueva ciudad
    GETDATE(), NULL, 1                -- fecha_inicio, fecha_fin=NULL, es_actual=1
);
```

---

## 6. Proceso ETL

**ETL = Extract, Transform, Load.** Es el corazón de un DW.

```
[Fuentes] ──Extract──► [Staging] ──Transform──► [Limpieza/Integración] ──Load──► [DW]
```

### Extract (Extracción)
- Leer datos de las fuentes: queries SQL, archivos CSV/Excel, APIs.
- Puede ser **incremental** (solo cambios desde la última carga) o **completa**.

### Transform (Transformación)
- Limpiar datos sucios (nulos, duplicados, formatos).
- Estandarizar (fechas, monedas, nombres).
- Calcular medidas derivadas.
- Aplicar reglas de negocio.

### Load (Carga)
- Insertar en el DW (INSERT o MERGE/UPSERT).
- Actualizar dimensiones (SCD).
- Actualizar tablas de hechos.

---

## 7. Metodologías de Diseño de DW

### 7.1 Metodología Inmon (Top-Down)

Propuesta por **Bill Inmon**, creador del concepto DW.

- **Enfoque:** Diseñar primero el DW corporativo completo (normalizado, 3FN) y luego construir Data Marts a partir de él.
- **Proceso:** Análisis global → Modelo ER corporativo → DW centralizado → Data Marts.
- ✅ Consistencia total de datos.
- ❌ Tarda mucho en dar resultados, requiere gran inversión inicial.

### 7.2 Metodología Kimball (Bottom-Up)

Propuesta por **Ralph Kimball**, creador del modelo dimensional.

- **Enfoque:** Construir Data Marts independientes que luego se integran en un "DW Bus".
- **Proceso:** Identificar proceso → Modelo estrella → Data Mart → Integrar con bus.
- ✅ Resultados rápidos, iterativo.
- ❌ Puede haber inconsistencias si los DM no se planifican bien.

### 7.3 Metodología HEFESTO ⭐

Metodología **latinoamericana** propuesta por **Bernabeu (2010)**, diseñada específicamente para facilitar la construcción de DW. Basada en Kimball pero con pasos más claros y documentados.

| Metodología | Enfoque | Complejidad | Resultados |
|---|---|---|---|
| Inmon | Top-Down, corporativo | Alta | Lentos |
| Kimball | Bottom-Up, Data Marts | Media | Rápidos |
| **HEFESTO** | **Bottom-Up, iterativo, simplificado** | **Baja-Media** | **Rápidos** |

---

## 8. Metodología HEFESTO — Paso a Paso Detallado

HEFESTO tiene **4 etapas principales**, cada una con pasos concretos.

```
HEFESTO
  │
  ├── ETAPA I:   Análisis de Requerimientos
  │     ├── Paso 1: Identificar preguntas del negocio
  │     └── Paso 2: Identificar indicadores y perspectivas
  │
  ├── ETAPA II:  Análisis de los OLTP
  │     ├── Paso 3: Establecer correspondencia con las fuentes
  │     └── Paso 4: Nivel de granularidad
  │
  ├── ETAPA III: Modelo Conceptual del DW
  │     ├── Paso 5: Modelo conceptual (tabla de hechos + dimensiones)
  │     └── Paso 6: Tabla de hechos y dimensiones definitivas
  │
  └── ETAPA IV:  Integración de Datos (ETL)
        ├── Paso 7: Carga inicial
        └── Paso 8: Actualización (carga incremental)
```

---

### ETAPA I — Análisis de Requerimientos

#### Paso 1: Identificar las preguntas del negocio

Entrevistar a los usuarios finales (gerentes, analistas) y formular preguntas como:

> *"¿Cuánto vendimos por producto en cada mes del año pasado?"*
> *"¿Qué región genera más ingresos por trimestre?"*
> *"¿Cuál es el producto más vendido por categoría y tienda?"*

Estas preguntas determinan **qué** construir.

#### Paso 2: Identificar Indicadores y Perspectivas

De las preguntas se extraen:

- **Indicadores (medidas):** Los valores numéricos que se quieren analizar.
- **Perspectivas (dimensiones):** Los ejes de análisis (por qué, cuándo, dónde).

```
Pregunta: "¿Cuánto vendimos por producto en cada mes?"
                    ↓
Indicadores:   cantidad vendida, monto total
Perspectivas:  Producto, Tiempo
```

**Resultado del Paso 2 — Tabla de Indicadores y Perspectivas:**

| Indicador | Perspectivas |
|---|---|
| Cantidad vendida | Producto, Tiempo, Cliente, Tienda |
| Monto total | Producto, Tiempo, Cliente, Tienda |
| Descuento aplicado | Producto, Tiempo, Cliente |

---

### ETAPA II — Análisis de los OLTP

#### Paso 3: Establecer correspondencia (Mapeo)

Identificar en qué tablas y columnas del OLTP se encuentran los indicadores y perspectivas.

```
OLTP: Base de datos transaccional ElectroAndina
┌─────────────────────────────────────────────────┐
│ Tabla VENTAS:  id_venta, fecha, id_cliente,      │
│                id_producto, cantidad, monto      │
│ Tabla PRODUCTOS: id_producto, nombre, categoria  │
│ Tabla CLIENTES:  id_cliente, nombre, ciudad      │
│ Tabla TIENDAS:   id_tienda, nombre, region       │
└─────────────────────────────────────────────────┘
         ↓ Mapeo
Indicador "cantidad vendida"  → VENTAS.cantidad
Indicador "monto total"       → VENTAS.monto
Perspectiva "Producto"        → PRODUCTOS (nombre, categoria)
Perspectiva "Tiempo"          → VENTAS.fecha (descomponer en dia, mes, anio)
Perspectiva "Cliente"         → CLIENTES (nombre, ciudad)
```

#### Paso 4: Nivel de Granularidad

Definir el **nivel de detalle más fino** que va a tener la tabla de hechos.

> ¿Una fila por transacción individual? ¿Por día? ¿Por mes?

**Regla HEFESTO:** La granularidad debe ser la más baja posible para responder todas las preguntas.

```
Granularidad elegida: Una fila por cada línea de venta (id_venta + id_producto)

Esto permite agrupar por:
  → Día, mes, trimestre, año
  → Producto, categoría
  → Cliente, ciudad
  → Tienda, región
```

---

### ETAPA III — Modelo Conceptual del DW

#### Paso 5: Modelo Conceptual

Dibujar el esquema con las tablas de hechos y dimensiones identificadas.

```
              DIM_TIEMPO
             (id_tiempo, fecha,
              dia, mes, trimestre, anio)
                    │
DIM_CLIENTE ────────┼──── FACT_VENTAS ──────── DIM_PRODUCTO
(id_cliente,        │    (id_tiempo,           (id_producto,
 nombre,            │     id_producto,          nombre,
 ciudad)            │     id_cliente,           categoria,
                    │     id_tienda,            subcategoria,
              DIM_TIENDA   cantidad,            marca)
             (id_tienda,   monto_total,
              nombre,      descuento)
              region)
```

#### Paso 6: Definición de Tablas y Atributos

Especificar exactamente qué columnas tiene cada tabla.

**FACT_VENTAS:**

| Columna | Tipo | Descripción |
|---|---|---|
| id_tiempo | INT (FK) | Referencia a DIM_TIEMPO |
| id_producto | INT (FK) | Referencia a DIM_PRODUCTO |
| id_cliente | INT (FK) | Referencia a DIM_CLIENTE |
| id_tienda | INT (FK) | Referencia a DIM_TIENDA |
| cantidad | INT | Medida aditiva |
| monto_total | DECIMAL(12,2) | Medida aditiva |
| descuento | DECIMAL(5,2) | Medida semi-aditiva |

**DIM_PRODUCTO:**

| Columna | Tipo | Descripción |
|---|---|---|
| id_producto | INT (PK) | Surrogate key (generada en el DW) |
| id_producto_origen | INT | Clave del sistema OLTP (para trazabilidad) |
| nombre | VARCHAR(100) | |
| categoria | VARCHAR(50) | |
| subcategoria | VARCHAR(50) | |
| marca | VARCHAR(50) | |

> ⚡ **Surrogate Key:** En HEFESTO y Kimball, las dimensiones usan claves propias del DW (surrogate keys), no las del OLTP. Esto desacopla el DW de cambios en el sistema origen.

---

### ETAPA IV — Integración de Datos (ETL)

#### Paso 7: Carga Inicial

Primera carga masiva al DW. Se hace una sola vez.

```sql
-- Carga inicial DIM_PRODUCTO desde OLTP
INSERT INTO DIM_PRODUCTO (id_producto_origen, nombre, categoria, subcategoria, marca)
SELECT DISTINCT
    p.id_producto,
    p.nombre,
    c.nombre_categoria,
    s.nombre_subcategoria,
    p.marca
FROM OLTP.dbo.Productos p
JOIN OLTP.dbo.Subcategorias s ON p.id_subcategoria = s.id
JOIN OLTP.dbo.Categorias c    ON s.id_categoria = c.id;

-- Carga inicial DIM_TIEMPO (generar fechas)
-- (ver script de DIM_TIEMPO en sección 5)

-- Carga inicial FACT_VENTAS
INSERT INTO FACT_VENTAS (id_tiempo, id_producto, id_cliente, id_tienda, cantidad, monto_total, descuento)
SELECT
    t.id_tiempo,
    dp.id_producto,          -- surrogate key del DW
    dc.id_cliente,
    dt.id_tienda,
    v.cantidad,
    v.monto,
    ISNULL(v.descuento, 0)
FROM OLTP.dbo.Ventas v
JOIN DIM_TIEMPO   t  ON t.fecha = CAST(v.fecha_venta AS DATE)
JOIN DIM_PRODUCTO dp ON dp.id_producto_origen = v.id_producto
JOIN DIM_CLIENTE  dc ON dc.id_cliente_origen  = v.id_cliente
JOIN DIM_TIENDA   dt ON dt.id_tienda_origen   = v.id_tienda;
```

#### Paso 8: Actualización (Carga Incremental)

Cargas periódicas (diaria, semanal) que incorporan solo los datos nuevos.

```sql
-- Solo cargar ventas del día anterior (carga incremental)
INSERT INTO FACT_VENTAS (id_tiempo, id_producto, id_cliente, id_tienda, cantidad, monto_total, descuento)
SELECT
    t.id_tiempo,
    dp.id_producto,
    dc.id_cliente,
    dt.id_tienda,
    v.cantidad,
    v.monto,
    ISNULL(v.descuento, 0)
FROM OLTP.dbo.Ventas v
JOIN DIM_TIEMPO   t  ON t.fecha = CAST(v.fecha_venta AS DATE)
JOIN DIM_PRODUCTO dp ON dp.id_producto_origen = v.id_producto
JOIN DIM_CLIENTE  dc ON dc.id_cliente_origen  = v.id_cliente
JOIN DIM_TIENDA   dt ON dt.id_tienda_origen   = v.id_tienda
WHERE CAST(v.fecha_venta AS DATE) = CAST(GETDATE()-1 AS DATE)  -- solo ayer
  AND NOT EXISTS (
      -- Evitar duplicados
      SELECT 1 FROM FACT_VENTAS f
      WHERE f.id_tiempo = t.id_tiempo
        AND f.id_producto = dp.id_producto
        AND f.id_cliente = dc.id_cliente
  );
```

---

## 9. Creación Completa del DW en SQL Server (HEFESTO aplicado)

```sql
-- ============================================================
-- PASO 1: Crear la base de datos del DW
-- ============================================================
CREATE DATABASE DW_ElectroAndina;
GO
USE DW_ElectroAndina;
GO

-- ============================================================
-- PASO 2: Crear dimensiones
-- ============================================================

CREATE TABLE DIM_TIEMPO (
    id_tiempo   INT PRIMARY KEY,
    fecha       DATE NOT NULL,
    dia         TINYINT,
    mes         TINYINT,
    nombre_mes  VARCHAR(20),
    trimestre   TINYINT,
    semestre    TINYINT,
    anio        SMALLINT,
    dia_semana  VARCHAR(20),
    es_laboral  BIT DEFAULT 1
);

CREATE TABLE DIM_PRODUCTO (
    id_producto         INT IDENTITY(1,1) PRIMARY KEY,  -- surrogate key
    id_producto_origen  INT,
    nombre              VARCHAR(100),
    categoria           VARCHAR(50),
    subcategoria        VARCHAR(50),
    marca               VARCHAR(50),
    precio_lista        DECIMAL(10,2)
);

CREATE TABLE DIM_CLIENTE (
    id_cliente         INT IDENTITY(1,1) PRIMARY KEY,
    id_cliente_origen  INT,
    nombre             VARCHAR(100),
    ciudad             VARCHAR(50),
    departamento       VARCHAR(50),
    tipo_cliente       VARCHAR(30)    -- 'Minorista', 'Mayorista', etc.
);

CREATE TABLE DIM_TIENDA (
    id_tienda         INT IDENTITY(1,1) PRIMARY KEY,
    id_tienda_origen  INT,
    nombre            VARCHAR(100),
    ciudad            VARCHAR(50),
    region            VARCHAR(50)
);

-- ============================================================
-- PASO 3: Crear tabla de hechos
-- ============================================================

CREATE TABLE FACT_VENTAS (
    id_tiempo     INT NOT NULL REFERENCES DIM_TIEMPO(id_tiempo),
    id_producto   INT NOT NULL REFERENCES DIM_PRODUCTO(id_producto),
    id_cliente    INT NOT NULL REFERENCES DIM_CLIENTE(id_cliente),
    id_tienda     INT NOT NULL REFERENCES DIM_TIENDA(id_tienda),
    cantidad      INT,
    monto_total   DECIMAL(12,2),
    costo_total   DECIMAL(12,2),
    descuento     DECIMAL(5,2),
    ganancia      AS (monto_total - costo_total),  -- columna calculada
    CONSTRAINT PK_FACT_VENTAS PRIMARY KEY (id_tiempo, id_producto, id_cliente, id_tienda)
);

-- ============================================================
-- PASO 4: Índices para optimizar consultas analíticas
-- ============================================================

CREATE INDEX IX_FACT_VENTAS_TIEMPO    ON FACT_VENTAS (id_tiempo);
CREATE INDEX IX_FACT_VENTAS_PRODUCTO  ON FACT_VENTAS (id_producto);
CREATE INDEX IX_FACT_VENTAS_CLIENTE   ON FACT_VENTAS (id_cliente);
CREATE INDEX IX_FACT_VENTAS_TIENDA    ON FACT_VENTAS (id_tienda);
```

---

## 10. Consultas Analíticas OLAP típicas

```sql
-- Ventas totales por mes y categoría
SELECT
    t.anio,
    t.nombre_mes,
    p.categoria,
    SUM(f.cantidad)    AS total_unidades,
    SUM(f.monto_total) AS total_ingresos,
    AVG(f.descuento)   AS descuento_promedio
FROM FACT_VENTAS f
JOIN DIM_TIEMPO   t ON f.id_tiempo   = t.id_tiempo
JOIN DIM_PRODUCTO p ON f.id_producto = p.id_producto
GROUP BY t.anio, t.nombre_mes, t.mes, p.categoria
ORDER BY t.anio, t.mes, total_ingresos DESC;

-- Top 5 productos más vendidos del año
SELECT TOP 5
    p.nombre,
    p.categoria,
    SUM(f.cantidad)    AS unidades_vendidas,
    SUM(f.monto_total) AS ingresos_totales
FROM FACT_VENTAS f
JOIN DIM_TIEMPO   t ON f.id_tiempo   = t.id_tiempo
JOIN DIM_PRODUCTO p ON f.id_producto = p.id_producto
WHERE t.anio = 2024
GROUP BY p.nombre, p.categoria
ORDER BY ingresos_totales DESC;

-- Ventas por región y trimestre (drill-down)
SELECT
    ti.nombre AS nombre_tienda,
    ti.region,
    t.anio,
    t.trimestre,
    SUM(f.monto_total) AS ventas
FROM FACT_VENTAS f
JOIN DIM_TIENDA ti ON f.id_tienda  = ti.id_tienda
JOIN DIM_TIEMPO t  ON f.id_tiempo  = t.id_tiempo
GROUP BY ROLLUP(ti.region, ti.nombre, t.anio, t.trimestre);
-- ROLLUP genera subtotales por región, luego gran total

-- Comparación año actual vs año anterior
SELECT
    p.categoria,
    SUM(CASE WHEN t.anio = 2024 THEN f.monto_total ELSE 0 END) AS ventas_2024,
    SUM(CASE WHEN t.anio = 2023 THEN f.monto_total ELSE 0 END) AS ventas_2023,
    SUM(CASE WHEN t.anio = 2024 THEN f.monto_total ELSE 0 END) -
    SUM(CASE WHEN t.anio = 2023 THEN f.monto_total ELSE 0 END) AS variacion
FROM FACT_VENTAS f
JOIN DIM_TIEMPO   t ON f.id_tiempo   = t.id_tiempo
JOIN DIM_PRODUCTO p ON f.id_producto = p.id_producto
WHERE t.anio IN (2023, 2024)
GROUP BY p.categoria
ORDER BY variacion DESC;
```

---

## 11. Operaciones OLAP

| Operación | Descripción | Ejemplo |
|---|---|---|
| **Roll-up** | Subir nivel de granularidad (agregar) | Días → Meses → Años |
| **Drill-down** | Bajar nivel de granularidad (detallar) | Años → Meses → Días |
| **Slice** | Filtrar una dimensión (fijar un valor) | Solo ventas de 2024 |
| **Dice** | Filtrar varias dimensiones a la vez | 2024 + Región Sur + Categoría A |
| **Pivot** | Rotar el cubo (cambiar filas/columnas) | Ver por región en filas o en columnas |

---

## 12. ETL con Python (sin Pentaho)

### 12.1 Librerías necesarias

```bash
pip install pandas sqlalchemy pyodbc psycopg2-binary openpyxl
```

### 12.2 Extracción desde SQL Server (OLTP)

```python
import pandas as pd
from sqlalchemy import create_engine

# Engine hacia el OLTP (fuente)
engine_oltp = create_engine(
    "mssql+pyodbc://sa:Password123@192.168.1.10:1433/ElectroAndina_OLTP"
    "?driver=ODBC+Driver+17+for+SQL+Server&TrustServerCertificate=yes"
)

# Engine hacia el DW (destino)
engine_dw = create_engine(
    "mssql+pyodbc://sa:Password123@192.168.1.10:1433/DW_ElectroAndina"
    "?driver=ODBC+Driver+17+for+SQL+Server&TrustServerCertificate=yes"
)

def extraer_ventas_oltp(fecha_desde: str, fecha_hasta: str) -> pd.DataFrame:
    """Extrae ventas del OLTP para un rango de fechas."""
    query = """
        SELECT
            v.id_venta,
            v.fecha_venta,
            v.id_producto,
            v.id_cliente,
            v.id_tienda,
            v.cantidad,
            v.monto,
            ISNULL(v.descuento, 0) AS descuento,
            v.costo
        FROM Ventas v
        WHERE CAST(v.fecha_venta AS DATE) BETWEEN :fecha_desde AND :fecha_hasta
    """
    with engine_oltp.connect() as conn:
        df = pd.read_sql(query, conn, params={"fecha_desde": fecha_desde, "fecha_hasta": fecha_hasta})
    print(f"[EXTRACT] {len(df)} registros extraídos del OLTP.")
    return df
```

### 12.3 Transformación

```python
def transformar_hechos(df_ventas: pd.DataFrame,
                        df_dim_tiempo: pd.DataFrame,
                        df_dim_producto: pd.DataFrame,
                        df_dim_cliente: pd.DataFrame,
                        df_dim_tienda: pd.DataFrame) -> pd.DataFrame:
    """
    Transforma los datos crudos del OLTP al formato del DW.
    Reemplaza claves del OLTP por surrogate keys del DW.
    """
    # 1. Limpiar nulos en columnas críticas
    df_ventas = df_ventas.dropna(subset=["id_producto", "id_cliente"])
    df_ventas["descuento"] = df_ventas["descuento"].fillna(0)

    # 2. Crear id_tiempo a partir de la fecha (formato YYYYMMDD)
    df_ventas["fecha_solo"] = pd.to_datetime(df_ventas["fecha_venta"]).dt.date
    df_ventas["id_tiempo"]  = pd.to_datetime(df_ventas["fecha_venta"]).dt.strftime("%Y%m%d").astype(int)

    # 3. Mapear claves OLTP → surrogate keys del DW
    mapa_producto = df_dim_producto.set_index("id_producto_origen")["id_producto"].to_dict()
    mapa_cliente  = df_dim_cliente.set_index("id_cliente_origen")["id_cliente"].to_dict()
    mapa_tienda   = df_dim_tienda.set_index("id_tienda_origen")["id_tienda"].to_dict()

    df_ventas["id_producto_dw"] = df_ventas["id_producto"].map(mapa_producto)
    df_ventas["id_cliente_dw"]  = df_ventas["id_cliente"].map(mapa_cliente)
    df_ventas["id_tienda_dw"]   = df_ventas["id_tienda"].map(mapa_tienda)

    # 4. Eliminar registros sin surrogate key (datos huérfanos)
    antes = len(df_ventas)
    df_ventas = df_ventas.dropna(subset=["id_producto_dw", "id_cliente_dw", "id_tienda_dw"])
    despues = len(df_ventas)
    if antes != despues:
        print(f"[TRANSFORM] {antes - despues} registros descartados por FK sin match.")

    # 5. Seleccionar y renombrar columnas finales
    df_fact = df_ventas[[
        "id_tiempo", "id_producto_dw", "id_cliente_dw", "id_tienda_dw",
        "cantidad", "monto", "costo", "descuento"
    ]].rename(columns={
        "id_producto_dw": "id_producto",
        "id_cliente_dw":  "id_cliente",
        "id_tienda_dw":   "id_tienda",
        "monto":          "monto_total",
        "costo":          "costo_total"
    })

    print(f"[TRANSFORM] {len(df_fact)} registros listos para cargar.")
    return df_fact
```

### 12.4 Carga al DW

```python
def cargar_hechos_dw(df_fact: pd.DataFrame):
    """Carga los hechos transformados al DW."""
    with engine_dw.connect() as conn:
        # Verificar duplicados antes de insertar
        ids_existentes = pd.read_sql(
            "SELECT id_tiempo, id_producto, id_cliente, id_tienda FROM FACT_VENTAS",
            conn
        )
        df_nuevos = df_fact.merge(
            ids_existentes,
            on=["id_tiempo", "id_producto", "id_cliente", "id_tienda"],
            how="left",
            indicator=True
        )
        df_nuevos = df_nuevos[df_nuevos["_merge"] == "left_only"].drop(columns=["_merge"])

        if len(df_nuevos) == 0:
            print("[LOAD] Sin registros nuevos que cargar.")
            return

        # Insertar en lotes de 1000 para no saturar la red
        df_nuevos.to_sql(
            name="FACT_VENTAS",
            con=conn,
            if_exists="append",
            index=False,
            chunksize=1000,
            method="multi"
        )
        conn.commit()
    print(f"[LOAD] {len(df_nuevos)} registros cargados en FACT_VENTAS.")

def cargar_dimension(df: pd.DataFrame, tabla: str, col_origen: str):
    """Carga o actualiza una dimensión en el DW (SCD Tipo 1 simple)."""
    with engine_dw.connect() as conn:
        # MERGE: insertar nuevos, ignorar existentes
        ids_existentes = pd.read_sql(
            f"SELECT {col_origen} FROM {tabla}", conn
        )[col_origen].tolist()

        df_nuevos = df[~df[col_origen].isin(ids_existentes)]

        if len(df_nuevos) > 0:
            df_nuevos.to_sql(tabla, conn, if_exists="append", index=False)
            conn.commit()
            print(f"[LOAD] {len(df_nuevos)} registros nuevos en {tabla}.")
        else:
            print(f"[LOAD] Sin registros nuevos en {tabla}.")
```

### 12.5 Pipeline ETL completo

```python
from datetime import datetime, timedelta

def ejecutar_etl(fecha_desde: str = None, fecha_hasta: str = None):
    """
    Pipeline ETL completo para carga incremental diaria.
    Por defecto carga el día anterior.
    """
    if not fecha_desde:
        ayer = (datetime.now() - timedelta(days=1)).strftime("%Y-%m-%d")
        fecha_desde = fecha_hasta = ayer

    print(f"\n{'='*50}")
    print(f"ETL iniciado: {fecha_desde} → {fecha_hasta}")
    print(f"{'='*50}")

    try:
        # ── EXTRACT ──────────────────────────────────
        df_ventas    = extraer_ventas_oltp(fecha_desde, fecha_hasta)
        df_productos = pd.read_sql("SELECT * FROM Productos", engine_oltp)
        df_clientes  = pd.read_sql("SELECT * FROM Clientes",  engine_oltp)
        df_tiendas   = pd.read_sql("SELECT * FROM Tiendas",   engine_oltp)

        # ── TRANSFORM DIMENSIONES ────────────────────
        df_dim_prod = df_productos.rename(columns={"id_producto": "id_producto_origen",
                                                    "nombre": "nombre"})
        df_dim_cli  = df_clientes.rename(columns={"id_cliente": "id_cliente_origen"})
        df_dim_ti   = df_tiendas.rename(columns={"id_tienda": "id_tienda_origen"})

        # ── LOAD DIMENSIONES ─────────────────────────
        cargar_dimension(df_dim_prod, "DIM_PRODUCTO", "id_producto_origen")
        cargar_dimension(df_dim_cli,  "DIM_CLIENTE",  "id_cliente_origen")
        cargar_dimension(df_dim_ti,   "DIM_TIENDA",   "id_tienda_origen")

        # ── Leer surrogate keys para el mapeo ────────
        with engine_dw.connect() as conn:
            df_dp = pd.read_sql("SELECT id_producto, id_producto_origen FROM DIM_PRODUCTO", conn)
            df_dc = pd.read_sql("SELECT id_cliente,  id_cliente_origen  FROM DIM_CLIENTE",  conn)
            df_dt = pd.read_sql("SELECT id_tienda,   id_tienda_origen   FROM DIM_TIENDA",   conn)

        # ── TRANSFORM HECHOS ─────────────────────────
        df_fact = transformar_hechos(df_ventas, None, df_dp, df_dc, df_dt)

        # ── LOAD HECHOS ──────────────────────────────
        cargar_hechos_dw(df_fact)

        print(f"\n✅ ETL completado exitosamente.")

    except Exception as e:
        print(f"\n❌ ETL FALLÓ: {e}")
        raise

if __name__ == "__main__":
    ejecutar_etl()  # Carga el día anterior automáticamente
```

---

## 13. Conexión desde Python al DW para Consultas

```python
import pandas as pd
from sqlalchemy import create_engine

engine_dw = create_engine(
    "mssql+pyodbc://sa:Password123@192.168.1.10:1433/DW_ElectroAndina"
    "?driver=ODBC+Driver+17+for+SQL+Server&TrustServerCertificate=yes"
)

def ventas_por_categoria_anio(anio: int) -> pd.DataFrame:
    query = """
        SELECT
            p.categoria,
            t.nombre_mes,
            t.mes,
            SUM(f.cantidad)    AS unidades,
            SUM(f.monto_total) AS ingresos
        FROM FACT_VENTAS f
        JOIN DIM_TIEMPO   t ON f.id_tiempo   = t.id_tiempo
        JOIN DIM_PRODUCTO p ON f.id_producto = p.id_producto
        WHERE t.anio = :anio
        GROUP BY p.categoria, t.nombre_mes, t.mes
        ORDER BY p.categoria, t.mes
    """
    with engine_dw.connect() as conn:
        return pd.read_sql(query, conn, params={"anio": anio})

def top_clientes(n: int = 10) -> pd.DataFrame:
    query = f"""
        SELECT TOP {n}
            c.nombre,
            c.ciudad,
            SUM(f.monto_total)  AS total_compras,
            COUNT(*)            AS num_transacciones,
            AVG(f.monto_total)  AS ticket_promedio
        FROM FACT_VENTAS f
        JOIN DIM_CLIENTE c ON f.id_cliente = c.id_cliente
        GROUP BY c.nombre, c.ciudad
        ORDER BY total_compras DESC
    """
    with engine_dw.connect() as conn:
        return pd.read_sql(query, conn)

# Exportar resultado a Excel
def exportar_a_excel(df: pd.DataFrame, nombre_archivo: str):
    df.to_excel(nombre_archivo, index=False, engine="openpyxl")
    print(f"Exportado: {nombre_archivo}")

# Uso
df = ventas_por_categoria_anio(2024)
exportar_a_excel(df, "ventas_2024.xlsx")
```

---

## 14. Conexión DW con Flask (API para Power BI / Dashboard)

```python
from flask import Flask, jsonify, request
from sqlalchemy import create_engine, text
import pandas as pd

app   = Flask(__name__)
engine = create_engine(
    "mssql+pyodbc://sa:Password123@192.168.1.10:1433/DW_ElectroAndina"
    "?driver=ODBC+Driver+17+for+SQL+Server&TrustServerCertificate=yes"
)

@app.route("/api/ventas/resumen")
def resumen_ventas():
    anio = request.args.get("anio", 2024, type=int)
    with engine.connect() as conn:
        result = conn.execute(text("""
            SELECT
                t.trimestre,
                SUM(f.monto_total) AS ingresos,
                SUM(f.cantidad)    AS unidades
            FROM FACT_VENTAS f
            JOIN DIM_TIEMPO t ON f.id_tiempo = t.id_tiempo
            WHERE t.anio = :anio
            GROUP BY t.trimestre
            ORDER BY t.trimestre
        """), {"anio": anio})
        rows = [dict(r._mapping) for r in result]
    return jsonify({"anio": anio, "data": rows})

@app.route("/api/ventas/por_region")
def ventas_por_region():
    with engine.connect() as conn:
        result = conn.execute(text("""
            SELECT
                ti.region,
                t.anio,
                SUM(f.monto_total) AS ingresos
            FROM FACT_VENTAS f
            JOIN DIM_TIENDA ti ON f.id_tienda = ti.id_tienda
            JOIN DIM_TIEMPO t  ON f.id_tiempo  = t.id_tiempo
            GROUP BY ti.region, t.anio
            ORDER BY t.anio, ingresos DESC
        """))
        rows = [dict(r._mapping) for r in result]
    return jsonify(rows)

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

---

## 15. Preguntas Frecuentes de Examen — DW

**¿Cuál es la diferencia entre esquema estrella y copo de nieve?**
> Estrella: dimensiones desnormalizadas (planas), más rápido para consultas. Copo de nieve: dimensiones normalizadas con jerarquías separadas, menos redundancia pero más JOINs.

**¿Qué es una surrogate key y por qué se usa?**
> Es una clave artificial generada en el DW, independiente del sistema OLTP. Se usa para desacoplar el DW de cambios en el sistema origen y para manejar SCD Tipo 2 (varias versiones del mismo registro).

**¿Qué problema resuelve el SCD Tipo 2?**
> Permite conservar el histórico cuando un atributo dimensional cambia. Ej: si un cliente cambia de ciudad, con SCD Tipo 2 las ventas antiguas siguen apuntando a la versión anterior del cliente (ciudad antigua), y las nuevas a la versión actualizada.

**¿Cuáles son las 4 etapas de HEFESTO?**
> 1) Análisis de requerimientos (preguntas + indicadores/perspectivas). 2) Análisis de los OLTP (mapeo + granularidad). 3) Modelo conceptual (esquema DW). 4) Integración de datos (ETL inicial e incremental).

**¿Qué es la granularidad en HEFESTO?**
> El nivel de detalle más fino de la tabla de hechos. Define qué representa cada fila. A menor granularidad, más detalle y más espacio en disco, pero más flexibilidad de análisis.

**¿Diferencia entre DW y Data Mart?**
> Un DW es el almacén central de toda la organización. Un Data Mart es un subconjunto temático (ej: solo ventas, solo finanzas) de un DW, orientado a un área o departamento específico.

**¿Qué es ROLAP, MOLAP, HOLAP?**
> ROLAP: almacena datos en BD relacional. MOLAP: almacena datos en cubos multidimensionales. HOLAP: híbrido (datos detallados en relacional, agregados en cubos).

---

*Cheatsheet generado para examen de BD III — UMSA Informática*
