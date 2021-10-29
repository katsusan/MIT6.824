### 1. Introduction

Raft's novel features:
- Strong leader: stronger form of leadership. 比如log只会从leader流向其它server
- Leader election: 使用randomized timers选主
- Membership changes: 对于changing the set of servers in the cluster会使用 a new joint consensus approach(?)

### 2. Replicated state machines

GFS/HDFS/RAMCloud have a single cluster leader, typically use a separate replicated   
state machine to manage leader election and store configuration information.

Chubby/ZooKeeper use replicated state machines.

properties of consensus algorithms for practical systems： 
- safety: never returning an incorrect result
- available:  fully functional  as long as any majority of the servers are operational and can communicate. 
- do not depend on timing to ensure the consistency of logs
- a command can complete as soon as a majority of the cluster has responded to a single round of remote procedure calls;

### 3. What’s wrong with Paxos?

two significant drawbacks:
- exceptionally difficult to understand; single-decree, reaching consensus on multiple decisions
- does not provide a good foundation for building practical implementations.

### 4. Designing for understandability
goals in designing Raft:
- must provide a complete and practical foundation for system building
- t must be safe under all conditions and available under typical operating conditions
- must be efficient for common operations

### 5. The Raft consensus algorithm

Raft implements consensus by first electing a distinguished leader,   
then giving the leader complete responsibility for managing the replicated log.   

Leader accepts log entries from clients, replicates them on other servers, and tells servers   
when it is safe to apply log entries to their state machines.   

Consensus problem decomposition:
- Leader election: a new leader must be chosen when an existing leader fails.
- Log replication: the leader must accept log entries from clients and replicate them across the cluster.
- Safety: if any server has applied a particular log entry to its state machine, then no other server may
   apply a different command for the same log index.


