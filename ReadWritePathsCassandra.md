# Apache Cassandra - Read and Write Paths

---

## ✍️ Write Path (Insert/Update/Delete)

### 🛤️ Steps:

1. **Client sends write to Coordinator node**
   - Any node in the cluster can act as the coordinator.
   - It does **not need to be the replica node**.

2. **Coordinator hashes partition key**
   - Determines target replicas using **partitioner (e.g., Murmur3)**.
   - Based on **replication factor (RF)**, finds N replicas.

3. **Coordinator sends write to all replicas**
   - Every replica:
     - Appends to the **Commit Log** (for durability).
     - Applies to the **Memtable** (in-memory write buffer).

4. **If replica is down → Hinted Handoff**
   - Coordinator stores a **hint** to send later when node comes back.

5. **Consistency level check**
   - Coordinator waits for acknowledgments from replicas based on **CL**:
     - `ONE`, `QUORUM`, `ALL`, etc.
   - Responds to the client upon success.

6. **Flush to SSTable (on threshold)**
   - Memtable flushed to disk → creates immutable **SSTable**.
   - Corresponding Commit Log segment is deleted.

7. **Background Compaction**
   - Multiple SSTables are periodically **compacted**.
   - Merges rows, discards tombstones, and removes duplicates.

---

## 📖 Read Path (SELECT query)

### 🛤️ Steps:

1. **Client sends query to Coordinator**
   - Coordinator hashes the partition key.
   - Identifies the replicas.

2. **Coordinator sends read request to replicas**
   - Typically from **closest replicas** (based on snitch).
   - Chooses enough replicas to meet the **read CL**.

3. **Each replica performs local read**
   - **Memtable**: Checked first (latest in-memory writes).
   - **Partition Key Cache**: If enabled, fast lookup.
   - **Bloom Filter**: Quickly rules out SSTables that don’t contain the key.
   - **Index Summary**: Approximates location in SSTable.
   - **Partition Index**: Locates exact offset in Data file.
   - **SSTable Data file**: Reads and returns result.

4. **Coordinator merges data**
   - Reconciles responses based on **timestamp**.
   - If differences exist, triggers **read repair**.

5. **Returns result to client**

---

## 🧠 Supporting Components

| Component         | Used In       | Purpose                                                                 |
|------------------|---------------|-------------------------------------------------------------------------|
| Commit Log        | Write Path    | Durable write-ahead log                                                |
| Memtable          | Read/Write    | In-memory structure for fast write and read                            |
| SSTable           | Read/Write    | Immutable on-disk table after memtable flush                           |
| Bloom Filter      | Read Path     | Avoids unnecessary SSTable disk lookups                                |
| Index Summary     | Read Path     | In-memory sparse index to narrow down location in SSTable              |
| Partition Index   | Read Path     | On-disk index to locate exact position in Data file                    |
| Hinted Handoff    | Write Path    | Temporary storage of writes for unreachable nodes                      |
| Tombstones        | Read/Write    | Marker for deletes, removed during compaction                          |
| Compaction        | Write Path    | Merges SSTables, clears tombstones, improves read efficiency           |

---

## 🔁 Consistency Levels

| Level     | Required Acks (RF = 3) | Description                          |
|-----------|------------------------|--------------------------------------|
| ONE       | 1                      | Fast, less durable                   |
| QUORUM    | 2                      | Balance of performance and accuracy |
| ALL       | 3                      | Strongest consistency, slowest      |

---

## 🔄 Quorum Rule

To ensure strong consistency, read + write consistency levels must overlap.

Example:
- RF = 3
- W = QUORUM (2)
- R = QUORUM (2) ⇒ 2 + 2 > 3 ✔️

---


