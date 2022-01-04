#### 2021-12-26

# **Hackathon 2021 - TiMatch RFC**

Henry Long, Yujie Xia, Jiachen Bai

## **Abstract**

We debuted TiGraph as our first graph database attempt in 2020. This year team TiMatch brings a brand new evolution of graph database with a brand new design in both query language and storage layers.

This RFC implements the graph schema in TiDB, uses TiKV as storage layer, and supports the following features:

- A new graph database language dialect, combining completeness from Oracle PGQL and elegance from initial TiGraph clauses.
- Using TiKV as distributed storage, enabling the TiDB layer to push down tasks to TiKV coprocessors.
- Manipulating vertices/edges of graphs and records in relational tables in the same transaction.

## **Background**

### Graph Database

Graph databases have been a hot topic in the industry. It features graph structures for semantic querying, using vertices, edges, and attributes to represent and store data. Designed to support simple and fast retrieval of complex hierarchies that are difficult to model in relational database systems, it elegantly solves performance or failure issues often seen in the traditional relational databases when running for results for complex relationships or multi-table JOIN situations.

Products such as Neo4j, TigerGraph, Nebula and Dgraph have proved the value of graph databases in real-time recommendation, financial risk control, network security, knowledge graph, and other scenarios.

### Query Language

As a newly emerged database category, there is not a &quot;global standard&quot; query language for graph databases yet. Therefore producers of graph databases have to design their own graph query languages for the good of their own products. Popular languages such as Gremlin, Cypher and nGQL all have both pros and cons in different ways. Some are too complicated. Some cannot be well integrated with relational table queries. To build a graph database on top of TiDB, we need a language that is both user friendly and able to be embedded in SQL queries to fully exploit the power of TiDB.

PGQL [https://pgql-lang.org/](https://pgql-lang.org/), recently published by Oracle, is a substantial effort to establish standards for SQL-compatible graph query language with query completeness.

### Evolution

TiGraph was a well received innitiative as a graph database POC, based on TiDB in the 2020 PingCAP Hackathon. This time, team TiMatch continues the exploration in the path of building a full-fledged and easy-to-use graph database on top of TiDB.

TiGraph was designed with a simple but elegant home-grown query language. The clause **TRAVERSE..IN/OUT/BOTH** was a user friendly and straight-foward design. However it had completeness issues both externally and internally.

- Externally, it could not handle popular use cases, i.e. shortest path, pattern matching, etc.
- Internally, traversal only explicitly went through and filtered by edges. Therefore we could not filter by some vertices in the middle of the nested traversal.

Therefore, this time we decide to adopt PGQL&#39;s **MATCH** keyword together with the whole set of functionalities that follow. Also we are combining the elegant part of TiGraph ideas, to create essentially a PGQL dialect.

Another place we seek enhancement at is storage. TiGraph used a simple singleton storage to directly feed TiDB. This DID prove the versatility of key-value storage, however it DID NOT fully release the power of key-value storage: scalability and performance. This time we wire TiDB up with TiKV as a storage engine under the hood. We also implemented a graph coprocessor so that TiDB could push down tasks to scan and preprocess the graph data to TiKV nodes during traversal.

The codec inside TiKV was also a place of exciting innovation. Initially in 2020 we encoded vertices with a specific rule, whose functionality we later found the original TiDB table encoding algorithm could have totally covered. Therefore in this new evolution, we designed the encoding in a more carefully considered pattern to better facilitate customer&#39;s seamless data migration while fulfilling functionality and performance requirements.

## **Proposal**

### Syntax

The section defines the syntax that is implemented as the basics of manipulation of graph databases.

#### Concepts

- **VERTEX**: The VERTEX represents a vertex member of a graph, connected or not connected by EDGEs.
- **EDGE**: The EDGE represents the relationship between two vertices. An EDGE must have a source(src) VERTEX and a destination(dst) VERTEX.

#### DDL

The data description language (DDL) is a syntax for creating and modifying database objects such as tables, indices, and users. The DDL in the current context is the statement for defining VERTEX/EDGE schemas.

There were quite amout of odds in DDL design and implementations as simply a POC in Hackathon 2020 TiGraph:

- Since we totally separated encoding from normal tables (especially for vertices which later proved to be unnecessary), we had to implement all VERTEX/EDGE related DDL statements syntax, like:
  - SHOW VERTECIES/EDGES
  - SHOW CREATE VERTEX/EDGE name;
  - DROP VERTEX/EDGE name;
  - ALTER VERTEX/EDGE name;
  - …
- Due to the point above, we had to introduce more concepts into the database and make customers&#39; learning curve steeper.
- The extra intricated syntax introduced hinders compatiblity of the existing TiDB ecosystem, like: ORM/Migration tools/Diagnositics tools/Applications.

The time team TiMatch put efforts to improve the general design, in order to introduce as few incompatible changes as possible. Therefore we could best maintain our advantage by utilizing the large ecosystem around the existing relational database system, which we build our graph database on top of.

After research and design, our team explored an optimized solution that could satisfy all the functionalities of the graph database while introducing only minor changes.

- **VERTEX**: We managed to keep using the existing DDL syntax and encoding underneath to maintain consistency with existing tables.

```
CREATE TABLE students(id **BIGINT PRIMARY KEY ,age INT ,name VARCHAR(20));

CREATE TABLE schools(id BIGINT PRIMARY KEY ,name VARCHAR(20)); 
```

By doing this, we could avoid the situation in 2020, when we had to totally redefine vertex. Now with the new design, customer meets zero learning cost and zero operation cost in the vertex part. Neither do they need to learn new syntax to define vertex, nor do they need any actual DDL changes in order to convert existing tables in TiDB to vertices.

- **EDGE**: To enforce and identify the **SRC** and **DST** key features of EDGE, two column option keywords are added into mysql language set: **SOURCE KEY** and **DESTINATION KEY**

```
CREATE TABLE student_of(
  student BIGINT SOURCE KEY  REFERENCES students,
  school BIGINT DESTINATION KEY  REFERENCES schools,
  graduated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP 
);
```

By doing this, the only learning cost for customer is that they need to simply understand an edge has to have **SOURCE KEY** and **DESTINATION KEY**. And these option key words are just similar to foreign key in mysql. The learning cost and operation cost are bare minimal for customers. The key words are easy to learn, and customers only needs to do **ALTER** command to convert an existing table into an **EDGE** table.

#### DML

The data manipulation language (DML) is used for adding (inserting), deleting, and modifying (updating) VERTEX/EDGE in the graph mode in the TiDB database. We adopt Oracle PGQL&#39;s **MATCH** clause set to achieve functionality completeness. But the vanilla PGQL&#39;s expression for an edge **-[edge]-\&gt;**is way too tricky and not elegant enough, so we decide to use TiGraph&#39;s key words **IN/OUT/BOTH** to describe an edge. With the help of PGQL, now we can achieve:

- Basic CRUD operations to vertices or edges.
- Insert graph data.
```
INSERT INTO people(vertex_id, name, id)
VALUES (1, "aaa", 2021);

INSERT INTO friend(from, to) VALUES (1, 2);
```
- Query graph data like query table.
```
SELECT * FROM people WHERE vertex_id = 1;
SELECT * FROM friend WHERE vertex_id = 1;
```
- Updates
- Deletions
- Graph traversal query
```
SELECT * FROM
  MATCH (p1).BOTH(knows).BOTH(knows).BOTH(knows).(p2)
  ORDER BY name LIMIT 100;
```

- More cool features yet to come.

### Graph Data Codec

Because there is MVCC, the TiKV stored Key already has timestamp information, so the TiDB layer Key no longer has timestamp information.

**Vertex Key**

| t\_prefix | Vertex\_Type\_ID | Vertex\_ID |
| --- | --- | --- |

- t\_prefix: Simply normal table record prefix. No difference from common tables.
- Vertex\_Type\_ID: The type ID of the vertex. Actually exactly the table ID in tables.
- Vertex\_ID: The identity of the single vertex.

**Outbound Key**

| g\_prefix | srcVertex\_ID | Direction | Edge\_ID | dstVertex\_ID |
| --- | --- | --- | --- | --- |

- srcVertex\_ID: The identity of the source vertex
- Direction: The value 1 represent the outbound edge
- Edge\_ID： The identity of the Edge
- dstVertex\_ID: The identity of the destination Vertex

**Inbound Key**

| g\_prefix | dstVertex\_ID | Direction | Edge\_ID | srcVertex\_ID |
| --- | --- | --- | --- | --- |

- dstVertex\_ID: The identity of the destination Vertex
- Direction: The value 0 represent the inbound edge
- Edge\_ID： The identity of the Edge
- srcVertex\_ID: The identity of the source Vertex

### Storage and Coprocessor

Taking advantage of TiKV&#39;s coprocessor, we can offload a pretty large amount of computation from TiDB to TiKV. Therefore TiDB would save a lot of computation resources, as well as network traffic. It is especially important when we are traversing a graph while we have a lot of **SELECT…WHERE** or **LIMIT** tasks to process.

Similar to **BatchTableScanExecutor** coprocessor in TiKV, we implemented a graph coprocessor **BatchGraphScanExecutor** to scan through TiKV with an input key range, and then return with well decoded and formatted data. Thanks to the existing mechanisms already established in TiKV out of the box, the new coprocessor could be easily wired up with other upstream coprocessors such as **BatchSelectionExecutor** and **BatchTopNExecutor** to achieve comprehensive tasks with minimal code changes to TiDB and TiKV.
