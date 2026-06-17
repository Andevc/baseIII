# 📦 Cheatsheet — Bases de Datos Distribuidas
> Repaso completo para examen | BD III · UMSA Informática

---

## 1. ¿Qué es una Base de Datos Distribuida?

Una **Base de Datos Distribuida (BDD)** es una colección de datos que pertenece lógicamente a un mismo sistema, pero que está físicamente distribuida en múltiples nodos (computadoras) interconectadas en red.

> ⚡ **Diferencia clave:** No es lo mismo que tener copias de respaldo. En una BDD, los datos se gestionan de forma coordinada como si fueran uno solo, aunque estén en sitios distintos.

**Componentes principales:**
- **Nodo / Sitio:** Cada computadora participante con su propio SGBD local.
- **Red de comunicación:** Conecta los nodos (LAN, WAN, Internet).
- **SGBDD:** Sistema Gestor de Bases de Datos Distribuidas — el software que coordina todo.

---

## 2. Objetivos de una BDD

| Objetivo | Descripción |
|---|---|
| **Transparencia** | El usuario ve un sistema único, no múltiples nodos |
| **Disponibilidad** | Si un nodo falla, los demás siguen operando |
| **Autonomía local** | Cada nodo puede operar de forma independiente |
| **Rendimiento** | Las consultas se ejecutan cerca de los datos |
| **Escalabilidad** | Se pueden agregar nodos sin rediseñar todo |

---

## 3. Ventajas y Desventajas

### ✅ Ventajas
- **Autonomía local:** cada sitio gestiona sus propios datos.
- **Alta disponibilidad:** tolerancia a fallos de nodos individuales.
- **Mejor rendimiento:** procesamiento paralelo y datos cerca del usuario.
- **Expansión incremental:** se agregan nodos sin parar el sistema.
- **Reflejo de la organización:** una empresa distribuida tiene datos distribuidos.

### ❌ Desventajas
- **Complejidad:** control de concurrencia y recuperación son mucho más difíciles.
- **Seguridad:** mayor superficie de ataque.
- **Integridad:** mantener consistencia entre nodos es costoso.
- **Coste de comunicación:** las transacciones distribuidas generan mucho tráfico.
- **Falta de estándares:** cada SGBDD implementa las cosas a su manera.

---

## 4. Transparencia en BDD

La **transparencia** es la propiedad que oculta la complejidad de la distribución al usuario.

| Tipo de Transparencia | ¿Qué oculta? | Ejemplo |
|---|---|---|
| **De ubicación** | Dónde están los datos | `SELECT * FROM Clientes` (no sé en qué nodo está) |
| **De fragmentación** | Que los datos están fragmentados | La tabla se ve completa aunque esté partida |
| **De replicación** | Que hay copias en varios nodos | Leer de la copia más cercana sin saberlo |
| **De concurrencia** | Que otros acceden al mismo tiempo | Las transacciones parecen seriales |
| **De fallos** | Que un nodo cayó | El sistema sigue funcionando |
| **De migración** | Que los datos se mueven entre nodos | Los datos se reubican sin interrupciones |

---

## 5. Fragmentación

La fragmentación define **cómo se divide una tabla** entre los nodos. Es uno de los conceptos más importantes del examen.

### 5.1 Fragmentación Horizontal

Divide las filas (tuplas) entre nodos según una condición.

```
Tabla EMPLEADOS (id, nombre, ciudad, salario)

Fragmento 1 → Nodo La Paz:  WHERE ciudad = 'La Paz'
Fragmento 2 → Nodo Santa Cruz: WHERE ciudad = 'Santa Cruz'
Fragmento 3 → Nodo Cochabamba: WHERE ciudad = 'Cochabamba'
```

- **Regla de completitud:** Toda tupla debe pertenecer al menos a un fragmento.
- **Regla de disjunción:** Una tupla no debe repetirse en fragmentos distintos (salvo replicación).
- **Regla de reconstrucción:** `F1 ∪ F2 ∪ F3 = Tabla completa` (con UNION ALL).

### 5.2 Fragmentación Vertical

Divide las columnas (atributos) entre nodos. **Cada fragmento debe incluir la clave primaria.**

```
Tabla EMPLEADOS (id, nombre, ciudad, salario, historial_medico)

Fragmento 1 → Nodo RRHH:    (id, nombre, ciudad, salario)
Fragmento 2 → Nodo Médico:  (id, historial_medico)
```

- **Reconstrucción:** `F1 ⋈ F2` (JOIN por la clave primaria).
- Útil cuando diferentes áreas necesitan diferentes columnas.

### 5.3 Fragmentación Mixta (Híbrida)

Combina horizontal y vertical. Primero se fragmenta de una forma y luego de la otra.

```
Tabla → Fragmentación Horizontal → luego cada fragmento → Fragmentación Vertical
   o
Tabla → Fragmentación Vertical → luego cada fragmento → Fragmentación Horizontal
```

---

## 6. Replicación

La **replicación** consiste en mantener **copias de los datos en varios nodos**.

### Tipos de replicación

| Tipo | Descripción | Pros | Contras |
|---|---|---|---|
| **Sin replicación** | Cada fragmento existe solo en un nodo | Fácil consistencia | Sin tolerancia a fallos |
| **Replicación total** | Cada nodo tiene copia completa de toda la BD | Máxima disponibilidad | Actualizaciones costosísimas |
| **Replicación parcial** | Solo algunos fragmentos se replican | Balance entre costo y disponibilidad | Mayor complejidad |

### Propagación de actualizaciones

- **Síncrona:** Todos los nodos se actualizan antes de confirmar la transacción. → Consistencia fuerte, más lento.
- **Asíncrona:** Se actualiza el nodo primario y los demás se sincronizan después. → Más rápido, puede haber datos desactualizados temporalmente.

---

## 7. Diseño de una BDD — Fases

```
1. Diseño conceptual (ER global)
        ↓
2. Fragmentación (¿cómo partir los datos?)
        ↓
3. Asignación (¿en qué nodo va cada fragmento?)
        ↓
4. Replicación (¿qué se copia y dónde?)
        ↓
5. Esquema físico local (cada nodo configura su SGBD)
```

### Criterios para asignación de fragmentos

- **Localidad de referencia:** El fragmento va donde más se usa.
- **Capacidad del nodo:** No sobrecargar un nodo con todo.
- **Costo de comunicación:** Minimizar transferencias entre nodos.

---

## 8. Procesamiento de Consultas Distribuidas

Cuando haces una consulta en una BDD, el sistema debe:

1. **Descomponer** la consulta en sub-consultas para cada nodo relevante.
2. **Localizar** qué fragmentos/réplicas son necesarios.
3. **Optimizar** el plan de ejecución (¿dónde hago el JOIN? ¿qué muevo?).
4. **Ejecutar** en paralelo en los nodos.
5. **Reunir** los resultados.

### El problema del semi-join

Para reducir datos transferidos en red, se usa la operación **semi-join**:

```sql
-- En lugar de enviar toda la tabla B al nodo A para hacer el JOIN:
-- 1. Se proyecta solo la columna de unión de B → B'
-- 2. Se envía B' al nodo A
-- 3. Nodo A hace el JOIN con su parte y devuelve el resultado
-- Ahorra mucho ancho de banda cuando B es grande
```

---

## 9. Control de Concurrencia Distribuido

### Problema: si dos nodos actualizan el mismo dato al mismo tiempo → inconsistencia.

### 9.1 Protocolo de Bloqueo Distribuido (2PL Distribuido)

Igual que 2PL local, pero los bloqueos se coordinan entre nodos.

- **Fase de crecimiento:** solo se adquieren bloqueos.
- **Fase de decrecimiento:** solo se liberan bloqueos.

**Gestor de bloqueos centralizado:** Un nodo central gestiona todos los bloqueos.
- ✅ Simple de implementar.
- ❌ Cuello de botella y punto único de fallo.

**Gestor de bloqueos distribuido:** Cada nodo gestiona los bloqueos de sus propios datos.
- ✅ No hay cuello de botella.
- ❌ Mayor complejidad, riesgo de deadlock distribuido.

### 9.2 Protocolo de Marca de Tiempo (Timestamp Ordering)

A cada transacción se le asigna un **timestamp** único al inicio. Las operaciones se ordenan según ese timestamp.

- Si una transacción "más vieja" quiere leer/escribir datos tocados por una "más joven" ya confirmada → se **aborta y reinicia**.

### 9.3 Control de Concurrencia Optimista (OCC)

- **Fase de lectura:** la transacción lee y trabaja en copias locales.
- **Fase de validación:** antes de confirmar, verifica si hubo conflictos.
- **Fase de escritura:** si no hubo conflictos, aplica los cambios.

Útil cuando los conflictos son raros.

---

## 10. Gestión de Transacciones Distribuidas

Una **transacción distribuida** involucra múltiples nodos. Necesita garantizar las **propiedades ACID** de forma global.

| Propiedad | Significado |
|---|---|
| **Atomicidad** | Todo o nada — si falla un nodo, se deshace todo |
| **Consistencia** | La BD pasa de un estado válido a otro válido |
| **Aislamiento** | Las transacciones no se interfieren entre sí |
| **Durabilidad** | Los cambios confirmados persisten aunque haya fallos |

### El problema principal: ¿Cómo garantizar Atomicidad si un nodo falla a mitad?

---

## 11. Protocolo de Confirmación en Dos Fases (2PC)

El **Two-Phase Commit (2PC)** es el protocolo estándar para confirmar transacciones distribuidas.

### Roles
- **Coordinador:** El nodo que inició la transacción y dirige el protocolo.
- **Participantes:** Los demás nodos involucrados en la transacción.

### Fase 1: Preparación (Voting Phase)

```
Coordinador → "¿Puedes confirmar?" → a todos los Participantes

Cada Participante:
  - Escribe en su log los cambios (sin aplicarlos aún)
  - Responde "LISTO" si puede confirmar
  - Responde "ABORT" si algo falló
```

### Fase 2: Decisión (Commit Phase)

```
Si TODOS respondieron "LISTO":
    Coordinador → "COMMIT" → a todos
    Cada participante aplica los cambios y responde "ACK"

Si AL MENOS UNO respondió "ABORT":
    Coordinador → "ROLLBACK" → a todos
    Cada participante deshace los cambios
```

### Problemas del 2PC

- **Bloqueo:** Si el coordinador falla después de enviar "COMMIT", los participantes quedan bloqueados esperando.
- **Solución parcial:** 3PC (Three-Phase Commit) reduce el riesgo de bloqueo pero es más complejo.

---

## 12. Protocolo de Confirmación en Tres Fases (3PC)

Agrega una fase intermedia para evitar el bloqueo del 2PC.

```
Fase 1: PREPARACIÓN    → igual que 2PC fase 1
Fase 2: PRE-COMMIT     → Coordinador avisa "voy a hacer commit" (nueva fase)
Fase 3: COMMIT         → Confirmación final

Si un participante no recibe respuesta del coordinador en Fase 2,
puede asumir que los demás también recibieron PRE-COMMIT
y proceder sin el coordinador.
```

---

## 13. Recuperación ante Fallos

### Tipos de fallos

| Tipo | Descripción | Recuperación |
|---|---|---|
| **Fallo de transacción** | Abort por error lógico o deadlock | ROLLBACK local |
| **Fallo de nodo** | Un servidor cae (crash) | Recuperación con log (redo/undo) |
| **Fallo de red** | Partición de la red | Protocolos de consenso, espera |
| **Fallo de disco** | Pérdida de almacenamiento | Réplicas, backups |

### Write-Ahead Logging (WAL)

Antes de modificar un dato en disco, se escribe primero en el **log**. Si hay fallo:

- **REDO:** Re-aplicar operaciones de transacciones confirmadas (para durabilidad).
- **UNDO:** Deshacer operaciones de transacciones no confirmadas (para atomicidad).

---

## 14. Teorema CAP

Todo sistema distribuido solo puede garantizar **2 de estas 3 propiedades** al mismo tiempo:

```
         C — Consistencia
        / \
       /   \
      A --- P
  Disponibilidad   Tolerancia a Particiones
```

| Combinación | ¿Qué sacrifica? | Ejemplos |
|---|---|---|
| **CA** | Tolerancia a particiones | SGBD relacionales tradicionales (en red confiable) |
| **CP** | Disponibilidad | MongoDB (modo strict), HBase, Zookeeper |
| **AP** | Consistencia fuerte | CouchDB, Cassandra, DynamoDB |

> ⚡ **Importante:** En un sistema distribuido real, las particiones de red **siempre pueden ocurrir**, así que en la práctica se elige entre **CP** o **AP**.

---

## 15. Teorema PACELC

Extiende CAP: incluso **sin partición**, hay que elegir entre **latencia** y **consistencia**.

```
Si hay Partición (P):
    elegir entre Disponibilidad (A) o Consistencia (C)
Sino (E = Else):
    elegir entre Latencia (L) o Consistencia (C)
```

---

## 16. Consistencia Eventual

En sistemas AP (como Cassandra), se usa **consistencia eventual**:

- No se garantiza que todos los nodos tengan el mismo valor **en todo momento**.
- Se garantiza que **eventualmente** (si no hay nuevas escrituras) todos convergen al mismo valor.
- Es el modelo BASE: **B**asically Available, **S**oft state, **E**ventually consistent.

**BASE vs ACID:**

| | ACID | BASE |
|---|---|---|
| Consistencia | Fuerte, inmediata | Eventual |
| Disponibilidad | Puede sacrificarse | Prioridad |
| Rigidez | Alta | Flexible |
| Ejemplo | PostgreSQL, SQL Server | Cassandra, DynamoDB |

---

## 17. Catálogo Distribuido

El **catálogo** (o diccionario de datos) en una BDD almacena metadatos: qué fragmentos existen, dónde están, cómo se reconstruyen, etc.

### Tipos de organización del catálogo

| Tipo | Descripción | Ventaja | Desventaja |
|---|---|---|---|
| **Centralizado** | Todo el catálogo en un nodo | Simple | Cuello de botella |
| **Completamente replicado** | Cada nodo tiene copia total | Lectura rápida | Actualizaciones costosas |
| **Particionado** | Cada nodo guarda info de sus propios datos | Autonomía local | Consultas entre nodos complicadas |
| **Particionado + replicado** | Híbrido | Balance | Complejidad media |

---

## 18. Arquitecturas de BDD

### 18.1 Arquitectura Cliente-Servidor
- Los clientes hacen peticiones, los servidores las procesan.
- Simple pero el servidor puede ser cuello de botella.

### 18.2 Arquitectura Peer-to-Peer (P2P)
- Todos los nodos son iguales, cualquiera puede ser cliente o servidor.
- Mayor tolerancia a fallos.

### 18.3 Arquitectura Maestro-Esclavo
- Un nodo maestro gestiona las escrituras.
- Los esclavos replican y sirven lecturas.
- Ejemplo: MySQL Replication, SQL Server Availability Groups.

### 18.4 Multi-Maestro
- Varios nodos aceptan escrituras.
- Requiere resolución de conflictos.
- Ejemplo: CouchDB multi-master, Galera Cluster.

---

## 19. Tipos de Sistemas Distribuidos por Heterogeneidad

| Tipo | Descripción |
|---|---|
| **Homogéneo** | Todos los nodos usan el mismo SGBD y esquema |
| **Heterogéneo** | Nodos con distintos SGBD (SQL Server + PostgreSQL + MySQL) |
| **Federado** | Cada nodo es autónomo; la federación permite consultas globales con mínima interferencia |

---

## 20. Resolución de Deadlocks Distribuidos

Un **deadlock distribuido** ocurre cuando hay ciclos de espera que cruzan varios nodos.

### Detección con grafo de espera (Wait-For Graph)

- Cada nodo mantiene un grafo local de "quién espera a quién".
- Periódicamente se fusionan todos los grafos en uno global.
- Si hay un **ciclo** → hay deadlock → se aborta una transacción del ciclo.

### Prevención

- **Wait-Die:** Una transacción más vieja espera; una más joven muere (se aborta).
- **Wound-Wait:** Una transacción más vieja "hiere" (aborta) a la más joven; una más joven espera.

---

## 21. Resumen Visual — Mapa de Conceptos

```
BASES DE DATOS DISTRIBUIDAS
│
├── DISEÑO
│   ├── Fragmentación Horizontal (filas por condición)
│   ├── Fragmentación Vertical (columnas + PK)
│   └── Fragmentación Mixta (combinación)
│
├── REPLICACIÓN
│   ├── Total / Parcial / Sin réplicas
│   └── Síncrona / Asíncrona
│
├── TRANSPARENCIA
│   └── Ubicación / Fragmentación / Replicación / Concurrencia / Fallos
│
├── TRANSACCIONES
│   ├── ACID distribuido
│   ├── 2PC (Confirmación en Dos Fases)
│   └── 3PC (Confirmación en Tres Fases)
│
├── CONCURRENCIA
│   ├── 2PL Distribuido (bloqueos)
│   ├── Timestamp Ordering
│   └── Optimistic Concurrency Control (OCC)
│
├── FALLOS Y RECUPERACIÓN
│   ├── WAL (Write-Ahead Logging)
│   ├── REDO / UNDO
│   └── Tipos de fallos (transacción, nodo, red, disco)
│
└── TEOREMAS Y MODELOS
    ├── CAP (Consistencia, Disponibilidad, Tolerancia a particiones)
    ├── PACELC (extiende CAP con Latencia)
    ├── ACID (consistencia fuerte)
    └── BASE (consistencia eventual)
```

---

## 22. Preguntas Frecuentes de Examen

**¿Cuál es la diferencia entre fragmentación horizontal y vertical?**
> Horizontal divide filas (por condición WHERE); vertical divide columnas (diferentes atributos en diferentes nodos, siempre incluyendo la PK).

**¿Qué problema resuelve el 2PC y cuál es su debilidad?**
> Resuelve la atomicidad distribuida (todo-o-nada). Su debilidad es que si el coordinador falla después de la fase 1, los participantes quedan bloqueados indefinidamente.

**¿Qué es el Teorema CAP?**
> Todo sistema distribuido solo puede garantizar 2 de 3: Consistencia, Disponibilidad, Tolerancia a Particiones. En la práctica se elige entre CP o AP porque P siempre se necesita.

**¿Cuándo usarías fragmentación vertical?**
> Cuando diferentes grupos de usuarios o aplicaciones acceden a subconjuntos diferentes de columnas de una tabla. Ejemplo: RRHH accede a datos de nómina; Médicos acceden al historial clínico.

**¿Qué es transparencia de replicación?**
> El usuario no sabe cuántas copias existen ni cuál consulta; el sistema elige la réplica más conveniente automáticamente.

**¿Diferencia entre ACID y BASE?**
> ACID garantiza consistencia inmediata y fuerte (sistemas relacionales). BASE acepta consistencia eventual a cambio de mayor disponibilidad y rendimiento (NoSQL distribuido).

---

---

## 23. Linked Servers en SQL Server

Un **Linked Server** permite que SQL Server ejecute consultas contra **fuentes de datos externas** (otro SQL Server, PostgreSQL, MySQL, Oracle, Excel, etc.) como si fueran tablas locales. Es la implementación práctica de una BDD heterogénea/homogénea en el ecosistema Microsoft.

### Sintaxis general para consultar un Linked Server

```sql
-- Notación de 4 partes: [servidor].[base_de_datos].[esquema].[tabla]
SELECT * FROM [NombreLinkedServer].[BaseDeDatos].[dbo].[Tabla]

-- Con OPENQUERY (más eficiente, la consulta se ejecuta en el servidor remoto)
SELECT * FROM OPENQUERY([NombreLinkedServer], 'SELECT * FROM esquema.tabla')
```

> ⚡ **OPENQUERY es preferible** porque envía la consulta completa al servidor remoto y solo trae el resultado. La notación de 4 partes trae toda la tabla y filtra localmente.

---

## 24. Linked Server — Homogéneo (SQL Server → SQL Server)

Ambos nodos corren SQL Server. Es el caso más simple.

### Paso 1: Crear el Linked Server

```sql
-- Ejecutar en el SQL Server ORIGEN (el que va a consultar al remoto)
EXEC sp_addlinkedserver
    @server     = N'NODO_REMOTO',          -- Nombre que le damos al linked server
    @srvproduct = N'SQL Server';           -- Producto: 'SQL Server' para homogéneo

-- Si el servidor remoto tiene instancia nombrada:
EXEC sp_addlinkedserver
    @server     = N'NODO_REMOTO',
    @srvproduct = N'SQL Server',
    @datasrc    = N'192.168.1.10\SQLEXPRESS';  -- IP\NombreInstancia o solo IP
```

### Paso 2: Configurar credenciales de acceso

```sql
-- Opción A: Mapear login local a login remoto (recomendado)
EXEC sp_addlinkedsrvlogin
    @rmtsrvname  = N'NODO_REMOTO',
    @useself     = N'FALSE',
    @locallogin  = NULL,               -- NULL = aplica a cualquier login local
    @rmtuser     = N'sa',              -- Usuario en el servidor remoto
    @rmtpassword = N'TuPassword123';

-- Opción B: Usar autenticación Windows integrada (mismo dominio)
EXEC sp_addlinkedsrvlogin
    @rmtsrvname = N'NODO_REMOTO',
    @useself    = N'TRUE';
```

### Paso 3: Opciones útiles de configuración

```sql
-- Habilitar RPC (necesario para llamar stored procedures remotos)
EXEC sp_serveroption @server = N'NODO_REMOTO', @optname = N'rpc',        @optvalue = N'true';
EXEC sp_serveroption @server = N'NODO_REMOTO', @optname = N'rpc out',    @optvalue = N'true';

-- Habilitar consultas distribuidas
EXEC sp_serveroption @server = N'NODO_REMOTO', @optname = N'data access', @optvalue = N'true';

-- Permitir consultas remotas con collation diferente
EXEC sp_serveroption @server = N'NODO_REMOTO', @optname = N'collation compatible', @optvalue = N'true';
```

### Paso 4: Verificar y usar

```sql
-- Verificar que el linked server responde
EXEC sp_testlinkedserver N'NODO_REMOTO';

-- Listar linked servers configurados
SELECT * FROM sys.servers WHERE is_linked = 1;

-- Consulta distribuida homogénea
SELECT
    l.id_pedido,
    l.fecha,
    r.nombre_cliente
FROM [NODO_LOCAL].[QuickShip].[dbo].[Pedidos] l
JOIN [NODO_REMOTO].[QuickShip].[dbo].[Clientes] r
    ON l.id_cliente = r.id_cliente;

-- INSERT remoto
INSERT INTO [NODO_REMOTO].[QuickShip].[dbo].[Logs]
VALUES (GETDATE(), 'Operacion completada', @@SERVERNAME);

-- Llamar stored procedure remoto
EXEC [NODO_REMOTO].[QuickShip].[dbo].[sp_ActualizarStock] @id_producto = 5, @cantidad = 100;
```

### Eliminar un Linked Server

```sql
-- Primero eliminar los logins asociados
EXEC sp_droplinkedsrvlogin @rmtsrvname = N'NODO_REMOTO', @locallogin = NULL;

-- Luego eliminar el servidor
EXEC sp_dropserver @server = N'NODO_REMOTO', @droplogins = N'droplogins';
```

---

## 25. Linked Server — Heterogéneo (SQL Server → PostgreSQL)

Para conectar SQL Server a PostgreSQL se necesita un **proveedor OLE DB**. El más usado es el **Microsoft OLE DB Provider for ODBC (MSDASQL)** junto con un driver ODBC de PostgreSQL.

### Requisitos previos

1. Instalar **PostgreSQL ODBC Driver (psqlODBC)** en el servidor SQL Server.
   - Descargar desde: https://www.postgresql.org/ftp/odbc/versions/msi/
   - Instalar la versión de 64 bits si SQL Server es 64 bits.

2. Crear un **DSN del sistema** (o usar conexión sin DSN):
   - Panel de control → Herramientas administrativas → Orígenes de datos ODBC (64 bits)
   - Agregar DSN de sistema con driver "PostgreSQL Unicode"

### Opción A: Linked Server con DSN ODBC

```sql
-- Crear linked server apuntando al DSN configurado
EXEC sp_addlinkedserver
    @server     = N'PG_NODO2',
    @srvproduct = N'PostgreSQL',
    @provider   = N'MSDASQL',
    @datasrc    = N'NombreDSN';          -- Nombre del DSN ODBC del sistema

-- Configurar credenciales PostgreSQL
EXEC sp_addlinkedsrvlogin
    @rmtsrvname  = N'PG_NODO2',
    @useself     = N'FALSE',
    @locallogin  = NULL,
    @rmtuser     = N'postgres',
    @rmtpassword = N'TuPasswordPG';
```

### Opción B: Linked Server sin DSN (connection string directo)

```sql
EXEC sp_addlinkedserver
    @server     = N'PG_NODO2',
    @srvproduct = N'PostgreSQL',
    @provider   = N'MSDASQL',
    @provstr    = N'DRIVER={PostgreSQL Unicode};
                    SERVER=192.168.1.20;
                    PORT=5432;
                    DATABASE=quickship_db;
                    UID=postgres;
                    PWD=TuPasswordPG;';

EXEC sp_addlinkedsrvlogin
    @rmtsrvname  = N'PG_NODO2',
    @useself     = N'FALSE',
    @locallogin  = NULL,
    @rmtuser     = N'postgres',
    @rmtpassword = N'TuPasswordPG';
```

### Habilitar el proveedor MSDASQL (si está deshabilitado)

```sql
-- Habilitar AllowInProcess y DynamicParameters para MSDASQL
EXEC sp_MSset_oledb_prop N'MSDASQL', N'AllowInProcess',    1;
EXEC sp_MSset_oledb_prop N'MSDASQL', N'DynamicParameters', 1;

-- Habilitar queries ad hoc distribuidas (necesario en algunos entornos)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'Ad Hoc Distributed Queries', 1; RECONFIGURE;
```

### Consultar PostgreSQL desde SQL Server

```sql
-- ⚡ SIEMPRE usar OPENQUERY para PostgreSQL (la notación de 4 partes falla con PG)
SELECT * FROM OPENQUERY(PG_NODO2, 'SELECT * FROM public.clientes');

-- Con filtro (el WHERE viaja a PostgreSQL)
SELECT * FROM OPENQUERY(PG_NODO2,
    'SELECT id, nombre, ciudad FROM public.clientes WHERE activo = true');

-- JOIN entre SQL Server (local) y PostgreSQL (remoto)
SELECT
    s.id_pedido,
    s.total,
    pg.nombre_cliente
FROM dbo.Pedidos s
JOIN OPENQUERY(PG_NODO2,
    'SELECT id_cliente, nombre_cliente FROM public.clientes') pg
    ON s.id_cliente = pg.id_cliente;

-- INSERT en PostgreSQL desde SQL Server
INSERT INTO OPENQUERY(PG_NODO2, 'SELECT id, descripcion FROM public.logs')
VALUES (DEFAULT, 'Registro desde SQL Server');

-- UPDATE en PostgreSQL
UPDATE OPENQUERY(PG_NODO2, 'SELECT id, stock FROM public.productos WHERE id = 10')
SET stock = 50;
```

### Verificar conectividad

```sql
EXEC sp_testlinkedserver N'PG_NODO2';

-- Ver tablas disponibles en PostgreSQL
SELECT * FROM OPENQUERY(PG_NODO2,
    'SELECT table_name FROM information_schema.tables
     WHERE table_schema = ''public''');
```

---

## 26. Transacciones Distribuidas con Linked Servers

Para que una transacción abarque SQL Server + el servidor remoto se usa el **Coordinador de Transacciones Distribuidas de Microsoft (MSDTC)**.

```sql
-- Habilitar MSDTC en ambos nodos (ejecutar en cada servidor)
-- (Configuración en Servicios de Windows → Distributed Transaction Coordinator)

-- Usar transacción distribuida
BEGIN DISTRIBUTED TRANSACTION;

    -- Operación en nodo local (SQL Server)
    UPDATE dbo.Inventario
    SET stock = stock - 10
    WHERE id_producto = 5;

    -- Operación en nodo remoto (SQL Server o PostgreSQL vía OPENQUERY)
    UPDATE OPENQUERY(PG_NODO2,
        'SELECT id_producto, stock FROM public.inventario WHERE id_producto = 5')
    SET stock = stock + 10;

COMMIT TRANSACTION;
-- Si cualquiera falla → ROLLBACK automático en ambos nodos (2PC internamente)
```

---

## 27. Conexiones desde Python

### Librerías necesarias

```bash
pip install pyodbc          # Conexión a SQL Server (y ODBC en general)
pip install psycopg2-binary  # Conexión nativa a PostgreSQL
pip install sqlalchemy       # ORM / abstracción para ambos
pip install pandas           # Opcional: para trabajar con resultados como DataFrames
```

---

### 27.1 Conexión a SQL Server desde Python

```python
import pyodbc

# --- Configuración de conexión ---
SQL_SERVER_CONFIG = {
    "server":   "192.168.1.10",       # IP o nombre del servidor
    "port":     "1433",
    "database": "QuickShip",
    "username": "sa",
    "password": "TuPassword123",
}

def get_sqlserver_connection():
    conn_str = (
        f"DRIVER={{ODBC Driver 17 for SQL Server}};"
        f"SERVER={SQL_SERVER_CONFIG['server']},{SQL_SERVER_CONFIG['port']};"
        f"DATABASE={SQL_SERVER_CONFIG['database']};"
        f"UID={SQL_SERVER_CONFIG['username']};"
        f"PWD={SQL_SERVER_CONFIG['password']};"
        f"TrustServerCertificate=yes;"  # Para desarrollo sin SSL válido
    )
    return pyodbc.connect(conn_str)

# --- Uso básico ---
with get_sqlserver_connection() as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT id, nombre FROM dbo.Clientes WHERE activo = 1")
    for row in cursor.fetchall():
        print(row.id, row.nombre)
```

### 27.2 Consulta con parámetros (evitar SQL Injection)

```python
def buscar_pedido(id_pedido: int):
    with get_sqlserver_connection() as conn:
        cursor = conn.cursor()
        # Usar ? como placeholder en pyodbc (NO concatenar strings)
        cursor.execute(
            "SELECT id_pedido, total, estado FROM dbo.Pedidos WHERE id_pedido = ?",
            (id_pedido,)
        )
        return cursor.fetchone()
```

### 27.3 INSERT / UPDATE / DELETE en SQL Server

```python
def insertar_pedido(id_cliente: int, total: float, estado: str):
    with get_sqlserver_connection() as conn:
        cursor = conn.cursor()
        cursor.execute(
            """INSERT INTO dbo.Pedidos (id_cliente, total, estado, fecha)
               VALUES (?, ?, ?, GETDATE())""",
            (id_cliente, total, estado)
        )
        conn.commit()  # ⚡ Siempre hacer commit para DML

def actualizar_estado(id_pedido: int, nuevo_estado: str):
    with get_sqlserver_connection() as conn:
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE dbo.Pedidos SET estado = ? WHERE id_pedido = ?",
            (nuevo_estado, id_pedido)
        )
        conn.commit()
        return cursor.rowcount  # Filas afectadas
```

### 27.4 Llamar Stored Procedures en SQL Server

```python
def ejecutar_sp_actualizar_stock(id_producto: int, cantidad: int):
    with get_sqlserver_connection() as conn:
        cursor = conn.cursor()
        # Stored procedures con EXEC
        cursor.execute(
            "EXEC dbo.sp_ActualizarStock @id_producto = ?, @cantidad = ?",
            (id_producto, cantidad)
        )
        conn.commit()

def sp_con_output(id_pedido: int):
    with get_sqlserver_connection() as conn:
        cursor = conn.cursor()
        # SP con parámetro OUTPUT
        cursor.execute(
            "{CALL dbo.sp_ObtenerTotalPedido(?, ?)}",
            (id_pedido, 0)  # el 0 es placeholder para el OUTPUT
        )
        resultado = cursor.fetchone()
        return resultado
```

---

### 27.5 Conexión a PostgreSQL desde Python

```python
import psycopg2
from psycopg2.extras import RealDictCursor

# --- Configuración ---
PG_CONFIG = {
    "host":     "192.168.1.20",
    "port":     5432,
    "dbname":   "quickship_db",
    "user":     "postgres",
    "password": "TuPasswordPG",
}

def get_postgres_connection():
    return psycopg2.connect(**PG_CONFIG)

# --- Consulta básica ---
def obtener_clientes_activos():
    with get_postgres_connection() as conn:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            # RealDictCursor devuelve filas como diccionarios
            cur.execute("SELECT id, nombre, ciudad FROM clientes WHERE activo = TRUE")
            return cur.fetchall()  # Lista de dicts: [{'id': 1, 'nombre': '...'}]
```

### 27.6 INSERT / UPDATE en PostgreSQL

```python
def insertar_cliente(nombre: str, ciudad: str, email: str):
    with get_postgres_connection() as conn:
        with conn.cursor() as cur:
            # %s como placeholder en psycopg2 (no ? como en pyodbc)
            cur.execute(
                """INSERT INTO clientes (nombre, ciudad, email, activo)
                   VALUES (%s, %s, %s, TRUE)
                   RETURNING id""",   # RETURNING devuelve el ID insertado
                (nombre, ciudad, email)
            )
            nuevo_id = cur.fetchone()[0]
        conn.commit()
    return nuevo_id

def actualizar_stock_pg(id_producto: int, nuevo_stock: int):
    with get_postgres_connection() as conn:
        with conn.cursor() as cur:
            cur.execute(
                "UPDATE productos SET stock = %s WHERE id = %s",
                (nuevo_stock, id_producto)
            )
        conn.commit()
```

---

### 27.7 Consulta distribuida desde Python (SQL Server + PostgreSQL)

Este patrón conecta a ambas BDs y hace el JOIN en Python (útil cuando no se tiene Linked Server configurado).

```python
import pyodbc
import psycopg2
import pandas as pd

def consulta_distribuida_pedidos_clientes():
    """
    Pedidos están en SQL Server (Nodo 1).
    Clientes están en PostgreSQL (Nodo 2).
    Se hace el JOIN en Python.
    """
    # 1. Traer pedidos de SQL Server
    with get_sqlserver_connection() as conn_sql:
        df_pedidos = pd.read_sql(
            "SELECT id_pedido, id_cliente, total, estado FROM dbo.Pedidos",
            conn_sql
        )

    # 2. Traer clientes de PostgreSQL
    with get_postgres_connection() as conn_pg:
        df_clientes = pd.read_sql(
            "SELECT id, nombre, ciudad FROM clientes",
            conn_pg
        )

    # 3. JOIN en memoria con pandas
    df_resultado = pd.merge(
        df_pedidos,
        df_clientes,
        left_on="id_cliente",
        right_on="id",
        how="inner"
    )

    return df_resultado[["id_pedido", "nombre", "ciudad", "total", "estado"]]
```

---

### 27.8 Consulta a Linked Server desde Python

Si el Linked Server ya está configurado en SQL Server, Python solo habla con SQL Server y este se encarga de la distribución.

```python
def consulta_via_linked_server():
    """
    Python → SQL Server → (Linked Server) → PostgreSQL
    Python no sabe que hay un segundo motor involucrado.
    """
    with get_sqlserver_connection() as conn:
        cursor = conn.cursor()
        # SQL Server se encarga de hacer la consulta a PG_NODO2
        cursor.execute("""
            SELECT
                p.id_pedido,
                p.total,
                pg.nombre_cliente
            FROM dbo.Pedidos p
            JOIN OPENQUERY(PG_NODO2,
                'SELECT id_cliente, nombre_cliente FROM public.clientes') pg
                ON p.id_cliente = pg.id_cliente
            WHERE p.estado = 'PENDIENTE'
        """)
        return cursor.fetchall()
```

---

### 27.9 Transacción distribuida desde Python

```python
def transferir_stock_entre_nodos(id_producto: int, cantidad: int):
    """
    Descuenta stock en SQL Server y lo suma en PostgreSQL.
    Manejo manual de la transacción (sin MSDTC).
    """
    conn_sql = None
    conn_pg  = None
    try:
        conn_sql = get_sqlserver_connection()
        conn_pg  = get_postgres_connection()

        # Deshabilitar autocommit para control manual
        conn_sql.autocommit = False
        conn_pg.autocommit  = False

        cur_sql = conn_sql.cursor()
        cur_pg  = conn_pg.cursor()

        # Operación en SQL Server
        cur_sql.execute(
            "UPDATE dbo.Inventario SET stock = stock - ? WHERE id_producto = ?",
            (cantidad, id_producto)
        )

        # Operación en PostgreSQL
        cur_pg.execute(
            "UPDATE inventario SET stock = stock + %s WHERE id_producto = %s",
            (cantidad, id_producto)
        )

        # Confirmar ambas si todo salió bien
        conn_sql.commit()
        conn_pg.commit()
        print("Transacción distribuida exitosa.")

    except Exception as e:
        # Si algo falló → deshacer en ambas
        print(f"Error: {e} — haciendo rollback en ambos nodos.")
        if conn_sql: conn_sql.rollback()
        if conn_pg:  conn_pg.rollback()
        raise

    finally:
        if conn_sql: conn_sql.close()
        if conn_pg:  conn_pg.close()
```

---

### 27.10 Conexión con SQLAlchemy (ORM / Flask)

SQLAlchemy es la librería estándar para conectar BDs en aplicaciones Flask/FastAPI.

```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

# --- Engines ---
# SQL Server
engine_sqlserver = create_engine(
    "mssql+pyodbc://sa:TuPassword123@192.168.1.10:1433/QuickShip"
    "?driver=ODBC+Driver+17+for+SQL+Server&TrustServerCertificate=yes",
    pool_size=5,        # Conexiones en el pool
    max_overflow=10,    # Conexiones extra permitidas
    pool_timeout=30,
    echo=False          # True para ver el SQL generado (debug)
)

# PostgreSQL
engine_postgres = create_engine(
    "postgresql+psycopg2://postgres:TuPasswordPG@192.168.1.20:5432/quickship_db",
    pool_size=5,
    max_overflow=10,
)

# --- Sessions ---
SessionSQL = sessionmaker(bind=engine_sqlserver)
SessionPG  = sessionmaker(bind=engine_postgres)

# --- Uso con context manager ---
def obtener_pedidos_sqlalchemy():
    with SessionSQL() as session:
        result = session.execute(
            text("SELECT id_pedido, total FROM dbo.Pedidos WHERE estado = :estado"),
            {"estado": "PENDIENTE"}
        )
        return result.fetchall()

def obtener_clientes_sqlalchemy():
    with SessionPG() as session:
        result = session.execute(
            text("SELECT id, nombre FROM clientes WHERE activo = TRUE")
        )
        return result.mappings().all()  # Lista de dicts

# --- En Flask ---
# app.py
from flask import Flask, jsonify
app = Flask(__name__)

@app.route("/pedidos")
def pedidos():
    with SessionSQL() as session:
        rows = session.execute(text("SELECT * FROM dbo.Pedidos")).fetchall()
        return jsonify([dict(r._mapping) for r in rows])

@app.route("/clientes")
def clientes():
    with SessionPG() as session:
        rows = session.execute(text("SELECT * FROM clientes")).fetchall()
        return jsonify([dict(r._mapping) for r in rows])
```

---

### 27.11 Pool de conexiones y buenas prácticas

```python
# ✅ Buenas prácticas para producción

# 1. Usar variables de entorno para credenciales (nunca hardcodear)
import os
from dotenv import load_dotenv

load_dotenv()  # Lee .env en la raíz del proyecto

engine_sql = create_engine(
    f"mssql+pyodbc://{os.getenv('SQL_USER')}:{os.getenv('SQL_PASS')}"
    f"@{os.getenv('SQL_HOST')}:{os.getenv('SQL_PORT', 1433)}/{os.getenv('SQL_DB')}"
    f"?driver=ODBC+Driver+17+for+SQL+Server&TrustServerCertificate=yes"
)

# 2. Siempre usar context managers (with) → cierra la conexión automáticamente
# ❌ Mal:
conn = get_sqlserver_connection()
cursor = conn.cursor()
cursor.execute("SELECT ...")
# Si hay excepción aquí, la conexión queda abierta

# ✅ Bien:
with get_sqlserver_connection() as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT ...")
    # La conexión se cierra aunque haya excepción

# 3. Parametrizar siempre las queries
# ❌ Vulnerable a SQL Injection:
nombre = input()
cursor.execute(f"SELECT * FROM clientes WHERE nombre = '{nombre}'")

# ✅ Seguro:
cursor.execute("SELECT * FROM clientes WHERE nombre = ?", (nombre,))  # pyodbc
cursor.execute("SELECT * FROM clientes WHERE nombre = %s", (nombre,)) # psycopg2

# .env de ejemplo:
# SQL_HOST=192.168.1.10
# SQL_PORT=1433
# SQL_DB=QuickShip
# SQL_USER=sa
# SQL_PASS=TuPassword123
# PG_HOST=192.168.1.20
# PG_PORT=5432
# PG_DB=quickship_db
# PG_USER=postgres
# PG_PASS=TuPasswordPG
```

---

*Cheatsheet generado para examen de BD III — UMSA Informática*
