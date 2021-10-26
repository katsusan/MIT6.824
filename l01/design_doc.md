worker flow:

- client  →   server   
  RPC：AddWorker   
  args: nil // maybe need uuid here?   
  reply: nil   

- client  →   server   
  RPC: MapTaskAssign   
  args: nil   
  reply: MapTask{Filename, mapTaskId, mapTaskDone}   

- client  →   server   
  RPC：MapTaskSubmit   
  args：MapTaskSubmitReq{intmdFilename, taskId}   
  reply: MapTaskSubmitResp{Errmsg}   

- client  →   server 
  RPC: FinishReduce   
  args: ReduceTaskAssignReq{WorkerId}   
  reply: ReduceTaskAssignResp{AssignId, ReduceTaskAllDone}   

- client →    server   
  RPC: ReduceTaskSubmit   
  args: ReduceTaskSubmitReq{WorkerId, AssignId}   
  reply: CommonResp{}
  
************************************************************************
// ******Wait all map tasks done****** //   
// intermediate files: mr-X-Y, with X=mapId, Y=reduceId  //   
// mapId is the index of files, reduceId is hash(word)%nReduce //    

every worker calls Coordinator.FinishReduce to request for available reduce tasks,
when some workers crashed during reduce, its reduce task will be released into task pool.
************************************************************************
Q&A:
1. Worker Failure   
Q: How do keepalive work here?   
A: In paper it is decided that `The master pings every worker periodically`,   
   but in lab we can exploit the original RPC communication, letting worker    
   doing RPC call periodically to show it is alive.      
   This is different from paper but doesn't need to establish an extra transport   
   layer, thus decreases the complexity of the lab work.   
   
2. map work failure   
  Paper 3.3   
  Completed map tasks are re-executed on a failure because their output is stored   
  on the local disk(s) of the failed machine and is therefore inaccessible.    
  We use Coordinator.mapTaskAssignation to record map task assigning.

3. reduce work failure   
  Paper 3.3   
  Completed reduce tasks do not need to be re-executed since their output is stored   
  in a global file system.

  
  












