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

