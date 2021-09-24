worker flow:

- client  →   server   
  RPC：AddWorker   
  args: nil // maybe need uuid here?   
  reply: nil   

- client  →   server   
  RPC: MapTaskAssign   
  args: nil   
  reply: MapTask{Filename, mapTaskId}   

- client  →   server   
  RPC：EmitMapTask   
  args：MapTaskEmitReq{intmdFilename}   
  reply: nil   

************************************************************************
// ******Wait all map tasks done****** //   
// intermediate files: mr-X-Y, with X=mapId, Y=reduceId  //   
// mapId is the index of files, reduceId is hash(filename)%nReduce //    

Assuming there are T workers and R reduce tasks,   
each worker should pick at most [R/T]+1 reduce tasks.   
(we assuming every worker can afford almost the same workload here)   

That is, worker[x] should handle reduce task[x/x+T/x+2T/...].  
Eg: T=3, R=10, then    
  - worker[0]->reduceTask[0/3/6/9]   
  - worker[1]->reduceTask[1/4/7]
  - worker[2]->reduceTask[2/5/8]

When worker[x] crashes, reduce task[x, x+T,...] need redo, which means   
the worker who accomplishes its reduce task can pick the crashed one.

*That is*, before every worker exits, it should check if there are remaining
work left.

  
  












