# Sharded Transactional Key-Value Store using Paxos

## Overview

This project documents the architecture and design of a sharded, transactional key-value store built using Paxos replication. The system was developed as part of my graduate distributed systems coursework and is presented here as a design focused portfolio project without publishing the private course solution code.

The system supports replicated state machines, shard partitioning, dynamic reconfiguration, and multikey transactions across shard groups. It was designed to ensure correctness under unreliable networks, duplicate messages, node failures, client retries, and configuration changes.

The system was implemented and tested using the DSLabs framework, which provides deterministic simulation, unreliable network testing, and model checking support. Development and testing were performed in a Docker-based Ubuntu 24.04 environment.

## Core Concepts

A distributed database or storage system must balance scalability, fault tolerance, consistency, and operational complexity. This project demonstrates this using several core distributed systems concepts:

* Paxos consensus
* Replicated state machines
* Sharding and partitioning
* Dynamic shard reconfiguration
* Multikey distributed transactions
* At-most-once client semantics
* Fault tolerance under unreliable networks

The goal was to build a system that behaves like a simplified distributed database: data is partitioned across replica groups, each group uses consensus to remain fault tolerant, and transactions can span multiple shards while preserving correctness.

## System Architecture

The system contains four major components:

1. Clients
2. ShardMaster
3. Shard replica groups
4. Paxos replication layer

```text
Client: Query current configuration
  |
  v
ShardMaster: Returns shard ownership map
  |
  v
Client: Sends operation to owning shard group
  |
  v
Shard Group: Replicates operation through Paxos
  |
  v
KVStore
```

### Clients

Clients query the ShardMaster to determine which replica group owns the shard for a given key. They then route read, write, and transaction requests to the appropriate group. Clients retry requests when messages are lost, servers are unavailable, or configurations change.

### ShardMaster

The ShardMaster maintains the global configuration of the system. A configuration maps shards to replica groups. It supports operations such as:

* Join:   add a new replica group
* Leave:  remove an existing replica group
* Move:   manually move a shard to a group
* Query:  retrieve the current or historical configuration

### Replica Groups

Each replica group owns a subset of shards. A group consists of one or more replicated servers that use Paxos to agree on the order of operations. This ensures that all replicas within a group apply commands deterministically and maintain equivalent state.

### Paxos Layer

The Paxos layer provides consensus within each replica group. Operations such as client commands, shard transfers, configuration updates, prepare messages, commits, and aborts are proposed through the replicated log so that all replicas observe the same sequence of state transitions.

## Core Features

### Paxos Replication

Each shard group is replicated using Paxos. Client commands are ordered through consensus before being applied to the key-value store. This provides fault tolerance and prevents replicas from diverging under unreliable message delivery.

### Sharding

Keys are mapped to shards, and shards are assigned to replica groups. Sharding allows the system to divide responsibility across groups, improving scalability compared with a single replicated state machine.

### Dynamic Reconfiguration

The system supports configuration changes as groups join, leave, or receive moved shards. During reconfiguration, shards must be transferred to their new owners without violating correctness or allowing two groups to serve the same shard inconsistently.

### Multikey Transactions

The system supports transactions that may access keys across multiple shards. Transactions use a coordinator/participant flow to prepare, commit, or abort operations across the relevant shard groups.

### At-Most-Once Semantics

Client commands include identifiers that allow servers to detect and suppress duplicate execution. This is necessary because clients may retry requests when messages are delayed, dropped, or when ownership changes.

## Design Emphasis

The implementation prioritized correctness under failure over raw throughput. Many design decisions were driven by the need to preserve linearizability, avoid duplicate command execution, and maintain safe shard ownership during configuration changes.

The hardest engineering challenge was coordinating three interacting concerns at once:

* Paxos replication within each shard group
* shard movement across configurations
* multi-key transactions spanning multiple shard groups

## Transaction Flow

A multi-shard transaction proceeds conceptually as follows:

```text
Client
  |
  | Transaction request
  v
Coordinator Group
  |
  | Prepare
  v
Participant Groups
  |
  | Prepare OK / Abort
  v
Coordinator Group
  |
  | Commit / Abort decision
  v
Participant Groups
  |
  | Apply or release locks
  v
Client Response
```

The coordinator determines which shards participate in the transaction, sends prepare requests, collects responses, and then decides whether to commit or abort. Participants must ensure that prepared state is handled safely during retries, failures, and configuration changes.

## Shard Reconfiguration Flow

Reconfiguration requires careful coordination between old and new shard owners.

```text
Starting Configuration (N)
  |
  | Shard ownership changes
  v
Old Owner locks and transfers shard state
  |
  | Shard data sent to new owner
  v
New Owner installs shard
  |
  | Configuration N+1 becomes active
  v
New Owner serves shard
```

The key correctness requirement is that at most one group can serve a shard for a given configuration, and shard state must not be lost or duplicated during transfer.

## Correctness Challenges

The most difficult parts of the system were:

* Preventing stale owners from serving moved shards
* Coordinating transaction state during configuration changes
* Preserving at-most-once execution across retries
* Avoiding duplicate client replies from non-authoritative replicas
* Handling unreliable message delivery without excessive retry churn
* Keeping model checking state space manageable
* Ensuring correctness in unreliable networks

## Testing

The implementation was evaluated using deterministic and unreliable network tests under the DSLabs framework. Test scenarios included:

* Single key operations
* Multikey transactions
* Repeated shard movement
* Group joins and leaves
* Unreliable message delivery
* Duplicate requests
* Node failures
* Search/model-checking scenarios

The implementation passes nearly all local functional test scenarios. Some resource constrained container runs and exhaustive model checking scenarios expose performance and state-space limitations, especially around retry behavior, Paxos coordination, and distributed transaction complexity.

## Known Limitations

This public repository does not include private course solution code. It documents the architecture, design decisions, and lessons learned from the implementation.

Known engineering limitations of the implementation include:

* The system was designed for an academic model checking framework rather than production deployment.
* Some exhaustive search tests can suffer from state space explosion due to retry timers and distributed transaction state.
* Persistence and disk recovery are not implemented.
* Production observability, metrics, tracing, and operational tooling are not implemented.
* The implementation would benefit from further refactoring around transaction coordination, reconfiguration handling, retry behavior, and Paxos layer optimization.

## Future Improvements

Planned improvements for a non-coursework, production inspired implementation:

* Build a standalone implementation outside the DSLabs framework
* Add TCP or gRPC networking
* Implement persistent logs and crash recovery
* Add snapshotting and log compaction
* Add benchmarking for throughput and latency under different shard counts
* Add structured logging and metrics
* Simplify retry behavior to reduce unnecessary message churn
* Add integration tests for failure and reconfiguration scenarios

## What I Learned

This project reinforced several important distributed systems lessons:

* Correctness is more difficult than nominal case functionality.
* Retry logic can preserve liveness but also dramatically increase state space complexity.
* Reconfiguration and transactions interact in subtle ways. For example, when to defer reconfiguration or when to abort transactions.
* Deterministic replicated state machines require careful control over who is allowed to reply to clients.
* Distributed systems need clear ownership boundaries, especially during configuration changes.

## Status

This repository is currently a design and architecture portfolio artifact. A separate clean-room implementation may be added in the future.
