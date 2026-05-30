# 📚 Cheatsheet — Gestores NoSQL + SQL (Examen BD3)

> **Énfasis:** Modelado de datos · Criterios de selección · Consideraciones para soluciones NoSQL

---

## 🗂️ Índice

1. [Criterios de Selección de BD NoSQL](#1-criterios-de-selección-de-bd-nosql)
2. [Comparativa Rápida](#2-comparativa-rápida)
3. [MongoDB](#3-mongodb-documental)
4. [Redis / KeyDB](#4-redis--keydb-clave-valor)
5. [Neo4j](#5-neo4j-grafos)
6. [Cassandra](#6-cassandra-columnar-wide-column)
7. [CouchDB](#7-couchdb-documental-http)
8. [DynamoDB](#8-dynamodb-clave-valor--documento-aws)
9. [SQL Server (referencia)](#9-sql-server-referencia-relacional)
10. [Teorema CAP y Consistencia](#10-teorema-cap-y-consistencia)
11. [Consideraciones para Plantear una Solución NoSQL](#11-consideraciones-para-plantear-una-solución-nosql)
12. [Metodología DDD (Data-Driven Design para NoSQL)](#12-metodología-ddd-data-driven-design-para-nosql)

---

## 1. Criterios de Selección de BD NoSQL

| Criterio | Preguntas clave |
|---|---|
| **Modelo de datos** | ¿Son los datos jerárquicos, relacionales, grafos, series de tiempo? |
| **Escala** | ¿Escala horizontal? ¿Millones de registros? |
| **Patrón de acceso** | ¿Lecturas frecuentes, escrituras masivas, búsquedas por clave, traversals? |
| **Consistencia requerida** | ¿Se tolera consistencia eventual? ¿Se necesita ACID? |
| **Disponibilidad** | ¿Alta disponibilidad 24/7 o puede haber downtime? |
| **Latencia** | ¿Se necesita respuesta en microsegundos (caché) o milisegundos es suficiente? |
| **Relaciones** | ¿Los datos tienen muchas relaciones complejas → Neo4j? ¿Pocas → documental? |
| **Esquema** | ¿El esquema cambia frecuentemente → NoSQL? ¿Es fijo → SQL? |
| **Infraestructura** | ¿Cloud (AWS → DynamoDB)? ¿On-premise? ¿Self-hosted? |

### Regla de oro para elegir gestor

```
Clave-Valor simple / caché        → Redis / KeyDB / DynamoDB
Documentos JSON flexibles         → MongoDB / CouchDB
Relaciones complejas / grafos     → Neo4j
Escrituras masivas / time-series  → Cassandra
Transacciones ACID complejas      → SQL Server / PostgreSQL
Análisis / reporting              → SQL Server + Data Warehouse
```

---

## 2. Comparativa Rápida

| Gestor | Tipo | Modelo | CAP | Casos de uso |
|---|---|---|---|---|
| **MongoDB** | NoSQL | Documento (BSON/JSON) | CP | Apps web, catálogos, IoT |
| **Redis** | NoSQL | Clave-Valor + estructuras | CP | Caché, sesiones, pub/sub, cola |
| **KeyDB** | NoSQL | Clave-Valor (fork Redis) | CP | Igual que Redis, mayor throughput |
| **Neo4j** | NoSQL | Grafo (nodos + aristas) | CA | Redes sociales, recomendaciones, fraude |
| **Cassandra** | NoSQL | Wide Column | AP | Series de tiempo, IoT, logs, alta escala |
| **CouchDB** | NoSQL | Documento (JSON/HTTP) | AP | Sync offline, apps móviles, REST-native |
| **DynamoDB** | NoSQL | Clave-Valor + Documento | AP | Cloud AWS, serverless, e-commerce |
| **SQL Server** | SQL | Relacional | CA | ERP, finanzas, reportes, DW |

---

## 3. MongoDB (Documental)

### Conceptos clave

| SQL | MongoDB |
|---|---|
| Base de datos | Base de datos |
| Tabla | Colección |
| Fila | Documento (BSON) |
| Columna | Campo |
| JOIN | `$lookup` / embedding |
| Primary Key | `_id` (ObjectId) |

### Modelado de datos

**Embedding (1:N donde N es pequeño y siempre se accede junto)**
```json
{
  "_id": ObjectId("..."),
  "nombre": "Laptop HP",
  "especificaciones": [
    { "clave": "RAM", "valor": "16GB" },
    { "clave": "CPU", "valor": "i7" }
  ]
}
```

**Referencia (N:M o colecciones grandes independientes)**
```json
{ "_id": ObjectId("..."), "usuario_id": ObjectId("..."), "producto_id": ObjectId("...") }
```

### Operaciones CRUD esenciales

```javascript
// INSERT
db.productos.insertOne({ nombre: "Laptop", precio: 1200 })
db.productos.insertMany([{...}, {...}])

// READ
db.productos.find({ precio: { $gt: 500 } })
db.productos.find({ categoria: "electronico" }, { nombre: 1, precio: 1 })
db.productos.findOne({ _id: ObjectId("...") })

// UPDATE
db.productos.updateOne({ _id: ObjectId("...") }, { $set: { precio: 999 } })
db.productos.updateMany({ categoria: "ropa" }, { $inc: { stock: -1 } })

// DELETE
db.productos.deleteOne({ _id: ObjectId("...") })
db.productos.deleteMany({ stock: 0 })
```

### Aggregation Pipeline

```javascript
db.ventas.aggregate([
  { $match: { estado: "completado" } },            // filtrar
  { $group: { _id: "$producto_id",
              total: { $sum: "$monto" },
              cant: { $count: {} } } },              // agrupar
  { $sort: { total: -1 } },                         // ordenar
  { $limit: 10 },                                   // limitar
  { $lookup: {                                      // JOIN
      from: "productos",
      localField: "_id",
      foreignField: "_id",
      as: "producto_info"
  }}
])
```

### Índices

```javascript
db.usuarios.createIndex({ email: 1 }, { unique: true })
db.productos.createIndex({ nombre: "text" })         // texto completo
db.sensores.createIndex({ timestamp: -1 })           // descendente
db.pedidos.createIndex({ usuario_id: 1, fecha: -1 }) // compuesto
```

### Cuándo usar MongoDB

- ✅ Esquema variable / evoluciona rápido
- ✅ Datos jerárquicos o anidados
- ✅ APIs REST / apps web modernas
- ✅ Prototipado rápido
- ❌ Transacciones ACID complejas multi-colección
- ❌ Datos altamente relacionales con muchos JOINs

---

## 4. Redis / KeyDB (Clave-Valor)

### Estructuras de datos

| Estructura | Comando clave | Uso típico |
|---|---|---|
| String | `SET` / `GET` | Caché, contadores, flags |
| Hash | `HSET` / `HGET` | Objetos, perfiles de usuario |
| List | `LPUSH` / `RPOP` | Colas, histórico, feeds |
| Set | `SADD` / `SMEMBERS` | Tags únicos, seguidores |
| Sorted Set | `ZADD` / `ZRANGE` | Rankings, leaderboards |
| Stream | `XADD` / `XREAD` | Eventos, logs, IoT |

### Comandos esenciales

```bash
# String
SET usuario:1:nombre "Juan"  EX 3600     # con TTL de 1 hora
GET usuario:1:nombre
INCR visitas:pagina:home
EXPIRE session:abc123 1800               # expira en 30 min

# Hash
HSET producto:10 nombre "Laptop" precio 1200 stock 5
HGET producto:10 precio
HGETALL producto:10
HMSET producto:10 precio 999 stock 3

# List (cola FIFO)
RPUSH cola:emails "email1@x.com"
LPOP cola:emails

# Set
SADD etiquetas:post:5 "tech" "python" "bd"
SMEMBERS etiquetas:post:5
SISMEMBER etiquetas:post:5 "tech"        # 1 si existe

# Sorted Set (ranking)
ZADD ranking:jugadores 1500 "PlayerA"
ZADD ranking:jugadores 2300 "PlayerB"
ZRANGE ranking:jugadores 0 -1 WITHSCORES # asc
ZREVRANGE ranking:jugadores 0 9          # top 10 desc

# TTL y expiración
TTL clave         # tiempo restante en seg (-1 = sin TTL)
PERSIST clave     # elimina el TTL
DEL clave
```

### Persistencia

| Modo | Descripción | Cuándo usar |
|---|---|---|
| **RDB** | Snapshot en disco periódico | Backups, no importa perder últimos datos |
| **AOF** | Log de cada operación escrita | Durabilidad máxima |
| **RDB+AOF** | Ambos combinados | Producción crítica |
| **No persistence** | Solo en memoria | Caché pura |

### KeyDB vs Redis

| Aspecto | Redis | KeyDB |
|---|---|---|
| Arquitectura | Single-threaded | Multi-threaded |
| Throughput | ~100k ops/s | ~2-3x más |
| Compatibilidad | — | 100% compatible con Redis |
| Uso típico | Caché estándar | Alta carga, IoT, eventos |

### Cuándo usar Redis/KeyDB

- ✅ Caché de consultas costosas
- ✅ Sesiones de usuario
- ✅ Rate limiting / contadores
- ✅ Pub/Sub y colas de mensajes
- ✅ Tablas de clasificación (rankings)
- ❌ Datos que superan la RAM disponible
- ❌ Consultas complejas tipo SQL

---

## 5. Neo4j (Grafos)

### Modelo de datos: Property Graph

```
(Nodo)-[:RELACIÓN {propiedad}]->(Nodo)
```

- **Nodo**: Entidad (`:Persona`, `:Producto`)
- **Arista (Relación)**: Conexión con tipo y dirección
- **Propiedad**: Atributo en nodo o relación

### Cypher — lenguaje de consulta

```cypher
-- Crear nodos
CREATE (p:Persona {nombre: "Ana", edad: 30})
CREATE (c:Ciudad {nombre: "La Paz"})

-- Crear relación
MATCH (p:Persona {nombre: "Ana"}), (c:Ciudad {nombre: "La Paz"})
CREATE (p)-[:VIVE_EN {desde: 2020}]->(c)

-- Consultar
MATCH (p:Persona)-[:VIVE_EN]->(c:Ciudad)
RETURN p.nombre, c.nombre

-- Amigos de amigos (traversal)
MATCH (yo:Persona {nombre: "Ana"})-[:AMIGO]->(amigo)-[:AMIGO]->(amigo_de_amigo)
WHERE amigo_de_amigo <> yo
RETURN amigo_de_amigo.nombre

-- Camino más corto
MATCH path = shortestPath((a:Persona {nombre:"Ana"})-[*]-(b:Persona {nombre:"Luis"}))
RETURN path

-- Filtrar y agregar
MATCH (p:Persona)-[:COMPRÓ]->(prod:Producto)
WHERE prod.precio > 100
RETURN p.nombre, COUNT(prod) AS total_compras
ORDER BY total_compras DESC
LIMIT 5

-- UPDATE
MATCH (p:Persona {nombre: "Ana"})
SET p.edad = 31

-- DELETE
MATCH (p:Persona {nombre: "Ana"})
DETACH DELETE p    -- elimina nodo y todas sus relaciones
```

### Cuándo usar Neo4j

- ✅ Redes sociales (amigos, seguidores)
- ✅ Sistemas de recomendación
- ✅ Detección de fraude
- ✅ Grafos de conocimiento
- ✅ Rutas y logística
- ❌ Datos tabulares simples
- ❌ Escrituras masivas de series de tiempo

---

## 6. Cassandra (Wide Column)

### Modelo de datos

- **Keyspace** ≈ Base de datos (con factor de replicación)
- **Table** ≈ Tabla, pero el esquema es orientado a consultas
- **Partition Key**: Define en qué nodo viven los datos
- **Clustering Key**: Ordena datos dentro de la partición
- **No JOINs**, **no subqueries**, **no GROUP BY** (en versiones básicas)

### CQL — Cassandra Query Language

```sql
-- Crear keyspace
CREATE KEYSPACE tienda
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

USE tienda;

-- Crear tabla (diseñada para la consulta)
CREATE TABLE pedidos_por_usuario (
  usuario_id  UUID,
  fecha       TIMESTAMP,
  pedido_id   UUID,
  total       DECIMAL,
  estado      TEXT,
  PRIMARY KEY ((usuario_id), fecha, pedido_id)   -- PK compuesta
) WITH CLUSTERING ORDER BY (fecha DESC);

-- INSERT
INSERT INTO pedidos_por_usuario (usuario_id, fecha, pedido_id, total, estado)
VALUES (uuid(), toTimestamp(now()), uuid(), 250.00, 'completado');

-- SELECT (siempre por partition key)
SELECT * FROM pedidos_por_usuario WHERE usuario_id = ?;
SELECT * FROM pedidos_por_usuario WHERE usuario_id = ? AND fecha > '2024-01-01';

-- UPDATE
UPDATE pedidos_por_usuario SET estado = 'enviado'
WHERE usuario_id = ? AND fecha = ? AND pedido_id = ?;

-- DELETE
DELETE FROM pedidos_por_usuario
WHERE usuario_id = ? AND fecha = ? AND pedido_id = ?;
```

### Reglas de modelado en Cassandra

| Regla | Descripción |
|---|---|
| **Query-first** | Diseñar la tabla según la consulta, no según la entidad |
| **Una tabla por consulta** | Duplicar datos es aceptable y esperado |
| **Partition key** | Distribuye datos; elegir campo de alta cardinalidad |
| **Evitar particiones calientes** | No usar un campo con pocos valores únicos como partition key |
| **Limit de partición** | Máximo ~100 MB por partición recomendado |

### Cuándo usar Cassandra

- ✅ Series de tiempo (logs, métricas, IoT)
- ✅ Escrituras masivas y continuas
- ✅ Alta disponibilidad multi-datacenter
- ✅ Datos de sensores / telemetría
- ❌ Transacciones ACID
- ❌ Consultas ad-hoc complejas
- ❌ Relaciones entre entidades

---

## 7. CouchDB (Documental HTTP)

### Características distintivas

- API 100% HTTP/REST (cada operación es una petición HTTP)
- Documentos JSON con campo `_id` y `_rev` (revisión)
- Sincronización bidireccional nativa (ideal para apps offline)
- Vistas definidas con MapReduce (JavaScript)
- Consistencia eventual, disponibilidad alta (AP)

### API REST básica

```bash
# Crear base de datos
PUT http://localhost:5984/tienda

# Insertar documento
POST http://localhost:5984/tienda
Content-Type: application/json
{ "tipo": "producto", "nombre": "Laptop", "precio": 1200 }

# Leer documento
GET http://localhost:5984/tienda/{id}

# Actualizar (REQUIERE _rev)
PUT http://localhost:5984/tienda/{id}
{ "_rev": "1-abc123", "precio": 999 }

# Eliminar (REQUIERE _rev)
DELETE http://localhost:5984/tienda/{id}?rev=1-abc123

# Consultar todos los documentos
GET http://localhost:5984/tienda/_all_docs?include_docs=true

# Mango queries (sin MapReduce)
POST http://localhost:5984/tienda/_find
{ "selector": { "precio": { "$gt": 500 } }, "fields": ["nombre", "precio"] }
```

### Cuándo usar CouchDB

- ✅ Apps móviles con sincronización offline
- ✅ Integración con servicios REST/HTTP puros
- ✅ Replicación entre nodos simple
- ❌ Consultas analíticas complejas
- ❌ Alto volumen de escrituras concurrentes

---

## 8. DynamoDB (Clave-Valor + Documento, AWS)

### Conceptos clave

| Concepto | Descripción |
|---|---|
| **Table** | Colección de ítems |
| **Item** | Registro (equivale a documento/fila) |
| **Attribute** | Campo del ítem |
| **Partition Key** | Clave primaria simple (hash) |
| **Sort Key** | Clave de ordenamiento (opcional, forma PK compuesta) |
| **GSI** | Global Secondary Index — consulta por otro atributo |
| **LSI** | Local Secondary Index — mismo partition key, otra sort key |

### Modelo de acceso (Single-Table Design)

```
PK                   SK                    Attributes
--------------------------------------------------------------
USER#123             PROFILE              nombre, email
USER#123             ORDER#2024-001       total, estado
USER#123             ORDER#2024-002       total, estado
PRODUCT#ABC          INFO                 nombre, precio
```

### Operaciones (SDK / Python boto3)

```python
import boto3
dynamodb = boto3.resource('dynamodb')
tabla = dynamodb.Table('Tienda')

# PUT (crear o reemplazar)
tabla.put_item(Item={ 'PK': 'USER#123', 'SK': 'PROFILE', 'nombre': 'Ana' })

# GET
resp = tabla.get_item(Key={ 'PK': 'USER#123', 'SK': 'PROFILE' })
item = resp['Item']

# QUERY (por PK, eficiente)
resp = tabla.query(
    KeyConditionExpression=Key('PK').eq('USER#123') & Key('SK').begins_with('ORDER#')
)

# SCAN (recorre toda la tabla, costoso)
resp = tabla.scan(FilterExpression=Attr('estado').eq('pendiente'))

# UPDATE
tabla.update_item(
    Key={ 'PK': 'USER#123', 'SK': 'PROFILE' },
    UpdateExpression='SET nombre = :n',
    ExpressionAttributeValues={ ':n': 'Ana López' }
)

# DELETE
tabla.delete_item(Key={ 'PK': 'USER#123', 'SK': 'PROFILE' })
```

### Cuándo usar DynamoDB

- ✅ Aplicaciones serverless en AWS
- ✅ Escala automática sin administración
- ✅ Alta disponibilidad gestionada
- ✅ E-commerce, gaming, IoT en cloud
- ❌ Consultas analíticas complejas (usar Redshift/Athena)
- ❌ Fuera de ecosistema AWS (lock-in)

---

## 9. SQL Server (Referencia Relacional)

### DDL esencial

```sql
-- Crear tabla
CREATE TABLE Productos (
    id          INT IDENTITY(1,1) PRIMARY KEY,
    nombre      NVARCHAR(200)     NOT NULL,
    precio      DECIMAL(10,2)     NOT NULL,
    categoria   NVARCHAR(50),
    created_at  DATETIME2         DEFAULT GETDATE()
);

-- Índice
CREATE INDEX idx_categoria ON Productos(categoria);
CREATE UNIQUE INDEX idx_email ON Usuarios(email);

-- Foreign Key
ALTER TABLE Pedidos
ADD CONSTRAINT fk_usuario FOREIGN KEY (usuario_id) REFERENCES Usuarios(id);
```

### DML esencial

```sql
-- SELECT con JOIN
SELECT p.nombre, u.nombre AS comprador, ped.fecha
FROM Pedidos ped
INNER JOIN Usuarios u ON ped.usuario_id = u.id
INNER JOIN Productos p ON ped.producto_id = p.id
WHERE ped.estado = 'completado'
ORDER BY ped.fecha DESC;

-- Agregaciones
SELECT categoria, COUNT(*) AS cantidad, AVG(precio) AS precio_promedio
FROM Productos
GROUP BY categoria
HAVING COUNT(*) > 5;

-- INSERT, UPDATE, DELETE
INSERT INTO Productos (nombre, precio, categoria) VALUES ('Laptop', 1200.00, 'Electrónico');
UPDATE Productos SET precio = 999.00 WHERE id = 1;
DELETE FROM Productos WHERE stock = 0;

-- Transacción
BEGIN TRANSACTION;
  UPDATE Cuentas SET saldo = saldo - 100 WHERE id = 1;
  UPDATE Cuentas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;
-- (o ROLLBACK si hay error)
```

### Linked Servers (SQL Server distribuido)

```sql
-- Consultar servidor remoto
SELECT * FROM [SERVIDOR_COCHABAMBA].BaseDatos.dbo.Tabla;

-- INSERT desde remoto
INSERT INTO Tabla_Local
SELECT * FROM [SERVIDOR_REMOTO].BD_Remota.dbo.Tabla_Remota
WHERE condicion = 'valor';
```

---

## 10. Teorema CAP y Consistencia

```
        C (Consistency)
       /
      /
     *  ← Solo 2 de 3
    / \
   /   \
  A --- P
(Availability) (Partition Tolerance)
```

| Combinación | Gestores | Qué sacrifica |
|---|---|---|
| **CP** | MongoDB, Redis, Neo4j | Disponibilidad ante partición |
| **AP** | Cassandra, CouchDB, DynamoDB | Consistencia fuerte (eventual) |
| **CA** | SQL Server, PostgreSQL | Tolerancia a partición (red confiable) |

### Niveles de consistencia (Cassandra)

| Nivel | Descripción |
|---|---|
| `ONE` | Responde con 1 nodo (rápido, menos consistente) |
| `QUORUM` | Mayoría de nodos (balance) |
| `ALL` | Todos los nodos (más consistente, más lento) |
| `LOCAL_QUORUM` | Quorum dentro del datacenter local |

---

## 11. Consideraciones para Plantear una Solución NoSQL

### Checklist de análisis

```
1. DOMINIO DEL PROBLEMA
   □ ¿Cuál es el volumen de datos esperado?
   □ ¿Cuál es el patrón de acceso principal (lectura vs escritura)?
   □ ¿Los datos tienen estructura fija o variable?
   □ ¿Hay relaciones complejas entre entidades?

2. REQUERIMIENTOS NO FUNCIONALES
   □ ¿Qué nivel de disponibilidad se requiere? (99.9%? 99.999%?)
   □ ¿Se tolera consistencia eventual?
   □ ¿Cuál es la latencia máxima aceptable?
   □ ¿Se necesita escala horizontal?

3. OPERACIONES CLAVE
   □ Identificar las 3-5 consultas más frecuentes
   □ Diseñar el modelo de datos orientado a esas consultas
   □ Evaluar si se necesitan índices secundarios

4. ESTRATEGIA DE MODELADO
   □ ¿Embedding o referencias? (MongoDB)
   □ ¿Una tabla por consulta? (Cassandra)
   □ ¿Single-table design? (DynamoDB)
   □ ¿Nodos y relaciones? (Neo4j)

5. TRADE-OFFS A JUSTIFICAR
   □ Duplicación de datos vs. performance
   □ Consistencia vs. disponibilidad (CAP)
   □ Flexibilidad de esquema vs. integridad
   □ Complejidad operacional vs. características
```

### Patrón de justificación para el examen

> **"Se elige [Gestor] porque el problema requiere [característica principal], los datos son [descripción], el patrón de acceso es [lectura/escritura/grafo/etc.], y se tolera [consistencia eventual / no se tolera downtime / etc.]"**

**Ejemplo:**
> "Se elige **Cassandra** para el sistema de logs de sensores IoT porque el volumen de escrituras es masivo (millones/hora), los datos son series de tiempo por dispositivo, el patrón de acceso es write-heavy con consultas por dispositivo+rango de fecha, y se tolera consistencia eventual (AP). El modelo query-first define la partition key como `device_id` y la clustering key como `timestamp DESC`."

---

## 12. Metodología DDD (Data-Driven Design para NoSQL)

### Fases principales

```
1. IDENTIFICAR ENTIDADES Y CONTEXTOS
   └─ Definir bounded contexts
   └─ Identificar aggregates (raíz de agregado)

2. DEFINIR PATRONES DE ACCESO
   └─ Listar todas las consultas necesarias
   └─ Estimar frecuencia y volumen

3. MODELAR SEGÚN ACCESO (no según entidad)
   └─ Query-first design
   └─ Embedding vs. referencia
   └─ Desnormalización controlada

4. ELEGIR GESTOR
   └─ Aplicar criterios de selección
   └─ Validar con CAP theorem

5. DEFINIR ÍNDICES Y CLAVES
   └─ Partition keys
   └─ Índices secundarios
   └─ Estrategia de TTL si aplica

6. VALIDAR
   └─ Probar consultas principales
   └─ Evaluar performance en escala
```

### Modelos de datos por tipo de relación

| Relación | MongoDB | Cassandra | Neo4j |
|---|---|---|---|
| 1:1 | Embed en mismo doc | Atributos en misma fila | 1 nodo con propiedades |
| 1:N (N pequeño) | Array embebido | Clustering key | Relaciones directas |
| 1:N (N grande) | Referencia con `_id` | Tabla separada por query | Relaciones directas |
| N:M | Colección intermedia o arrays de refs | Tabla de join desnormalizada | Relación con propiedades |

---

## 📝 Resumen ultra-rápido para el examen

| Necesito... | Uso... |
|---|---|
| Cachear resultados de BD | Redis |
| Guardar documentos JSON flexibles | MongoDB |
| Modelar una red social | Neo4j |
| Ingestar millones de eventos/hora | Cassandra |
| API REST que sincroniza offline | CouchDB |
| Serverless escalable en AWS | DynamoDB |
| Transacciones ACID, reportes, DW | SQL Server |

---

*Cheatsheet generado para examen BD3 — INF-262 UMSA*
