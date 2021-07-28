flow:

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

//******Wait all map tasks done******//   

Assuming there are T workers and R reduce tasks,   
each worker should pick at most [R/T]+1 reduce tasks.

- client  →   server   
  RPC：ReduceTaskAssign   
  args:
  reply:   
  
  
  












