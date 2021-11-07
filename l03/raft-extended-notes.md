### 1. Introduction

Raft's novel features:
- Strong leader: stronger form of leadership. // 比如log只会从leader流向其它server
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


#### 5.1 Raft basics

At any given time each server is in one of three states: leader/follower/candidate.

In normal operation there is exactly one leader and all of the other servers are followers.   
// 通常情况下只有一个leader，其余均为follower。

Followers are passive: they issue no requests on their own but simply respond to requests    
from leaders and candidates.   
// follower只会响应来自leader或candidate的请求。

The leader handles all client requests (if a client contacts a follower, the follower redirects   
it to the leader).   
// leader负责处理所有的client请求，如果有client请求一个follower，follower会把请求重定向给leader。  

[**Terms**]   
![image](https://user-images.githubusercontent.com/11537821/139576295-c89f52a1-710c-4b25-9213-8f3d98af31fa.png)

Raft divides time into *terms* of arbitrary length, terms are numbered with consecutive integers.   

Each term begins with an *election*, in which one or more candidates attempt to become leader.   
// 每个term始于一次election。

If a candidate wins the election, then it serves as leader for the rest of the term.   

In some situations an election will result in a split vote. In this case the term will end with   
no leader; a new term (with a new election) will begin shortly.   
// 极端情况下选举无法选出leader后会马上开始一个新的term开始选举。   

Terms act as a logical clock [14] in Raft.   

Each server stores a current term number, which increases monotonically over time.   
// 每个server存储随当前term编号，随时间单调递增。   
 
one server’s current term is smaller than the other’s →  it updates its current term to the larger value.   

a candidate or leader discovers that its term is out of date → reverts to follower state.   
// candidate/leader发现自己的term过期则转换为follower状态。   

a server receives a request with a stale term number → rejects the request.   
// server会拒绝带过期term的请求。   

Raft servers communicate using RPC, the basic consensus algorithm requires only two types of RPCs:
- RequestVote RPCs: initiated by candidates during elections
- AppendEntries RPCs: initiated by leaders to replicate log entries and to provide a form of heartbeat


#### 5.2 Leader election

Raft uses a heartbeat mechanism to trigger leader election. When servers start up, they begin as followers.   

A server remains in follower state as long as it receives valid RPCs from a leader or candidate.   
// server在收到leader或candidate的valid RPC前处于follower状态。

Leaders send periodic heartbeats (AppendEntries RPCs that carry no log entries) to all followers in order to   
maintain their authority.   
// leader周期性向所有follower发送心跳(无log的AppendEntries RPC)来维护自己的leader地位。

If a follower receives no communication over a period of time called the *election timeout*, then it assumes   
there is no viable leader and begins an election to choose a new leader.   
// follower在election timeout内没有收到心跳消息，那么它会认为leader不可用并且开始一轮选举。   

To begin an election, a follower increments its current term and transitions to candidate state.   
// 开始一轮新的选举之前，follower自增其term然后转换到candidate状态。

It then votes for itself and issues RequestVote RPCs in parallel to each of the other servers in the cluster.   
// 然后它会vote给自己，同时发送RequestVote RPC给集群内其它server。

A candidate continues in this state until one of three things happens:
- it wins the election
- another server establishes itself as leader
- a period of time goes by with no winner

A candidate wins an election if it receives votes from a majority of the servers in the full cluster for the same   
term.    // 获取到*相同term*内整个集群的大多数vote视作赢得选举。   

Each server will vote for at most one candidate in a given term, on a first-come-first-served basis.   
// 一个term内每台server只能投票给最多一位candidate，原则上采用先到先得的方式。   

Once a candidate wins an election, it becomes leader. It then sends heartbeat messages to all of the other servers   
to establish its authority and prevent new elections.   
// candidate一旦赢得选举则转换为leader。然后发送heartbeat信息到所有其它server,行使leader权限防止新的选举。   

While waiting for votes, a candidate may receive an AppendEntries RPC from another server claiming to be leader：  
- the leader’s term (included in its RPC) is at least as large as the candidate’s current term  →  
   recognizes the leader as legitimate and returns to follower state
- the term in the RPC is smaller than the candidate’s current term   →  
   the candidate rejects the RPC and continues in candidate state 
// 只有RPC的term大于等于candidate当前term的时候candidate才会停止选举然后转换到follower状态。   

The third possible outcome is that a candidate neither wins nor loses the election: if many followers become candidates   
at the same time, votes could be split so that no candidate obtains a majority.   
When this happens, each candidate will time out and start a new election by incrementing its term and initiating another   
round of RequestVote RPCs. However, without extra measures split votes could repeat indefinitely.   
// 还有一种可能性是没有candidate赢得大多数vote，这时candidate会自增其term然后开始一轮新的选举，需要额外措施防止无限重复选举。   

Raft uses randomized election timeouts to ensure that split votes are rare and that they are resolved quickly.   
To prevent split votes in the first place, election timeouts are chosen randomly from a fixed interval (e.g., 150–300ms).   
This spreads out the servers so that in most cases only a single server will time out; it wins the election and sends   
heartbeats before any other servers time out.   
The same mechanism is used to handle split votes. Each candidate restarts its randomized election timeout at the start of an   
election, and it waits for that timeout to elapse before starting the next election.   
// Raft使用随机选举超时机制保证split votes很难发生。?(见Section9.3)


#### 5.3 Log replication

Once a leader has been elected, it begins servicing client requests. Each client request contains a command to be executed   
by the replicated state machines.   

The leader appends the command to its log as a new entry, then issues AppendEntries RPCs in parallel to each of the other   
servers to replicate the entry.   
// leader把command作为一个新entry合并到log里，然后发出AppendEntries RPC到其它server来复制该日志条目。

When the entry has been safely replicated (as described below), the leader applies the entry to its state machine and    
returns the result of that execution to the client.   
// 日志条目被安全地复制后，leader把它应用到状态机上然后返回执行结果给client。

If followers crash or run slowly, or if network packets are lost, the leader retries AppendEntries RPCs indefinitely   
(even after it has responded to the client) until all followers eventually store all log entries.   
// 如果follower崩溃or运行缓慢，或者RPC时发生网络丢包，leader会无限重试AppendEntries RPC直到所有follower最终存储同样的日志条目。   

![image](https://user-images.githubusercontent.com/11537821/140048937-60e0ddb1-de5c-4c98-9c36-6d3bd099d70a.png)

The leader decides when it is safe to apply a log entry to the state machines; such an entry is called committed.   
// leader决定何时把log entry应用到状态机上，也就是所谓的commmitted entry。   

Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines.   
// Raft保证committed日志都是持久化的，并且最终被所有可用的状态机执行。  
  
A log entry is committed once the leader that created the entry has replicated it on a majority of the servers.   
// 一旦leader把entry复制到集群内的大多数server上则该log entry会被提交，比如上图中的entry 7。   

This also commits all preceding entries in the leader’s log, including entries created by previous leaders.   
// 同时leader中所有之前的日志条目也会被提交，包括之前leader所创建的条目。   

The leader keeps track of the highest index it knows to be committed, and it includes that index in future AppendEntries RPCs   
(including heartbeats) so that the other servers eventually find out.   
// leader跟踪目前已提交日志的最大index，然后把这个index包含在之后发出的AppendEntries RPC(心跳也算)，这样其它server最终能发现该index。   

Once a follower learns that a log entry is committed, it applies the entry to its local state machine (in log order).   
// 一旦follower了解到某个log entry提交了，它会在本地状态机上执行该entry。   

Raft maintains the following properties, which together constitute the Log Matching Property in Figure 3:   
- If two entries in different logs have the same index and term, then they store the same command.
- If two entries in different logs have the same index and term, then the logs are identical in all preceding entries.

The first property follows from the fact that a leader creates at most one entry with a given log index in a given term,   
and log entries never change their position in the log.   
// 第一条源于这样一个事实：leader在给定term的给定log index上只能创建最多一条log entry，并且log entry在日志序列中永远不会改变位置。   

The second property is guaranteed by a simple consistency check performed by AppendEntries. When sending an AppendEntries   
RPC, the leader includes the index and term of the entry in its log that immediately precedes the new entries.   
If the follower does not find an entry in its log with the same index and term, then it refuses the new entries.    
// 第二条由AppendEntries执行时的简单一致性check保证。leader在发送AppendEntries时会附加新条目之前的index和term，   
// 如果follower在自己的log里找不到该entry则拒绝添加该条目。   

-------------------------------------------------------------
![image](https://user-images.githubusercontent.com/11537821/140494659-e7285296-e94a-46b0-8a0d-291bc12bedac.png)

In Raft, the leader handles inconsistencies by forcing the followers’ logs to duplicate its own. This means that   
conflicting entries in follower logs will be overwritten with entries from the leader’s log.   
// Raft里leader强制follower复制自己的log来达成一致性。也就是follower里冲突的log会被leader里对应的log覆盖。   

[**solution**]   
The leader maintains a nextIndex for each follower, which is the index of the next log entry the leader will   
send to that follower.  When a leader first comes to power, it initializes all nextIndex values to the index   
just after the last one in its log (11 in Figure 7).   
// leader为每个follower维护一个nextIndex，代表下一条发送给follower的log index。   
// leader选举成功后，会初始化nextIndex为自己log的下一条index。   
**↓**    
If a follower’s log is inconsistent with the leader’s, the AppendEntries consistency check will fail in the next AppendEntries RPC.   
// follower日志与leader不一致时，下次AppendEntries RPC会失败。   
**↓**    
After a rejection, the leader decrements nextIndex and retries the AppendEntries RPC.    
**↓**    
Eventually nextIndex will reach a point where the leader and follower logs match.   
**↓**    
When this happens, AppendEntries will succeed, which removes any conflicting entries in the follower’s log and appends   
entries from the leader’s log (if any).   
**↓**    
Once AppendEntries succeeds, the follower’s log is consistent with the leader’s, and it will remain that way for the rest of the term.   

[**Optimization**]   
If desired, the protocol can be optimized to reduce the number of rejected AppendEntries RPCs.   

For example, when rejecting an AppendEntries request, the follower can include the term of the conflicting entry and    
the first index it stores for that term.   

With this information, the leader can decrement nextIndex to bypass all of the conflicting entries in that term;   
one AppendEntries RPC will be required for each term with conflicting entries, rather than one RPC per entry.   

In practice, we doubt this optimization is necessary, since failures happen infrequently and it is unlikely that   
there will be many inconsistent entries.   

With this mechanism, a leader does not need to take any special actions to restore log consistency when it comes to   
power. It just begins normal operation, and the logs automatically converge in response to failures of the AppendEntries   
consistency check. A leader never overwrites or deletes entries in its own log.   

This log replication mechanism exhibits the desirable consensus properties described in Section 2: Raft can accept,   
replicate, and apply new log entries as long as a majority of the servers are up; in the normal case a new entry can   
be replicated with a single round of RPCs to a majority of the cluster; and a single slow follower will not impact performance.   


#### 5.4 Safety

This section completes the Raft algorithm by adding a restriction on which servers may be elected leader.   

The restriction ensures that the leader for any given term contains all of the entries committed in previous terms   
(the Leader Completeness Property from Figure 3).   

##### 5.4.1 Election restriction

In any leader-based consensus algorithm, the leader must eventually store all of the committed log entries.   

Raft uses a simpler approach where it guarantees that all the committed entries from previous terms are present on   
each new leader from the moment of its election, without the need to transfer those entries to the leader.   
// Raft在选举时保证了new leader包含所有之前term已提交的entries，防止了选举成功后需要把这些entries传送给leader的工作。   

This means that log entries only flow in one direction, *from leaders to followers*, and leaders never overwrite   
existing entries in their logs.   
// log entries的流向只从leader到follower，leader从不改写已提交的log。   

Raft uses the voting process to prevent a candidate from winning an election unless its log contains all committed   
entries. A candidate must contact a majority of the cluster in order to be elected, which means that every committed   
entry must be present in at least one of those servers.   
// Raft选举过程中会阻止不包含所有已提交log的candidate获选。candidate必须与集群中的大多数通信以赢得选举，意味着每条已提交的entry   
// 必须存在于至少某台机器上。   

If the candidate’s log is at least as up-to-date as any other log in that majority, then it will hold all the committed entries.    
The RequestVote RPC implements this restriction: the RPC includes information about the candidate’s log, and the voter denies its    
vote if its own log is more up-to-date than that of the candidate.   
// 如果candidate的log至少新于任何大多数里的其它log，那么代表它包含了所有已提交的日志。   
// RequestVote RPC实现了这个限制：RPC里包括了candidate的log的相关信息，voter拒绝承认那些比自己log还旧的candidate。   

Raft determines which of two logs is more up-to-date by comparing the index and term of the last entries in the logs. If the logs    
have last entries with different terms, then the log with the later term is more up-to-date. If the logs end with the same term, then   
whichever log is longer is more up-to-date.   
// Raft通过比较logs中最后一条entry的index和term来判断哪个log更新。   
// 如果logs的最后一条entry的term不同，那么大一点的term代表更新；如果term相同则谁的log更长(多?)代表更新。   


##### 5.4.2 Committing entries from previous terms

As described in Section 5.3, a leader knows that an entry from its current term is committed once that entry is stored   
on a majority of the servers.   
// 5.3节提到了leader知道一旦当前term的entry被大多数机器所存储则会被直接commit。   

![image](https://user-images.githubusercontent.com/11537821/140638492-d041081e-78d9-40cb-b86e-4e1caae5e623.png)

If a leader crashes before committing an entry, future leaders will attempt to finish replicating the entry. However, a leader   
cannot immediately conclude that an entry from a previous term is committed once it is stored on a majority of servers.   
// 如果leader在commit前crash，后面的leader将尝试去继续复制这个entry。


##### 5.4.3 Safety argument

#### 5.5 Follower and candidate crashes

#### 5.6 Timing and availability

### 6 Cluster membership changes

### 7 Log Compaction

### 8 Client interaction

### 9 Implementation and evaluation

#### 9.1 Understandability

#### 9.2 Correctness

#### 9.3 Performance

### 10 Related work

### 11 Conclusion

### 12 Acknowledgment

