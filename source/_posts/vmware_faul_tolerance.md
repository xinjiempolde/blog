---
title: vmware fault tolerance
date: 2022-06-31 18:41:25
tags:
    - fault_tolerance
    - MIT6.824
categories:
    - paper

---

# failure types

Fail-stop failure(i.e. machine crashes, os kernel panic), not including software or hardware bugs.

<!--more-->

# two types of primary-backup replication

- **State Transfer**: you should transfer all the state of the primary(such as the content of RAM) via Internet to the backup
- **Replicated State Machine**: primary just sends the external inputs to the backup. The different states between primary and backup are all caused by external inputs.

Replicated State Machine is a good idea, but there are some non-determined instructions(such as get current time or get random number). So we must handle the non-determined instructions.

VMware-Fault-Tolerance only deals with **single-core** machine rather than multi-core machine because there will be interleavings of the instructions from two cores.  



# problems

## what states to be replicated?

VMware-Fault-Tolerance replicates states at machine level. It will transfer all the states such as the content of RAMs, registers and so on. It doesn't care what software you run on it, which is unique compared to other fault tolerances.



## non-determined events

- Inputs = network packet = data + interrupt(no-determined)
- Weird instructions(such as getting processor id)
- Multiple-cores



# log entries

requirements:

- Log entry number
- the type of log entry(determine or non-determine)
- Data



# output rule

**Problem 1**:If the primary has generated outputs to the client but crashed before sending records to the backup, the status of the backup is not correct. Output rule is to deal with the problem. Below is the output rule:

The primary is not allowed to generate outputs to the client until the backup acknowledges that it has received all log records.


**Problem 2**: let's talk about the following scenario. The primary has sent the input records to the backup. But it crashed after generating the output to the client. Then, the backup takes over as primary. The new primary will generates the same output the client as the old primary did.How to solve it?

Well, we send packest with TCP and the new primary is identical with the old primary. So the new primary will send packets with same packet number and the client will discard the same packet.



**Problem 3**: brain split. If the primary and the backup cannot talk with each other, They will both be the primary and generate output to the client. How to solve it?

Via Shared disk and using test-and-set to ensure there is only one primary. If test-and-set success, the one can go live. Otherwise, cannot.