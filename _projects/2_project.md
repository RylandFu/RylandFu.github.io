---
layout: page
title: Message Queue Project Report
description: 
img: /assets/img/proj1_cover.jpg
importance: 1
category: work
related_publications: false
---

> Internal Analysis & Performance Stress Testing of a C++ Middleware

## Executive Summary
My job in this project is system evaluation and deployment. I've benchmarked this lightweight MQ system reaching 520k TPS.

__Tech Stack:__ C++, Linux(Ubuntu), ZeroMQ

## System Architecture
This system implements a classic __Producer-Consumer model__ decoupled by a centralized __broker__ .
I need to explain some concepts before talking about other things of this project :
__Broker__ manages memory-resident message queues and handles topic distribution.
__Producer__ asynchronously pushes messages to specific topics.
__Consumer__ subscribes to topics and pulls messages using blocking or non-blocking retrieval.

## Source Code Analysis
To understand the system's performance limits, I conducted a deep-dive analysis of the core C++ source code, focusing on Thread Safety and Synchronization Mechanisms.

Unlike typical academic implementations that rely on coarse-grained `std::mutex`, my code review revealed that this system achieves high throughput through **Lock-free Queues**, **Asynchronous I/O**, and **Zero-Copy Memory Management**.

#### Core Composition (Header-Only Design)
The project adopts a **Header-only** design pattern, heavily integrating logic within the `include/` directory for compiler optimization.

* **`broker_manager.h`:** Acts as the central command unit. It maintains a `std::map<std::string, broker*>` to map Topic strings to specific Broker instances, handling management commands like `create_topic` or `stats`.
* **`broker_storage.h`:** Defines how messages are buffered and persisted. It supports three distinct modes:
    * **Transient:** Pure in-memory (fastest).
    * **Durable:** File-based storage.
    * **Hybrid:** Asynchronous persistence (Memory + Disk).
* **`connection.h`:** Abstracts the underlying transport, deriving `connection_zmq` (ZeroMQ) and `connection_file` to provide a unified I/O interface.

#### Concurrency Control Strategy
The secret to the system's 500k+ TPS lies in minimizing lock contention and decoupling threads.

* **Lock-free Queue Integration:** In `broker_storage.h`, the system utilizes `moodycamel::ReaderWriterQueue` (a third-party SPSC library). This allows the **Producer Thread** to push data and the **Consumer Thread** to poll data without fighting for a `std::mutex`, significantly reducing context switching overhead compared to standard locking mechanisms.

* **Asynchronous Persistence:** When running in `broker_queue_file` mode, the system spawns a dedicated background thread: `queue_to_file_thread_`. This thread is responsible for flushing in-memory messages to disk. This design ensures that the **critical message path (Send/Recv)** is never blocked by slow Disk I/O latency.

#### Memory Management Optimization
The code demonstrates a "Performance First" philosophy in handling resources.

* **Pre-allocated Buffers:** The code declares `char buffer_[utils::max_msg_size]`. By pre-allocating large fixed-size memory blocks for Read/Write operations, the system avoids the overhead of frequent `malloc/free` calls and reduces memory fragmentation.
    
* **Move Semantics:** Extensive use of `std::move(msg)` (e.g., in `add_to_storage`) ensures that message objects are **transferred** into the queue rather than logically **copied**. This avoids expensive deep-copy operations for string payloads.
    
* **RAII & Lifecycle Management:** Following C++ RAII (Resource Acquisition Is Initialization) principles, thread resources (like `broker_storage` background threads) are properly managed. The destructors explicitly call `join()` to ensure threads are cleanly reclaimed, preventing zombie processes or memory leaks.

## Benchmarking Methodology 
Here's a comprehensive stress test suit using Shell scripts to simulate varying load condition on an Ubuntu cluster

Test scenarios:
- Baseline Throughput: Single Producer / Single Consumer (1-to-1).
- Concurrency Stress: Multi-Producer (N-to-1) to test lock contention.
- Bandwidth Saturation: Large Payload (1KB - 10MB) testing.
- Partition Routing: Verifying logic correctness for multi-partition topics.

## Stress Test Process & Results
1. Installing dependencies, building projects and generates excecutable file using make.

2. Create a topic and start a broker. Then start a producer and a consumer. Making sure everything is functioning normally.

3. Stress test process：
**High-Volume Throughput Test**
**Objective:** Validate system stability under a massive stream of messages.
First we introduceConfigure consumer to receive **200M** messages，and configure producer to send **300M** messages:  

![mq_test01](/assets/img/mq_test01.png)  

As shown in mq_test01, the system maintained high throughput without memory leaks or crashes, confirming baseline stability.

**Concurrency & Contention Test**
**Concurrency & Contention Test**
**Objective:** Evaluate thread safety and lock contention overhead.
Then I scaled up to **2 concurrent producers** and **2 concurrent consumers** running simultaneously.  

![mq_test02](/assets/img/mq_test02.png)  

![mq_test03](/assets/img/mq_test03.png)  

Result: As shown in mq_test02 and mq_test03, the data flow remained consistent. The system successfully handled the context switching, though total throughput reflected the expected overhead from mutex locking.

**Bandwidth Saturation**
**Objective:** Test memory copy efficiency and bandwidth limits.
increased individual message size to 1KB using the -s 1024 flag.  

![mq_test04](/assets/img/mq_test04.png)

**Topic & Partition Management**
**Objective:** Verify the logical separation and routing of messages.
I create a new Topic configured with 3 partitions and targeted specific partitions for data injection and retrieval.  

![mq_test05](/assets/img/mq_test05.png)  

Result: As shown in mq_test05, messages were correctly routed to their assigned partitions, validating the flexibility of the storage engine.

### Summary
Through deep code analysis and stress testing, I verified that the system's high performance stems from its **Lock-free SPSC design** and **Asynchronous I/O**.