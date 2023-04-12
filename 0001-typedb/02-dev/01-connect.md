---
pageTitle: Connect
keywords: typedb, basics, connect, connection, session, database
longTailKeywords: basic concepts of typedb, typedb connection, typedb database, typedb session
summary: Brief description of connection to TypeDB.
toc: false
---

# Connect

## Clients

TypeDB server accepts remote connections from a number of [TypeDB Clients](../../02-clients/00-clients.md) via 
[gRPC](https://en.wikipedia.org/wiki/GRPC).

Once connected, TypeDB clients can manage [databases](#databases) and [sessions](#sessions).

<div class="note">
[Note]
It’s recommended to instantiate a single client per application.
</div>

<div class="tabs dark">

[tab:TypeDB Console]

<!-- test-ignore -->
```bash
typedb console --server=0.0.0.0:1729 
```

[tab:end]

[tab:Java]

```java
TypeDBClient client = TypeDB.coreClient("0.0.0.0:1729");
```

[tab:end]

[tab:Node.js]

<!-- test-ignore -->
```javascript
client = TypeDB.coreClient("0.0.0.0:1729");
```

[tab:end]

[tab:Python]

<!-- test-ignore -->
```python
client = TypeDB.core_client("0.0.0.0:1729")
```

[tab:end]
</div>

NOTE: To connect to TypeDB Cloud use `clusterClient`/`cluster_client` instead of `coreClient`/`core_client`. 
It requires a second argument of `TypeDBCredential` type.

## Databases

TypeDB instances can contain multiple databases. A database consists of a [schema](../02-dev/02-schema.md) and 
[data](../02-dev/04-write.md) – and is both separate and independent of any other database. It is not possible for two
databases to influence each other. However, clients can connect to multiple databases simultaneously.

<div class="note">
[Note]
TypeDB is optimized for a small number of databases. It’s recommended to start with a single database, and add more as 
necessary (e.g., to support more applications). The **best practice** is to keep the number of databases **under 10**.
</div>

<div class="tabs dark">

[tab:TypeDB Console]

<!-- test-ignore -->
```bash
# create database
database create test-db

# get database schema
database schema test-db

# list all databases
database list

# delete database
database delete test-db
```

[tab:end]

[tab:Java]

<!-- test-ignore -->
```java
// create database
client.databases().create("test-db");

// get database schema
client.databases().get("test-db").schema();

// get all databases
client.databases().all();

// check if database exists
client.databases().contains("test-db");

// delete database
client.databases().get("test-db").delete();
```
        
[tab:end]

[tab:Node.js]

<!-- test-ignore -->
```javascript
// create database
await client.databases().create("test-db");

// get database schema
await client.databases().get("test-db").schema();

// get all databases
await client.databases().all();

// check if database exists
await client.databases().contains("test-db");

// delete database
await (await client.databases().get("test-db")).delete();
```

[tab:end]

[tab:Python]

<!-- test-ignore -->
```python
# create database
client.databases().create("test-db")

# get database schema
client.databases().get("test-db").schema()

# get all databases
client.databases().all()

# check if database exists
client.databases().contains("test-db")

# delete database
client.databases().get("test-db").delete()
```

[tab:end]
</div>

## Sessions

There are two types of sessions: 

- SCHEMA sessions,
- DATA sessions. 

| Session type | Read data |  Write data  | Read schema |  Write schema   |
|:------------:|:---------:|:------------:|:-----------:|:---------------:|
|     DATA     |    Yes    |     Yes      |     Yes     |     **No**      |
|    SCHEMA    |    Yes    |    **No**    |     Yes     |       Yes       |

TypeDB clients should read and write data in DATA sessions.

TypeDB clients should read and write schema in SCHEMA sessions.

<div class="note">
[Note]
If a client needs to read both schema and data from a database, it can be done in any session type (usually used 
when data query needs information on types). But it is NOT possible to modify a schema and its data in the same session, 
regardless of the type. Write transactions are strict to the session types.
</div>

Once a session has been opened, clients can open and close transactions to read and write a database’s schema or data.

<div class="tabs dark">

[tab:TypeDB Console]

<!-- test-ignore -->
```
transaction iam data read
```

[tab:end]

[tab:Java]

<!-- test-ignore -->
```java
TypeDBSession session = client.session("iam", TypeDBSession.Type.DATA);
```

[tab:end]

[tab:Node.js]

<!-- test-ignore -->
```javascript
session = await client.session("iam", SessionType.DATA);
```

[tab:end]

[tab:Python]

<!-- test-ignore -->
```python
session = client.session("iam", SessionType.DATA)
```

[tab:end]
</div>

<div class="note">
[Note]
It is recommended to avoid long-running sessions, because of possible network failures.
</div>

A good principle to follow for logically coherent transactions to be grouped together into a session.

## Transactions

All queries to a TypeDB database are performed through transactions. TypeDB transactions provide full 
[ACID guarantees](#acid-guarantees) up to [snapshot isolation](#isolation).

There are two types of transactions: 

- READ transactions
- WRITE transactions

<div class="note">
[Note]
Transactions must be explicitly opened and closed by clients via [API](08-api.md).
</div>

<div class="tabs dark">

[tab:TypeDB Console]

<!-- test-ignore -->
```typeql
# start transaction
transaction iam data write

insert $x isa person;
$x has full-name "Kevin";
$x has email "Kevin@vaticle.com";

# commit changes
commit
```

[tab:end]

[tab:Java]

<!-- test-ignore -->
```java
// start transaction
TypeDBTransaction Transaction = session.transaction(TypeDBTransaction.Type.WRITE);

Transaction.query().insert(insertQuery1);
Transaction.query().insert(insertQuery2);

Transaction.query().insert(insertQueryN);
// commit changes
Transaction.commit();
```

[tab:end]

[tab:Node.js]

<!-- test-ignore -->
```javascript
// start transaction
const transaction = await session.transaction(TransactionType.WRITE);
transaction.query().insert(InsertQuery1);
transaction.query().insert(InsertQuery2);

transaction.query().insert(InsertQueryN);
// commit changes
transaction.commit();
```

[tab:end]

[tab:Python]

<!-- test-ignore -->
```python
# start transaction
transaction = session.transaction(TransactionType.WRITE)
transaction.query().insert(insert_query1)
transaction.query().insert(insert_query2)

transaction.query().insert(insert_queryN)
# commit changes
    transaction.commit()
```
[tab:end]
</div>


<div class="note">
[Note]
TypeDB Studio lets developers commit/rollback transactions through its GUI.
</div>

TypeDB transactions use snapshot isolation and optimistic concurrency control to support concurrent, lock-free 
read/write transactions. 

### Configuration

TypeDB transactions will **time out** after a set period of time, **5 minutes by default**. However, the timeout can 
be changed via a client option. The timeout is intended to encourage short-lived transactions, prevent memory leaks 
caused by transactions which will not be completed and terminate unresponsive transactions.

### Best practices

- Avoid long-running transactions which can result in conflicts and resource contention.
- A good principle to follow is that transactions group logically coherent queries.

## ACID guarantees

### Atomicity

TypeDB transactions are all or nothing. If a commit succeeds, all of its changes are persisted. If it fails, all of its 
changes will be rolled back.

### Consistency

TypeDB validates all changes to data and schemas. If changes to a database violate schema or data constraints, the 
transaction will fail and be rolled back.

### Isolation

TypeDB transactions use snapshot isolation and optimistic concurrency control to support simultaneous, lock-free 
read/write transactions. Thus, a transaction operates on its own snapshot of the data, independent of any other. All 
of its changes are hidden from other transactions. However, they will become visible immediately after a successful 
commit. 

If two transactions attempt to modify the same data, one will succeed on commit while the other will fail. However, 
one transaction can read data while another is writing it.

### Durability

TypeDB writes transactions to a write-ahead log upon commit, ensuring they can be recovered if an unexpected failure 
(e.g., power outage) occurs before the data is modified.

<div class="note">
[Note]
TypeDB durability guarantees do not apply when storage devices become corrupt or damaged.
</div>

Successful write transactions are written to the write-ahead log before returning a response to the client. If a 
transaction is not successful, all changes are rolled back.

For cluster installations of TypeDB Cloud transaction acknowledgement is sent to the client after majority of replicas 
replicated the transaction results. See [Replication](../03-admin/04-ha.md) for details.