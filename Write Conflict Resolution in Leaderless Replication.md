# Conflict Resolution Techniques in Leaderless Replication

Leaderless replication enables any node to accept writes, which can lead to concurrent updates on the same data. To resolve conflicts, various strategies are used. Below is a detailed summary.

| Technique                | Description                                                                                         | Pros                                                    | Cons                                                    | Example                                                   | Real-World Usage                                  |
|--------------------------|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------|-----------------------------------------------------------|------------------------------------------------------------|--------------------------------------------------|
| **Last Write Wins (LWW)** | Chooses the write with the latest timestamp.                                                       | Simple, fast, deterministic                              | Relies on synchronized clocks, may lose data              | `User(name=Alice, ts=1000)` vs `User(name=Bob, ts=1010)` → keeps Bob | Cassandra, Amazon S3 (versioning)                |
| **Hybrid Logical Clock (HLC)** | Logical + physical clock hybrid. Helps maintain causality with fewer collisions than LWW.           | Improves ordering over LWW, better causality guarantees | Still can lose writes in rare race conditions             | `(User, ts=100, lc=7)` is newer than `(User, ts=100, lc=5)` | CockroachDB, YugabyteDB                          |
| **Vector Clock (VC)**     | Maintains per-replica counters to track causality and concurrency.                                 | Detects concurrent updates, avoids data loss             | Metadata overhead, complex conflict resolution            | A(5,0,0), B(0,5,0), C(0,0,5) → concurrent                   | Amazon Dynamo, Riak                              |
| **Siblings + Manual Resolution** | Keeps all conflicting versions (siblings) and lets app/user resolve.                               | No automatic data loss                                   | Requires application logic to merge                      | `{User: Alice}`, `{User: Bob}` → app merges → `Alice Bob`  | Riak (siblings model)                            |
| **CRDT (Conflict-Free Replicated Data Types)** | Data structures that automatically converge without coordination.                          | Strong eventual consistency, automatic reconciliation    | Limited to specific use-cases (sets, counters, maps, etc.)| G-Counter: replicas increment → values summed               | Riak, Redis CRDTs (via modules), AntidoteDB       |
| **Application-Level Merge** | Custom merge logic at application layer, often using domain knowledge.                           | Highly flexible                                           | Hard to generalize, needs app changes                     | Merge two cart objects by summing quantities               | Shopping carts, user profiles in social apps     |
| **Operational Transformation / OT** | Maintains and replays operation history, adjusting based on order.                            | Fine-grained conflict resolution                         | High complexity                                           | Concurrent edits: Insert "a", delete "b" → final result     | Google Docs (real-time collaborative editors)    |
| **Custom Conflict Policies** | DB supports custom conflict resolution logic or plugins.                                          | Domain-specific logic                                    | Needs developer effort, harder to test                    | “Prefer update from trusted region”                        | CouchDB (custom conflict handler)                |

---

## ✅ Key Considerations

- **Clock Synchronization**: LWW/HLC depends on clock sync. NTP drift can cause issues.
- **Storage Overhead**: Vector Clocks and Siblings introduce metadata.
- **App Complexity**: CRDTs simplify system logic but require using specialized types.

---

## 🔧 Use Case Mapping

| Use Case                        | Recommended Technique        |
|--------------------------------|------------------------------|
| Simple key-value cache         | LWW                          |
| Distributed counters           | CRDT (G-Counter, PN-Counter)|
| Shopping carts                 | CRDT / App-level merge       |
| Collaborative editors          | OT / CRDT                    |
| Complex object replication     | Vector Clocks + App merge    |
| IoT data collection            | HLC                          |

---

## 📝 Notes

- **HLC** enhances LWW without requiring strict clock sync.
- **Vector Clocks** help detect conflict but need resolution logic.
- **CRDTs** work great for converging data without coordination.

---

# Conflict Resolution in Leaderless Replication

This document summarizes common interview questions and answers related to conflict resolution strategies in leaderless replication setups, such as those used in DynamoDB and Cassandra.

---

## ❓ Interview Questions and Answers

### 1. **What is conflict resolution in leaderless replication?**
Conflict resolution refers to the process of resolving discrepancies that arise when concurrent writes happen across different replicas in a distributed system with no single leader.

---

### 2. **What are the common techniques used to resolve write conflicts?**
- Last Write Wins (LWW)
- Hybrid Logical Clocks (HLC)
- Vector Clocks (VC)
- Client-side Reconciliation
- Read Repair
- Siblings and Merkle Trees
- CRDTs (Conflict-Free Replicated Data Types)

---

### 3. **How does Last Write Wins (LWW) work?**
- Each write is assigned a timestamp (usually epoch time).
- The write with the highest timestamp wins.
- Simpler but prone to data loss under concurrent updates.

**Example:**  
Replica A → (value: "Alice", timestamp: 1000)  
Replica B → (value: "Bob", timestamp: 999)  
Merged → "Alice"

---

### 4. **What are the limitations of LWW?**
- Risk of overwriting valid updates.
- Timestamp collisions can cause non-deterministic outcomes.
- Requires loosely synchronized clocks or hybrid logical clocks.

---

### 5. **What is Hybrid Logical Clock (HLC)?**
- A combination of physical and logical clocks.
- Each write has a (timestamp, counter) tuple.
- Ensures monotonicity without perfect clock sync.

**Example:**  
A(1000, 5), B(1000, 7) → B wins.

---

### 6. **What happens if HLC timestamps match?**
- Use the counter to break ties.
- If still same, fall back to node ID or deterministic hash for tie-breaking.

---

### 7. **What are Vector Clocks (VC)?**
- Each object keeps a vector of versions per replica.
- Allows the system to detect concurrent writes.

**Example:**

```json
{
  "value": "Alice",
  "vc": { "A": 3, "B": 1 }
}
```

Another version:

```json
{
  "value": "Bob",
  "vc": { "A": 1, "B": 3 }
}
```

These two are concurrent and need reconciliation.

---

### 8. **What happens when two vector clocks are concurrent?**
- System cannot decide automatically which is newer.
- Returns both versions to client for reconciliation (like in Dynamo).
- Client writes back merged data.

---

### 9. **What is Read Repair?**
- Upon a read, the client compares versions across replicas.
- If discrepancies exist, repairs the outdated replicas with the correct data.
- Happens in the background.

---

### 10. **What is hinted handoff?**
- If a replica is down, another replica stores the write on its behalf.
- Once the downed replica comes back, the hint is replayed.

---

### 11. **What are CRDTs and when are they used?**
- CRDTs are data structures that automatically merge without conflicts.
- Used when high availability and automatic resolution is needed.
- Ideal for counters, sets, maps.

**Examples:**
- G-Counter, PN-Counter
- OR-Set, LWW-Element-Set

---

### 12. **Which databases use these techniques?**
| Database       | Conflict Resolution Techniques       |
|----------------|--------------------------------------|
| DynamoDB       | Vector Clocks, Client-side Merge     |
| Cassandra      | LWW, HLC, Read Repair                |
| Riak           | Vector Clocks, Siblings, CRDTs       |
| CouchDB        | MVCC + Application-level merge       |
| Redis (Cluster)| No built-in CR, relies on app logic  |

---

### 13. **What are the trade-offs of each strategy?**

| Strategy       | Pros                          | Cons                              |
|----------------|-------------------------------|-----------------------------------|
| LWW            | Simple, fast                  | Can lose writes                   |
| HLC            | Handles clock skew            | Needs HLC logic everywhere        |
| VC             | Detects concurrency           | Requires metadata, client logic  |
| CRDTs          | Automatic merge               | Not suitable for all data types  |
| Read Repair    | Eventually consistent         | May have stale reads initially   |

---

### 14. **When would you use client-side conflict resolution?**
- When automatic conflict resolution is not safe or feasible.
- For example: user profile updates where merging needs domain knowledge.

---

## ✅ Summary

Leaderless replication gives availability and fault tolerance, but requires careful conflict resolution. Different strategies are chosen based on use case, consistency needs, and tolerance for complexity.

---


