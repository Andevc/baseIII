# 🖥️ Cheatsheet — Activar e Insertar datos desde Terminal (NoSQL)

> Cubre: MongoDB · Redis/KeyDB · Neo4j · Cassandra · CouchDB · DynamoDB (local)

---

## 🗂️ Índice

1. [MongoDB](#1-mongodb)
2. [Redis / KeyDB](#2-redis--keydb)
3. [Neo4j](#3-neo4j)
4. [Cassandra](#4-cassandra)
5. [CouchDB](#5-couchdb)
6. [DynamoDB Local](#6-dynamodb-local)
7. [Referencia rápida de puertos](#7-referencia-rápida-de-puertos)

---

## 1. MongoDB

### Iniciar el servicio

```bash
# Windows — como servicio
net start MongoDB

# Windows — manual (si no está como servicio)
mongod --dbpath "C:\data\db"

# Linux / WSL
sudo systemctl start mongod
sudo systemctl status mongod
sudo systemctl stop mongod

# Docker
docker run -d --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
  mongo:latest

# Verificar que corre
curl http://localhost:27017
# o simplemente conectar con mongosh
```

### Conectar a la terminal (mongosh)

```bash
# Conexión básica (sin auth)
mongosh

# Con usuario y contraseña
mongosh -u admin -p admin123 --authenticationDatabase admin

# A una BD específica
mongosh "mongodb://localhost:27017/tienda"

# Con URI completa
mongosh "mongodb://admin:admin123@localhost:27017/tienda?authSource=admin"
```

### Comandos de navegación

```javascript
// Ver bases de datos
show dbs

// Usar / crear una base de datos
use tienda

// Ver colecciones
show collections

// Ver en qué BD estás
db

// Eliminar la BD actual
db.dropDatabase()
```

### Insertar datos

```javascript
// Insertar un documento
db.productos.insertOne({
    nombre: "Laptop HP",
    precio: 1200.00,
    categoria: "Electronico",
    stock: 15,
    specs: { ram: "16GB", cpu: "i7" },
    tags: ["portatil", "oficina"]
})

// Insertar varios documentos
db.productos.insertMany([
    { nombre: "Mouse Logitech", precio: 25.00, categoria: "Periferico", stock: 50 },
    { nombre: "Teclado Mecánico", precio: 80.00, categoria: "Periferico", stock: 30 },
    { nombre: "Monitor 27\"", precio: 350.00, categoria: "Electronico", stock: 10 }
])

// Insertar con _id personalizado
db.clientes.insertOne({
    _id: "CLI-001",
    nombre: "Ana López",
    email: "ana@mail.com",
    ciudad: "La Paz"
})
```

### Consultas básicas de verificación

```javascript
// Ver todos
db.productos.find()

// Ver con formato legible
db.productos.find().pretty()

// Contar documentos
db.productos.countDocuments()

// Ver uno
db.productos.findOne({ nombre: "Laptop HP" })

// Solo algunos campos
db.productos.find({}, { nombre: 1, precio: 1, _id: 0 })
```

### Importar datos desde archivo JSON

```bash
# Desde terminal (fuera de mongosh)
mongoimport --db tienda \
            --collection productos \
            --file productos.json \
            --jsonArray

# Con autenticación
mongoimport --uri "mongodb://admin:admin123@localhost:27017/tienda" \
            --collection productos \
            --file productos.json \
            --jsonArray
```

---

## 2. Redis / KeyDB

### Iniciar el servicio

```bash
# Windows — Redis (descargado como .zip o MSI)
redis-server

# Windows — con archivo de configuración
redis-server redis.conf

# Linux / WSL
sudo systemctl start redis
sudo systemctl status redis
sudo systemctl stop redis

# KeyDB en Linux
sudo systemctl start keydb
sudo systemctl status keydb

# Docker — Redis
docker run -d --name redis \
  -p 6379:6379 \
  redis:latest

# Docker — KeyDB
docker run -d --name keydb \
  -p 6379:6379 \
  eqalpha/keydb

# Verificar que corre
redis-cli ping
# Respuesta esperada: PONG
```

### Conectar a la terminal (redis-cli)

```bash
# Conexión básica
redis-cli

# Puerto y host específico
redis-cli -h localhost -p 6379

# Con contraseña
redis-cli -a tupassword

# Ejecutar un comando directo sin entrar al CLI
redis-cli SET nombre "Juan"
redis-cli GET nombre

# KeyDB usa el mismo cliente
redis-cli -p 6379
```

### Insertar datos — por estructura

```bash
# ── STRING ──────────────────────────────────────────────
SET usuario:1:nombre "Ana López"
SET usuario:1:email "ana@mail.com"
SET contador:visitas 0
SET session:abc123 "datos_sesion" EX 1800    # expira en 30 min
SET producto:10:precio "1200.00"

# ── HASH ────────────────────────────────────────────────
HSET producto:10 nombre "Laptop HP" precio 1200 stock 15 categoria "Electronico"
HSET usuario:1 nombre "Ana" email "ana@mail.com" ciudad "La Paz" edad 28

# ── LIST ────────────────────────────────────────────────
RPUSH cola:emails "email1@test.com"
RPUSH cola:emails "email2@test.com"
RPUSH cola:emails "email3@test.com"

LPUSH historial:usuario:1 "login_2024-01-10"
LPUSH historial:usuario:1 "compra_2024-01-11"

# ── SET ─────────────────────────────────────────────────
SADD etiquetas:post:5 "tech" "python" "nosql" "database"
SADD seguidores:usuario:1 "usuario:2" "usuario:3" "usuario:5"

# ── SORTED SET ──────────────────────────────────────────
ZADD ranking:jugadores 2300 "PlayerA"
ZADD ranking:jugadores 1850 "PlayerB"
ZADD ranking:jugadores 3100 "PlayerC"
ZADD ranking:jugadores 950  "PlayerD"

# ── STREAM ──────────────────────────────────────────────
XADD sensores:temperatura * device_id "SENS-01" valor "23.5" unidad "C"
XADD sensores:temperatura * device_id "SENS-01" valor "24.1" unidad "C"
```

### Verificación rápida

```bash
# Ver tipo de una clave
TYPE usuario:1:nombre        # string
TYPE producto:10             # hash
TYPE cola:emails             # list
TYPE etiquetas:post:5        # set
TYPE ranking:jugadores       # zset

# Ver todas las claves (cuidado en producción)
KEYS *
KEYS usuario:*               # solo las de usuario

# Ver TTL de una clave
TTL session:abc123           # segundos restantes (-1 = sin TTL, -2 = no existe)

# HASH
HGETALL producto:10
HGET producto:10 precio

# LIST
LRANGE cola:emails 0 -1      # todos los elementos

# SET
SMEMBERS etiquetas:post:5

# SORTED SET
ZRANGE ranking:jugadores 0 -1 WITHSCORES      # ascendente
ZREVRANGE ranking:jugadores 0 4 WITHSCORES    # top 5 descendente

# STREAM
XRANGE sensores:temperatura - +               # todos los eventos
```

### Casos de uso desde terminal

```bash
# CACHÉ: guardar resultado con TTL
SET cache:productos:electronico "[{...}]" EX 300    # 5 min

# RATE LIMITING: máximo 10 requests por minuto
INCR ratelimit:ip:192.168.1.1
EXPIRE ratelimit:ip:192.168.1.1 60

# CONTADOR de visitas
INCR visitas:pagina:home
INCRBY stock:producto:10 -1                         # decrementar stock

# LOCK DISTRIBUIDO
SET lock:proceso_etl 1 NX EX 30    # NX = solo si no existe
# Si retorna OK → tienes el lock
# Si retorna nil → otro proceso lo tiene

# SESSION
SET session:token_xyz "user_id:123" EX 3600
GET session:token_xyz

# PUBLICAR evento (Pub/Sub)
PUBLISH canal:pedidos '{"id":"PED-001","estado":"confirmado"}'

# SUSCRIBIRSE a canal (en otra terminal)
SUBSCRIBE canal:pedidos
```

---

## 3. Neo4j

### Iniciar el servicio

```bash
# Windows — Neo4j Desktop
# Abrir Neo4j Desktop → Start (botón en la instancia)

# Windows — desde terminal (instalación manual)
cd C:\neo4j\bin
neo4j console          # modo foreground
neo4j start            # modo background
neo4j stop
neo4j status

# Linux
sudo systemctl start neo4j
sudo systemctl status neo4j
sudo systemctl stop neo4j

# Docker
docker run -d --name neo4j \
  -p 7474:7474 \
  -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password123 \
  neo4j:latest

# Verificar: abrir http://localhost:7474 en el navegador
# Usuario: neo4j | Contraseña: la que configuraste
```

### Conectar a la terminal (cypher-shell)

```bash
# Conexión básica
cypher-shell -u neo4j -p password123

# Con URI
cypher-shell -a bolt://localhost:7687 -u neo4j -p password123

# Ejecutar archivo .cypher
cypher-shell -u neo4j -p password123 < script.cypher

# Salir del shell
:exit
```

### Insertar datos

```cypher
-- Crear nodos simples
CREATE (:Persona {nombre: "Ana López", edad: 30, email: "ana@mail.com"});
CREATE (:Persona {nombre: "Luis Mamani", edad: 25, email: "luis@mail.com"});
CREATE (:Persona {nombre: "Carmen Quispe", edad: 35, email: "carmen@mail.com"});

-- Crear nodos con variable para usarlos después
CREATE (p:Producto {id: "PROD-001", nombre: "Laptop HP", precio: 1200, categoria: "Electronico"});
CREATE (c:Ciudad {nombre: "La Paz", pais: "Bolivia"});
CREATE (c:Ciudad {nombre: "Cochabamba", pais: "Bolivia"});

-- Crear nodos y relaciones en un solo comando
CREATE (a:Persona {nombre: "Ana"})-[:AMIGO_DE {desde: 2020}]->(b:Persona {nombre: "Luis"});

-- Crear relaciones entre nodos existentes (MATCH + CREATE)
MATCH (a:Persona {nombre: "Ana López"}), (b:Persona {nombre: "Luis Mamani"})
CREATE (a)-[:CONOCE {desde: 2021}]->(b);

MATCH (p:Persona {nombre: "Ana López"}), (c:Ciudad {nombre: "La Paz"})
CREATE (p)-[:VIVE_EN {desde: 2018}]->(c);

MATCH (p:Persona {nombre: "Ana López"}), (prod:Producto {nombre: "Laptop HP"})
CREATE (p)-[:COMPRÓ {fecha: "2024-01-10", monto: 1200}]->(prod);

-- Insertar múltiples nodos con UNWIND (equivale a insertMany)
UNWIND [
  {nombre: "Mario Condori", edad: 28},
  {nombre: "Sofia Flores", edad: 22},
  {nombre: "Pedro Huanca", edad: 45}
] AS datos
CREATE (:Persona {nombre: datos.nombre, edad: datos.edad});
```

### Verificación rápida

```cypher
-- Ver todos los nodos
MATCH (n) RETURN n LIMIT 25;

-- Ver nodos por label
MATCH (p:Persona) RETURN p;

-- Ver relaciones
MATCH ()-[r]->() RETURN type(r), count(r);

-- Contar nodos
MATCH (p:Persona) RETURN count(p);

-- Ver un nodo específico
MATCH (p:Persona {nombre: "Ana López"}) RETURN p;

-- Ver nodo con sus relaciones
MATCH (p:Persona {nombre: "Ana López"})-[r]->(destino)
RETURN p, r, destino;

-- Labels existentes
CALL db.labels();

-- Tipos de relaciones existentes
CALL db.relationshipTypes();
```

### Cargar desde CSV

```cypher
-- Archivo debe estar en import/ del directorio de Neo4j
-- o usar file:/// para ruta absoluta

LOAD CSV WITH HEADERS FROM 'file:///personas.csv' AS row
CREATE (:Persona {
    nombre: row.nombre,
    edad: toInteger(row.edad),
    email: row.email
});

-- Con relaciones desde CSV
LOAD CSV WITH HEADERS FROM 'file:///amistades.csv' AS row
MATCH (a:Persona {nombre: row.persona1})
MATCH (b:Persona {nombre: row.persona2})
CREATE (a)-[:AMIGO_DE]->(b);
```

---

## 4. Cassandra

### Iniciar el servicio

```bash
# Windows — iniciar manualmente
cd C:\cassandra\bin
cassandra -f             # foreground (ver logs)
cassandra                # background

# Linux
sudo systemctl start cassandra
sudo systemctl status cassandra
sudo systemctl stop cassandra

# Verificar que el nodo está listo (puede tardar 30-60 seg)
nodetool status
# Esperar hasta ver: UN  127.0.0.1  (Up + Normal)

# Docker
docker run -d --name cassandra \
  -p 9042:9042 \
  cassandra:latest

# Verificar
docker exec -it cassandra nodetool status
```

### Conectar a la terminal (cqlsh)

```bash
# Conexión básica
cqlsh

# Host y puerto específico
cqlsh localhost 9042

# Con usuario y contraseña (si está configurado)
cqlsh -u cassandra -p cassandra

# Docker
docker exec -it cassandra cqlsh

# Salir
exit
# o Ctrl+D
```

### Crear keyspace e insertar datos

```sql
-- Crear keyspace
CREATE KEYSPACE tienda
  WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 1
  };

-- Usar el keyspace
USE tienda;

-- Ver keyspaces
DESCRIBE KEYSPACES;

-- Crear tabla (diseñada para la consulta)
CREATE TABLE productos_por_categoria (
    categoria    TEXT,
    producto_id  UUID,
    nombre       TEXT,
    precio       DECIMAL,
    stock        INT,
    PRIMARY KEY ((categoria), precio, producto_id)
) WITH CLUSTERING ORDER BY (precio ASC);

CREATE TABLE pedidos_por_usuario (
    usuario_id   UUID,
    fecha        TIMESTAMP,
    pedido_id    UUID,
    total        DECIMAL,
    estado       TEXT,
    PRIMARY KEY ((usuario_id), fecha, pedido_id)
) WITH CLUSTERING ORDER BY (fecha DESC);

-- Insertar datos
INSERT INTO productos_por_categoria (categoria, producto_id, nombre, precio, stock)
VALUES ('Electronico', uuid(), 'Laptop HP', 1200.00, 15);

INSERT INTO productos_por_categoria (categoria, producto_id, nombre, precio, stock)
VALUES ('Electronico', uuid(), 'Monitor 27"', 350.00, 10);

INSERT INTO productos_por_categoria (categoria, producto_id, nombre, precio, stock)
VALUES ('Periferico', uuid(), 'Mouse Logitech', 25.00, 50);

-- Con TTL (expira en 1 hora)
INSERT INTO pedidos_por_usuario (usuario_id, fecha, pedido_id, total, estado)
VALUES (uuid(), toTimestamp(now()), uuid(), 250.00, 'pendiente')
USING TTL 3600;

-- Insertar con timestamp específico
INSERT INTO pedidos_por_usuario (usuario_id, fecha, pedido_id, total, estado)
VALUES (uuid(), '2024-01-10 10:30:00+0000', uuid(), 580.00, 'completado');
```

### Verificación rápida

```sql
-- Ver tablas
DESCRIBE TABLES;

-- Ver estructura de tabla
DESCRIBE TABLE productos_por_categoria;

-- Consultar (SIEMPRE por partition key)
SELECT * FROM productos_por_categoria WHERE categoria = 'Electronico';

SELECT * FROM productos_por_categoria
WHERE categoria = 'Electronico' AND precio > 100;

-- Contar (puede ser lento en producción)
SELECT COUNT(*) FROM productos_por_categoria;

-- Ver keyspace actual
SELECT keyspace_name FROM system_schema.keyspaces;
```

### Cargar desde CSV (cqlsh COPY)

```bash
# Dentro de cqlsh:
COPY productos_por_categoria (categoria, producto_id, nombre, precio, stock)
FROM '/ruta/productos.csv'
WITH HEADER = TRUE AND DELIMITER = ',';

# Exportar a CSV
COPY productos_por_categoria TO '/ruta/export.csv'
WITH HEADER = TRUE;
```

---

## 5. CouchDB

### Iniciar el servicio

```bash
# Windows — como servicio (instalado con el installer)
net start Apache CouchDB

# Windows — manual
cd "C:\Program Files\Apache CouchDB"
bin\couchdb.cmd

# Linux
sudo systemctl start couchdb
sudo systemctl status couchdb
sudo systemctl stop couchdb

# Docker
docker run -d --name couchdb \
  -p 5984:5984 \
  -e COUCHDB_USER=admin \
  -e COUCHDB_PASSWORD=admin123 \
  couchdb:latest

# Verificar (retorna info del servidor)
curl http://localhost:5984
# o en el navegador: http://localhost:5984/_utils  (Fauxton UI)
```

### CouchDB se maneja con HTTP — no tiene CLI propia

Usar `curl` desde la terminal:

```bash
# Variables para simplificar
HOST="http://admin:admin123@localhost:5984"

# ── BASE DE DATOS ──────────────────────────────────────
# Crear base de datos
curl -X PUT $HOST/tienda

# Listar bases de datos
curl $HOST/_all_dbs

# Eliminar base de datos
curl -X DELETE $HOST/tienda

# ── INSERTAR DOCUMENTOS ────────────────────────────────
# POST (CouchDB genera el _id automáticamente)
curl -X POST $HOST/tienda \
  -H "Content-Type: application/json" \
  -d '{
    "tipo": "producto",
    "nombre": "Laptop HP",
    "precio": 1200,
    "categoria": "Electronico",
    "stock": 15
  }'

# PUT (con _id definido por nosotros)
curl -X PUT $HOST/tienda/PROD-001 \
  -H "Content-Type: application/json" \
  -d '{
    "tipo": "producto",
    "nombre": "Mouse Logitech",
    "precio": 25,
    "categoria": "Periferico"
  }'

# Insertar varios (bulk)
curl -X POST $HOST/tienda/_bulk_docs \
  -H "Content-Type: application/json" \
  -d '{
    "docs": [
      {"tipo":"producto","nombre":"Teclado","precio":80},
      {"tipo":"producto","nombre":"Monitor","precio":350},
      {"tipo":"cliente","nombre":"Ana López","ciudad":"La Paz"}
    ]
  }'

# ── LEER ───────────────────────────────────────────────
# Leer documento por _id
curl $HOST/tienda/PROD-001

# Leer todos los documentos
curl "$HOST/tienda/_all_docs?include_docs=true"

# ── ACTUALIZAR (requiere _rev) ─────────────────────────
# Primero obtener el _rev actual
curl $HOST/tienda/PROD-001
# Respuesta: {"_id":"PROD-001","_rev":"1-abc123",...}

# Actualizar con el _rev
curl -X PUT $HOST/tienda/PROD-001 \
  -H "Content-Type: application/json" \
  -d '{
    "_rev": "1-abc123",
    "tipo": "producto",
    "nombre": "Mouse Logitech",
    "precio": 28
  }'

# ── ELIMINAR (requiere _rev) ───────────────────────────
curl -X DELETE "$HOST/tienda/PROD-001?rev=2-xyz789"

# ── BUSCAR (Mango Query) ───────────────────────────────
curl -X POST $HOST/tienda/_find \
  -H "Content-Type: application/json" \
  -d '{
    "selector": {
      "tipo": "producto",
      "precio": { "$gt": 100 }
    },
    "fields": ["nombre", "precio", "categoria"],
    "sort": [{"precio": "desc"}]
  }'
```

### Interfaz web Fauxton

```
http://localhost:5984/_utils
Usuario: admin
Contraseña: admin123

Desde Fauxton puedes:
- Crear/eliminar bases de datos con UI
- Ver y editar documentos
- Ejecutar Mango queries
- Ver índices y vistas
```

---

## 6. DynamoDB Local

> DynamoDB Local es la versión para desarrollo/testing que corre en tu máquina.

### Iniciar DynamoDB Local

```bash
# Opción 1: JAR descargado de AWS
java -Djava.library.path=./DynamoDBLocal_lib \
     -jar DynamoDBLocal.jar \
     -sharedDb \
     -port 8000

# Opción 2: Docker (más fácil)
docker run -d --name dynamodb-local \
  -p 8000:8000 \
  amazon/dynamodb-local

# Verificar
curl http://localhost:8000
```

### Configurar AWS CLI para DynamoDB Local

```bash
# Instalar AWS CLI si no está
pip install awscli

# Configurar con credenciales ficticias (para local no importan los valores)
aws configure
# AWS Access Key ID: fakekey
# AWS Secret Access Key: fakesecret
# Default region: us-east-1
# Default output format: json
```

### Insertar datos desde terminal (AWS CLI)

```bash
# Atajo: --endpoint-url siempre necesario para DynamoDB Local
ENDPOINT="--endpoint-url http://localhost:8000"

# ── CREAR TABLA ────────────────────────────────────────
aws dynamodb create-table $ENDPOINT \
  --table-name Productos \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# Ver tablas creadas
aws dynamodb list-tables $ENDPOINT

# ── INSERTAR (put-item) ────────────────────────────────
aws dynamodb put-item $ENDPOINT \
  --table-name Productos \
  --item '{
    "PK":     {"S": "PROD#001"},
    "SK":     {"S": "INFO"},
    "nombre": {"S": "Laptop HP"},
    "precio": {"N": "1200"},
    "stock":  {"N": "15"},
    "categoria": {"S": "Electronico"}
  }'

aws dynamodb put-item $ENDPOINT \
  --table-name Productos \
  --item '{
    "PK":     {"S": "USER#123"},
    "SK":     {"S": "PROFILE"},
    "nombre": {"S": "Ana López"},
    "email":  {"S": "ana@mail.com"}
  }'

aws dynamodb put-item $ENDPOINT \
  --table-name Productos \
  --item '{
    "PK":     {"S": "USER#123"},
    "SK":     {"S": "ORDER#2024-001"},
    "total":  {"N": "250"},
    "estado": {"S": "completado"}
  }'

# ── LEER ───────────────────────────────────────────────
# get-item: por clave exacta
aws dynamodb get-item $ENDPOINT \
  --table-name Productos \
  --key '{
    "PK": {"S": "PROD#001"},
    "SK": {"S": "INFO"}
  }'

# query: por PK (eficiente)
aws dynamodb query $ENDPOINT \
  --table-name Productos \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk": {"S": "USER#123"}}'

# query: PK + SK begins_with
aws dynamodb query $ENDPOINT \
  --table-name Productos \
  --key-condition-expression "PK = :pk AND begins_with(SK, :sk)" \
  --expression-attribute-values '{
    ":pk": {"S": "USER#123"},
    ":sk": {"S": "ORDER#"}
  }'

# scan: recorre toda la tabla (costoso)
aws dynamodb scan $ENDPOINT --table-name Productos

# ── ACTUALIZAR ─────────────────────────────────────────
aws dynamodb update-item $ENDPOINT \
  --table-name Productos \
  --key '{"PK": {"S": "PROD#001"}, "SK": {"S": "INFO"}}' \
  --update-expression "SET precio = :p, stock = :s" \
  --expression-attribute-values '{
    ":p": {"N": "999"},
    ":s": {"N": "20"}
  }'

# ── ELIMINAR ───────────────────────────────────────────
aws dynamodb delete-item $ENDPOINT \
  --table-name Productos \
  --key '{"PK": {"S": "PROD#001"}, "SK": {"S": "INFO"}}'
```

### Tipos de datos en DynamoDB

```
{"S": "texto"}          → String
{"N": "123"}            → Number (siempre como string en JSON)
{"BOOL": true}          → Boolean
{"NULL": true}          → Null
{"L": [...]}            → List
{"M": {...}}            → Map (objeto anidado)
{"SS": ["a","b"]}       → String Set
{"NS": ["1","2"]}       → Number Set
```

---

## 7. Referencia rápida de puertos

| Gestor | Puerto | Protocolo | URL / CLI |
|---|---|---|---|
| **MongoDB** | 27017 | TCP | `mongosh localhost:27017` |
| **Redis / KeyDB** | 6379 | TCP | `redis-cli -p 6379` |
| **Neo4j Bolt** | 7687 | Bolt | `cypher-shell -a bolt://localhost:7687` |
| **Neo4j HTTP** | 7474 | HTTP | `http://localhost:7474` |
| **Cassandra** | 9042 | CQL | `cqlsh localhost 9042` |
| **CouchDB** | 5984 | HTTP/REST | `curl http://localhost:5984` |
| **DynamoDB Local** | 8000 | HTTP | `aws dynamodb ... --endpoint-url http://localhost:8000` |

### Comandos de verificación rápida (todos)

```bash
# MongoDB
mongosh --eval "db.adminCommand({ping:1})"

# Redis
redis-cli ping

# Neo4j
curl http://localhost:7474

# Cassandra
nodetool status

# CouchDB
curl http://localhost:5984

# DynamoDB Local
aws dynamodb list-tables --endpoint-url http://localhost:8000

# Ver puertos en uso (Windows)
netstat -ano | findstr "27017\|6379\|7474\|7687\|9042\|5984\|8000"

# Ver puertos en uso (Linux)
ss -tlnp | grep -E "27017|6379|7474|7687|9042|5984|8000"
```

### Comandos Docker para todos (forma más rápida de levantar todo)

```bash
# MongoDB
docker run -d --name mongodb -p 27017:27017 mongo:latest

# Redis
docker run -d --name redis -p 6379:6379 redis:latest

# KeyDB
docker run -d --name keydb -p 6379:6379 eqalpha/keydb

# Neo4j
docker run -d --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password123 \
  neo4j:latest

# Cassandra
docker run -d --name cassandra -p 9042:9042 cassandra:latest

# CouchDB
docker run -d --name couchdb \
  -p 5984:5984 \
  -e COUCHDB_USER=admin \
  -e COUCHDB_PASSWORD=admin123 \
  couchdb:latest

# DynamoDB Local
docker run -d --name dynamodb-local -p 8000:8000 amazon/dynamodb-local

# Ver todos los contenedores corriendo
docker ps

# Detener todos
docker stop mongodb redis neo4j cassandra couchdb dynamodb-local
```

---

*Cheatsheet generado para examen BD3 — INF-262 UMSA*