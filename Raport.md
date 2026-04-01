# Assignment 1
## Ex1
This exercise demonstrates `CP` (Consistency and Partition Tolerance). Hazelcast’s CPSubsystem uses the Raft consensus algorithm to ensure that data remains consistent even during network failures, specifically by requiring a quorum (majority) to perform operations.
### Phase-by-Phase Breakdown
1. Healthy State (Nodes 1, 2, 3)
    - Behavior: Writes and reads are successful on all nodes.
    - Reason: The Raft leader replicates the AtomicLong update to at least two nodes (majority). Since all nodes are connected, they maintain a perfectly synchronized state (Consistency).

2. Partition State (Node 3 Isolated)
    - Minority Side (Node 3):
      - a:3 and g:3 both return (timeout - no quorum).
      - Why? Node 3 cannot see the other nodes. In a 3-node cluster, a quorum requires $\frac{3+1}{2} = 2$ nodes. Since Node 3 is alone, it cannot elect a leader or verify the "latest" data. It sacrifices Availability to prevent inconsistent data (stale reads or diverging writes).
    - Majority Side (Nodes 1 & 2):
      - a:1 and g:1 succeed.
      - Why? Nodes 1 and 2 can still communicate. They maintain a quorum ($2/3$), allowing them to continue processing requests safely.3. 
  
3. Healing State (Network Restored)
    - Node 3 initially reports a timeout, then eventually returns the value $3$.
    - Why? Once the "bridge" is restored, Node 3 must perform a catch-up procedure. It contacts the leader (from the 1-2 partition), replicates the missing log entries, and updates its local state.
  
### Summary Table
Step | Node 1 (Majority) | Node 3 (Minority) |Logic
| --- | --- | --- | --- |
Initial | Success | Success |Full connectivity.
Split | Available | Unavailable | Quorum required for Consistency.
Heal | Consistent | Catching up... | Node 3 synchronizes with the Leader.

## Ex2

This exercise explores a "Total Partition" scenario where every node is isolated from every other node. In this state, the cluster has zero communication, making it impossible to form a majority.
### Analysis of Operations
1. Healthy State
    - All nodes show a value of $1$ after the increment.
    - Why? Standard Raft behavior. The leader replicates the write to a majority of nodes before confirming success.

2. Total Partition (Nodes 1, 2, and 3 Isolated)
    - Every single request (g:1, a:1, ga) fails with a timeout.
    - Why? For a 3-node cluster, the quorum size is $2$. Under total isolation, each node is in its own partition of size $1$. Since $1 < 2$, no node can act as a leader or perform a valid read/write.
    - (CAP Property) The system prioritizes Consistency (C) and Partition Tolerance (P) by sacrificing Availability (A). It is better for the system to stop responding than to risk data corruption or inconsistent states.
    
3. Healing the Partition
    - Immediately after healing, timeouts persist. After some time, nodes return the value $1$ one by one.
    - Why? (Election Delay) Once the network is restored, the nodes must hold a new election to choose a leader. This involves timeouts and voting rounds, which takes time.
    - Re-synchronization: The nodes must verify their logs against each other to ensure they are on the same page before they can safely serve requests again.
    
### Comparison: Exercise 1 vs. Exercise 2
| Feature | Exercise 1 (Partial Split) | Exercise 2 (Total Split) |
| --- | --- | --- |
Majority Side | Available (Nodes 1 & 2) | None
Minority Side | Unavailable (Node 3) | All Nodes (1, 2, 3)
System Status | Partially Functional |Fully Stalled

## Ex3

This exercise demonstrates AP (Availability and Partition Tolerance). Unlike the previous exercises, we use a PNCounter (Positive-Negative Counter), which is a CRDT (Conflict-free Replicated Data Type). This structure prioritizes high availability and eventual consistency.

### Analysis of Operations
1. Healthy State
    - **Increment Node 1.**
      All nodes immediately reflect the value $1$.
    - Why? 
      In a healthy network, CRDT updates are broadcast to all peers. Each node maintains its own state and merges incoming updates.

2. Network Partition (Node 3 Isolated)
    - ** Increment Node 1, 2, and 3 while split.**
      - (Availability) All nodes, including the isolated Node 3, successfully performed reads and writes. No "no quorum" timeouts occurred.
      - (Divergence) Nodes 1 and 2 stayed synchronized (value $4$), while Node 3 drifted (value $2$).
      - Why? CRDTs do not require a quorum. Each replica can accept writes locally. Node 1 and 2 could talk to each other, so they merged their increments. Node 3 was alone, so it only knew about its own local increments. The system remains functional on both sides of a partition at the cost of Consistency.
3. Healing the Partition
    - **Network restored and ga called.**
    Initially, the values were still $4$ and $2$. After a brief moment, all nodes converged to $5$.
    - Why? Deterministic Merging: When the network healed, the nodes exchanged their internal CRDT state vectors.The Math: The system doesn't just "pick a winner." It merges the operations.Updates from Node 1/2 side: (Initial 1) + (new updates from 1 & 2) = $4$. Updates from Node 3 side: (Initial 1) + (new updates from 3) = $2$.Total: The combined unique increments across the whole cluster equal $5$.Eventual Consistency: The state converged automatically without manual conflict resolution.
    
### CP (Ex 1 & 2) vs. AP (Ex 3)

| Feature | CP (AtomicLong) |AP (PNCounter)
| --- | --- | --- |
Partition Behavior | Minority side blocks (Timeouts) | All sides remain functional
Data Integrity | Always accurate/latest | Eventually consistent
Requirement | Quorum (Majority) | Local availability
Conflict Handling | Prevention (only 1 leader) | Resolution (merge math)

# Assignment 2

In the context of Hazelcast and distributed systems, CP (Consistent and Partition-tolerant) data structures require the Raft algorithm. AP structures, like the PNCounter you used in Exercise 3, do not use Raft because they rely on replication and conflict resolution (CRDTs) rather than strict consensus.

## Why is Raft needed?
The Raft algorithm is a consensus protocol designed to ensure that a cluster of nodes acts like a single, atomic unit. It is required for CP structures for several reasons:

- Ensuring that at any given time, there is exactly one "Leader" responsible for managing the log. 

- Prevents "Split Brain" scenarios where two nodes might try to commit different values to the same variable.

- Every change (like `a:1`) is treated as a log entry. The leader sends this entry to all followers.

- A change is only considered "committed" and visible to the user once a majority of nodes (a quorum) have acknowledged it. This ensures that even if some nodes fail, the data is safely stored on enough nodes to survive.

- Raft ensures that reads and writes happen in a specific, predictable order, fulfilling the Consistency requirement of CAP.

## The Role of Raft After a Partition Heal
When a network partition is restored (heal), Raft is responsible for bringing the cluster back into a unified, consistent state. Its roles include:

1. Re-establishing Leadership
If the partition caused the cluster to lose its leader or created two competing leaders (in different terms), Raft uses Term Numbers to determine who the true leader is. The node with the highest term number wins, and any "stale" leaders step down to become followers.

2. Log Matching (The Catch-up)
The isolated node (like Node 3 in Exercise 1) will have a log that is shorter or different from the majority.
The Leader identifies the last point where their logs matched.
The Leader sends all missing log entries to the recovered node.
The recovered node overwrites any inconsistent entries to match the Leader exactly.

3. Restoring Quorum
Raft transitions the cluster from a "degraded" state (where it might have been barely holding a majority) back to a "healthy" state where all nodes are participating in the voting process.

# Assignment 3
### RYW
From what I've read Read-Your-Writes (RYW) is a guarantee that ensures that if you perform an update (a Write) to the PN Counter, any subsequent query (a Read) you make will reflect that update.

### RYW example

Image user increments Node 1. If they then immediately move to Node 3 (which might not have received the update yet due to network lag) and ask for the value, a system providing RYW will ensure you don't see the "old" value. The system either:

- Redirects your read to a node that has your update.
- Delays the read until Node 3 catches up.
- Returns an error if the guarantee cannot be met.

### Monotonic Reads

This guarantee ensures that if you read a certain value from the database, any subsequent reads will return a state that is at least as up-to-date as the first one. Your "view" of the data never moves backward in time.

### MR example

Suppose user reads the counter and sees the value 10. Even if the network is partitioned or they switch servers, the system guarantees that their next read will return 10 or greater. You will never see the counter "flicker" back to 9 just because they happened to connect to a slower replica.

### Why are these called "Session Guarantees"?
The paper defines a Session as an abstraction for a sequence of operations performed by a single application or user. These are called "session" guarantees because:

1. They are not global properties. They don't guarantee that everyone sees the same thing at the same time (strong consistency). Instead, they guarantee that the database appears consistent relative to your own actions.

2. They allow a client to move between different, potentially inconsistent servers while maintaining a coherent personal history. As the paper notes, as long as the "Session Manager" tracks what you have seen/written (using version vectors), it can ensure your next interaction happens with a server that meets your session's specific requirements.

3. By requesting these only for a specific session, the system can remain highly available (AP) for other users who don't need them. You only pay the performance/availability penalty for the specific consistency you require for your current task.