# 📚 Cheatsheet — Metodologías DW / NoSQL / BigData

> **Examen:** DataWarehouse · BD NoSQL · BigData | Énfasis: modelado, criterios de selección, planteamiento de soluciones.

---

## Índice
1. [Data Warehouse](#1-data-warehouse)
2. [Data Mart](#2-data-mart)
3. [Metodología Hefesto](#3-metodología-hefesto)
4. [Metodología DDD](#4-metodología-ddd-domain-driven-design)
5. [Arquitectura BigData](#5-arquitectura-bigdata)
6. [Criterios de Selección NoSQL](#6-criterios-de-selección-nosql)

---

## 1. Data Warehouse

> **Definición:** Repositorio central integrado de datos históricos orientado al análisis (OLAP), no a transacciones (OLTP).

### Pasos para construirlo

| # | Paso | Detalle |
|---|------|---------|
| 01 | **Definir requerimientos** | ¿Qué preguntas de negocio debe responder? ¿Qué KPIs? Entrevistas con usuarios. |
| 02 | **Identificar fuentes (OLTP)** | ERP, CRM, archivos planos, APIs. Analizar calidad y estructura de datos. |
| 03 | **Diseñar modelo dimensional** | Elegir esquema **Estrella** (Star) o **Copo de Nieve** (Snowflake). Definir hechos y dimensiones. |
| 04 | **Proceso ETL** | **E**xtracción (fuentes OLTP) → **T**ransformación (limpieza, integración) → **L**oad (carga al DW). |
| 05 | **Poblar y validar** | Carga inicial histórica. Validar integridad con conteos y checksums. |
| 06 | **Crear Data Marts / cubos OLAP** | Subconjuntos por área (Ventas, RRHH). Cubos para análisis multidimensional. |
| 07 | **Publicar y mantener** | Conectar herramienta BI (Power BI, Pentaho). Programar cargas incrementales. |

### Componentes clave
- **Tabla de Hechos:** métricas cuantificables (ventas, cantidades, montos)
- **Tabla de Dimensiones:** contexto descriptivo (Tiempo, Producto, Cliente, Región)
- **Surrogate Key (SK):** clave artificial en dimensiones, independiente del OLTP
- **Esquema Estrella:** dimensiones directamente relacionadas a la tabla de hechos
- **Esquema Snowflake:** dimensiones normalizadas (sub-dimensiones)

---

## 2. Data Mart

> **Definición:** Subconjunto temático del DW orientado a un área de negocio específica. Más pequeño y enfocado.

### Diferencia clave
| | **Data Warehouse** | **Data Mart** |
|---|---|---|
| Alcance | Toda la empresa | Un departamento / área |
| Tamaño | Grande | Pequeño |
| Audiencia | Toda la organización | Equipo específico (Ventas, RRHH) |
| Fuente | Múltiples fuentes | DW central o fuentes específicas |

### Pasos para construirlo

| # | Paso | Detalle |
|---|------|---------|
| 01 | **Definir el área de negocio** | Ej: Ventas, Finanzas, Marketing. Delimitar alcance. |
| 02 | **Elegir enfoque** | **Dependiente:** se alimenta del DW central. **Independiente:** tiene su propio ETL desde OLTP. |
| 03 | **Modelar hechos y dimensiones** | Tabla de hechos (métricas) + tablas de dimensiones (contexto). |
| 04 | **ETL específico** | Extraer desde DW (si dependiente) o OLTP. Transformar al modelo dimensional. |
| 05 | **Construir cubos OLAP** | Agregaciones pre-calculadas: drill-down, roll-up, slice & dice. |
| 06 | **Conectar a herramienta BI** | Power BI, Pentaho, Tableau. Definir roles y permisos por área. |

---

## 3. Metodología Hefesto

> **Definición:** Metodología latinoamericana para construir DW, orientada a requerimientos del usuario. **Iterativa**, parte de preguntas del negocio.

### Flujo general
```
Preguntas del negocio → Modelo conceptual → Análisis OLTP → Modelo lógico → ETL → BI → (Iterar)
```

---

### ETAPA 1 — Análisis de Requerimientos

| # | Actividad | Descripción |
|---|-----------|-------------|
| 1.1 | **Identificar preguntas del negocio** | Entrevistas con usuarios. ¿Qué necesitan analizar? Ej: "¿Cuánto vendimos por región este mes?" |
| 1.2 | **Identificar indicadores** | Métricas numéricas que se quieren medir. Ej: *ventas totales*, *cantidad de productos*. |
| 1.3 | **Identificar perspectivas** | Dimensiones de análisis. Ej: *Tiempo*, *Producto*, *Región*, *Cliente*. |
| 1.4 | **Modelo conceptual** | Diagrama de estrella conceptual: indicadores en el centro + perspectivas alrededor. |

---

### ETAPA 2 — Análisis de los OLTP

| # | Actividad | Descripción |
|---|-----------|-------------|
| 2.1 | **Explorar fuentes OLTP** | Mapear tablas de la BD transaccional. ¿Dónde están los indicadores y perspectivas? |
| 2.2 | **Conformar indicadores** | ¿Se calculan directamente o requieren transformación? Ej: suma, promedio, conteo. |
| 2.3 | **Establecer nivel de granularidad** | ¿Al detalle de qué se almacena? Ej: por día, por factura, por sucursal. |
| 2.4 | **Establecer correspondencias** | Mapear cada perspectiva e indicador a sus tablas/columnas en el OLTP. |

---

### ETAPA 3 — Modelo Lógico del DW

| # | Actividad | Descripción |
|---|-----------|-------------|
| 3.1 | **Diseñar tabla de hechos** | Columnas = indicadores (métricas) + FKs a dimensiones. Granularidad definida. |
| 3.2 | **Diseñar tablas de dimensiones** | Atributos descriptivos de cada perspectiva. Incluir Surrogate Key (SK) artificial. |
| 3.3 | **Completar diagrama estrella/snowflake** | Verificar integridad referencial y cardinalidad. |
| 3.4 | **Definir SCD si aplica** | *Slowly Changing Dimensions*: Type 1 (sobreescribir), Type 2 (historizar), Type 3 (columna adicional). |

---

### ETAPA 4 — Integración ETL

| # | Actividad | Descripción |
|---|-----------|-------------|
| 4.1 | **Diseñar el proceso ETL** | Mapeo OLTP → DW. Transformaciones, limpieza de nulos, homogenización de formatos. |
| 4.2 | **Carga inicial** | Poblar dimensiones primero (con SK), luego tabla de hechos. |
| 4.3 | **Carga incremental** | Programar actualizaciones periódicas. Detectar cambios: timestamps, CDC, flags. |
| 4.4 | **Validar y publicar** | Conectar BI. Verificar que reportes respondan las preguntas del paso 1.1. |

> ⚡ **Hefesto es iterativo:** al finalizar la etapa 4, se vuelve a la etapa 1 para nuevos requerimientos.

---

## 4. Metodología DDD (Domain-Driven Design)

> **Definición:** Enfoque de diseño de software centrado en el **dominio del negocio**. En NoSQL es clave porque define el modelo de datos según **cómo se accede**, no cómo se almacena relacionalmente.

### Conceptos fundamentales

| Concepto | Definición | Ejemplo |
|----------|-----------|---------|
| **Dominio** | Área de negocio que se modela | "Ventas Online" |
| **Entidad (Entity)** | Objeto con identidad única y ciclo de vida | `Cliente`, `Pedido` |
| **Objeto de Valor (Value Object)** | Sin identidad propia, se define por sus atributos | `Dirección`, `Precio` |
| **Agregado (Aggregate)** | Grupo de entidades/value objects con una raíz que garantiza consistencia | `Pedido` (raíz) + `ÍtemPedido` |
| **Raíz de Agregado (Aggregate Root)** | Única entrada al agregado. Solo se accede por ella | `Pedido.getId()` |
| **Repositorio** | Abstracción de acceso a datos por agregado | `PedidoRepository.findById()` |
| **Bounded Context** | Límite explícito donde un modelo domina | Contexto "Envíos" vs "Facturación" |
| **Ubiquitous Language** | Vocabulario común entre devs y negocio dentro del bounded context | Mismo término en código y conversación |

---

### Pasos para aplicar DDD en NoSQL

| # | Paso | Detalle |
|---|------|---------|
| 01 | **Entender el dominio** | Entrevistas con expertos del negocio. Construir el Ubiquitous Language. |
| 02 | **Definir Bounded Contexts** | Separar áreas con límites claros. Cada contexto puede tener su propia BD NoSQL. |
| 03 | **Identificar Agregados y raíces** | Definir qué entidades van juntas. La raíz = unidad de consistencia transaccional. |
| 04 | **Modelar según patrones de acceso** | En NoSQL: diseñar el modelo según las **QUERIES**, no la normalización. Desnormalizar si es necesario. |
| 05 | **Elegir BD NoSQL según el agregado** | Documentos complejos → MongoDB. Grafos → Neo4j. Clave-valor → Redis. Alta escala → Cassandra. |
| 06 | **Definir repositorios por agregado** | Interfaces: `findById`, `save`, `delete`. El repositorio oculta la BD subyacente. |
| 07 | **Context Mapping** | Si hay múltiples contextos: definir comunicación con Shared Kernel, Anti-Corruption Layer o Published Language. |

### Regla de oro DDD en NoSQL
> ⚡ El **Agregado** define el documento. Todo lo que se consulta junto → se guarda junto en el mismo documento. Evitar JOINs en NoSQL desnormalizando dentro del agregado.

---

## 5. Arquitectura BigData

> **Definición:** Diseño de sistemas para procesar grandes volúmenes, variedad y velocidad de datos. Implica decisiones de ingesta, almacenamiento, procesamiento y consumo.

### Las 5 V's del BigData

| V | Concepto | Descripción |
|---|----------|-------------|
| **Volumen** | Cantidad | Terabytes, Petabytes. Requiere almacenamiento distribuido (HDFS, S3). |
| **Velocidad** | Rapidez | Datos en tiempo real o near-real-time. Kafka, Spark Streaming. |
| **Variedad** | Tipos | Estructurados, semi-estructurados (JSON, XML), no estructurados (imágenes, texto). |
| **Veracidad** | Calidad | Incertidumbre y ruido en los datos. Requiere validación y limpieza. |
| **Valor** | Propósito | Extraer conocimiento accionable del dato. Justifica el sistema. |

---

### Arquitectura Lambda (3 capas)

```
           ┌─────────────────────────────────────────────────────────┐
Datos ─────┤  BATCH LAYER       (histórico completo, alta precisión) │
entrantes  │  SPEED LAYER       (datos recientes, baja latencia)     │
           │  SERVING LAYER     (combina Batch + Speed, responde)    │
           └─────────────────────────────────────────────────────────┘
```

| Capa | Función | Herramientas |
|------|---------|--------------|
| **Batch Layer** | Procesa TODOS los datos históricos. Alta latencia, alta precisión. | Hadoop MapReduce, Spark Batch |
| **Speed Layer** | Procesa datos en tiempo real. Baja latencia, datos recientes. | Spark Streaming, Flink, Storm |
| **Serving Layer** | Combina resultados de Batch + Speed. Responde queries del usuario. | HBase, Cassandra, ElasticSearch |

> ⚠️ **Desventaja Lambda:** mantener 2 pipelines (batch + speed) aumenta la complejidad operativa.

---

### Arquitectura Kappa (simplificada)

| Concepto | Descripción |
|----------|-------------|
| **Un solo pipeline** | Todo pasa por la capa de streaming. No hay batch separado. |
| **Log inmutable** | Kafka como fuente de verdad. Permite re-procesar desde el inicio. |
| **Serving Layer** | Resultados del stream vuelcan a BD de consulta. Más simple de operar. |

> ✅ **Ventaja Kappa:** un solo sistema. **Límite:** no siempre puede reemplazar el batch para análisis históricos complejos.

---

### Pasos para plantear una solución BigData

| # | Capa | Descripción |
|---|------|-------------|
| 01 | **Ingesta** | Definir fuentes y conectores: Kafka (streaming), Sqoop (RDBMS), Flume (logs), APIs REST. |
| 02 | **Almacenamiento crudo (Data Lake)** | HDFS, S3, Azure Blob. Guardar datos sin transformar en formato original. |
| 03 | **Procesamiento** | Batch (Spark) o Streaming (Flink). Limpiar, transformar, enriquecer datos. |
| 04 | **Almacenamiento analítico** | Data Warehouse (Hive, BigQuery) o NoSQL según tipo de query y acceso. |
| 05 | **Consumo y visualización** | BI tools, APIs REST, ML pipelines, dashboards (Power BI, Superset). |

---

## 6. Criterios de Selección NoSQL

> Regla DDD → el tipo de BD NoSQL se elige por el **Agregado** y su **patrón de acceso**, NO por el volumen ni la popularidad.

| Tipo | Modelo | Úsalo cuando... | Fortaleza | Evitar si... |
|------|--------|-----------------|-----------|--------------|
| **Documental** MongoDB, CouchDB | Documentos JSON/BSON anidados | Datos semi-estructurados, esquema variable, jerarquías complejas (catálogos, perfiles, CMS) | Consultas ricas, índices secundarios, flexibilidad de esquema | Relaciones complejas entre entidades o transacciones multi-documento críticas |
| **Clave-Valor** Redis, DynamoDB | Hash table distribuida | Caché, sesiones, contadores, colas, configuraciones. Acceso por ID exacto, ultra-rápido | Latencia sub-milisegundo, escalado horizontal simple | Necesitas consultar por campos distintos a la clave |
| **Grafo** Neo4j | Nodos + Relaciones + Propiedades | Redes sociales, recomendaciones, detección de fraude, rutas, datos altamente relacionados | Traversal de relaciones profundas sin JOINs costosos | Datos no relacionales o consultas simples por clave |
| **Columnar** Cassandra, HBase | Familias de columnas | Series temporales, IoT, logs, escritura masiva y continua, alta disponibilidad geográfica | Escritura extremadamente rápida, escalado lineal, tolerancia a fallos | Consultas ad-hoc complejas o esquema muy variable por fila |

### Consideraciones para plantear una solución NoSQL

1. **¿Cuál es el patrón de acceso?** → Define el modelo (no al revés)
2. **¿Qué es el Agregado?** → Lo que se consulta junto, se guarda junto
3. **¿Se necesita consistencia fuerte o eventual?** → SQL o NoSQL AP/CP (teorema CAP)
4. **¿El esquema es fijo o variable?** → Variable: documental. Fijo y masivo: columnar
5. **¿Hay relaciones complejas?** → Grafo. ¿Acceso por clave única? → Clave-Valor
6. **¿Alta disponibilidad geográfica?** → Cassandra. ¿Consultas ad-hoc complejas? → MongoDB o SQL

---

*Cheatsheet generado para examen de DataWarehouse · BD NoSQL · BigData*