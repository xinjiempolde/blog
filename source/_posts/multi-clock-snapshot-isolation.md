---
title: multi-clock snapshot isolation
date: 2023-06-07 11:07:01
tags:
categories:
    - paper
---

# 

# Motivatione

a previous single-layer NVM database N2DB[14] used the snapshot isolation from PostgreSQL[17]. Its snapshot generation process involves taking a lock and scanning a global active transaction list, which blocks concurrent transactions and is not scalable



# Background

## 2.3 Concurrency control for NVM databases

The snapshot isolation concurrency control implementation used in traditional MVCC databases is not well suited for NVM because of two reasons. (1) The concurrency control implementations of traditional databases mainly focus on their in-memory part without discussing durability and crash consistency. They leave those to another dedicated recovery protocol, such as ARIES or write-ahead logging. The situation is different in NVM databases because a single copy of data is used for both runtime access and durability. Ensuring crash consistency should unavoidably be part of the concurrency control itself. (2) Concurrency control methods from a two-layer architecture have suboptimal performance when directly applied to a single-layer NVM database. FOEDUS[28] points out that centralized components, such as lock manager and logs in traditional concurrency control implementations, will cause performance bottlenecks in a multicore NVM database. A previous NVM database N2DB[14] adopted the concurrency control from PostgreSQL[17] to implement snapshot isolation. The concurrency control scheme from PostgreSQL uses the currently completed transactions in the system to represent a snapshot. When generating a snapshot, it has to take a lock and scan the global active transaction list. This approach blocks concurrent transactions and is not scalable. N2DB also adopted the commit log in NVM to persistently store transaction status for crash consistency. The commit log is an array-like structure and is write-shared by all threads, which limits its scalability. Zen[29] also points out that traditional concurrency control methods often need to modify the tuple metadata not only by tuple writes but also by tuple reads, which incurs expensive NVM writes.



# Multi-Clock Concurrency Control
