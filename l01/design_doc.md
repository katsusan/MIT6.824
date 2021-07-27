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

Reducers should only 

- client  →   server
  RPC：ReduceTaskAssign
  












