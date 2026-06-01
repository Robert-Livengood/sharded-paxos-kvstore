# Single-Key Request Sequence

```mermaid
sequenceDiagram
    participant C as Client
    participant SM as ShardMaster
    participant G1 as Replica Group 1
    participant P as Paxos
    participant KV as KVStore

    autonumber

    C->>SM: Query config for key
    SM-->>C: Shard owned by Group 1
    C->>G1: Put or Get request
    G1->>P: Propose operation
    P-->>G1: Operation chosen
    G1->>KV: Apply operation
    KV-->>G1: Result
    G1-->>C: Reply
```