---
layout: page
title: Distributed Raft Consensus System
description:
img: /assets/img/proj1_cover.jpg
importance: 1
category: work
related_publications: false
---

# Project Report: Distributed Raft Consensus System

## Executive Summary

**My Role:** Engine Integration & System Deployment
**Tech Stack:** C++, RocksDB, Redis Protocol, Linux
**Key Achievement:** Integrated **RocksDB** into a Raft-based distributed framework to achieve data persistence; verified system consistency and fault tolerance through simulated node crashes.

## System Architecture

The project implements a distributed Key-Value store based on the **Raft Consensus Algorithm**. It exposes a **Redis-compatible interface** for client interaction and leverages **RocksDB** as the storage backend to ensure durability.

## Source Code Analysis

#### Core Code Structure

The project resides in the `raft-kv/` directory and is modularized into four key layers:

1. **`raft/` (Consensus Layer):**
   `raft.cpp`: This is the core implementation of the Raft state machine (Election, Log replication, Heartbeats).
   `node.cpp`: This cpp file acts as the bridge between the upper-layer application and the Raft state machine, handling Propose and Ready states.
   `raft_log.cpp`: It manages the append-only log entries.

2. **`server/` (Logic & Application Layer):**
   `raft_node.cpp`: This is the soul of the project. It arranges all components, and it connects the Raft state machine, network transport, and the KV engine.
   `redis_store.cpp`: This is the actual KV storage engine. _(My integration point for RocksDB)._
   `redis_session.cpp`: This file handles Redis protocol parsing, allowing interaction via standard `redis-cli`.

3. **`wal/` & `snap/` (Persistence Layer):**
   `wal.cpp`: It manages the Write-Ahead Log to ensure crash recovery.
   `snapshotter.cpp`: It handles data snapshotting to prevent the log from growing indefinitely.

#### The Lifecycle of a Write Request

The system follows the classic "Propose -> Consensus -> Commit -> Apply" flow. I analyzed the specific call stack for a `SET key value` command:

1. **Request Parsing:**
   The `RedisSession` receives the `SET` command from a client and invokes `RedisStore::set`.
2. **Proposal Submission:**
   `RedisStore` serializes the write operation into a `RaftCommit` packet. It calls `RaftNode::propose` -> `node_->propose(data)`, effectively pushing the data into the local Raft log.
3. **Log Replication & Consensus:**
   The `RaftNode` timer triggers `pull_ready_events`, and the `Transport` layer broadcasts the new log entries to Follower nodes. Once the majority acknowledges the log, the state machine marks the entry as `committed`.
4. **Application (The "Ready" State):**
   In the next `pull_ready_events` cycle, the node detects `committed_entries`. `RaftNode::publish_entries` iterates through the committed logs.

**Persistence (My Contribution):** The system calls `redis_server_->read_commit(entry)`. This is where I modified the logic to flush the data into RocksDB via `WriteBatch`, ensuring atomicity.

#### Read Request Strategy (`GET`)

For performance optimization, Read requests (`GET` key) bypass the Raft log proposal flow.
The `RedisStore::get` function retrieves data directly from the local storage engine.

## Implementation Process

1. **Cluster Simulation:**
   Deployed the Raft cluster using `goreman` to manage multiple process instances, simulating a distributed environment on a Linux server.

2. **RocksDB Integration:**
   Get RocksDB source code on GitHub and compile.

- I modified `redis_store.h` to include RocksDB headers and define the database pointer `db_`
- Then implemented `RedisStore::set` to map Redis commands to `rocksdb::DB::Put`.
- Also implemented `RedisStore::get` to map to `rocksdb::DB::Get`.
- Finally **snapshot recovery**: rewote `get_snapshot` to use `rocksdb::Iterator`, ensuring that when a node restarts, it can rebuild its state machine from the RocksDB SST files.

3. **Benchmarking:**
   I used standard benchmark tools to stress test the cluster's read/write performance and verify leader election latencies.

## Results & Validation

Functional Validation:

1. Leader Election: Verified that killing the Leader process triggers a new election within the timeout period.
2. Data Consistency: Verified that data written to the Leader is successfully replicated to Followers and persisted in RocksDB even after a cluster restart.

**Performance Data:**
![](/assets/img/raft01.jpg)

![](/assets/img/raft02.jpg)

![](/assets/img/raft03.png)

## Summary

This project bridged the gap between theoretical consensus algorithms and practical storage engineering. By integrating RocksDB into the Raft framework, I tackled challenges in dependency management (CMake/C++ Linkers) and learned how to implement Fault Tolerance in distributed systems.

**References:**

1.  Ongaro, D. and Ousterhout, J., 2014. In search of an understandable consensus algorithm. _USENIX ATC 14_.
2.  [RocksDB Official Documentation](https://rocksdb.org)
3.  [Redis Protocol Specification](https://redis.io/topics/protocol)
4.  [Goreman Process Manager](https://github.com/mattn/goreman)
