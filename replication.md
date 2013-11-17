slave 启动流程  
  初始化时，读取配置文件，根据slaveof将server.repl_state 赋值为REDIS_REPL_CONNECT。  
  replicationCron 中判断到server.repl_state == REDIS_REPL_CONNECT便开始进入连接master的流程。  
connectWithMaster则进入实际的非阻塞连接master,随后将server.repl_state 赋值为REDIS_REPL_CONNECTING。  
在connect的回调函数中，当server.repl_state == REDIS_REPL_CONNECT时，取消对master fd的写事件监听，  
读事件仍然监听，然后非阻塞ping下master确认是否仍然可用，状态改为REDIS_REPL_RECEIVE_PONG。下一次  
读事件回调时， 判断到server.repl_state == REDIS_REPL_RECEIVE_PONG， 取消读事件的监听，开始阻塞的读  
ping的master端的答复。随后将slave自己的端口配置告知master. 阻塞的发起sync命令。创建tmp.rdb文件，等待  
异步读取master replication答复的回调(readSyncBulkPayload)来append tmp.rdb文件。将状态改为  
REDIS_REPL_TRANSFER.在readSyncBulkPayload中，判断到server.repl_transfer_read == server.repl_transfer_size  
则表示replication传输完成，状态改为REDIS_REPL_CONNECTED.



