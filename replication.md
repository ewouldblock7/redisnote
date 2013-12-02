slave 启动流程
=============
 
1. 初始化时，读取配置文件，根据slaveof将server.repl_state 赋值为REDIS_REPL_CONNECT.
2. replicationCron 中判断到server.repl_state == REDIS_REPL_CONNECT便开始进入连接master的流程。
3. connectWithMaster则进入实际的非阻塞连接master,随后将server.repl_state 赋值为REDIS_REPL_CONNECTING。
4. 在connect的回调函数中，当server.repl_state == REDIS_REPL_CONNECT时，取消对master fd的写事件监听，读事件仍然监听。
5. 非阻塞ping下master确认是否仍然可用，状态改为REDIS_REPL_RECEIVE_PONG。
6. 读事件回调时，判断到server.repl_state == REDIS_REPL_RECEIVE_PONG，取消读事件的监听, 开始阻塞的读ping的master端的答复。
7. 将slave自己的端口配置告知master. 阻塞的发起sync命令。创建tmp.rdb文件，等待异步读取master replication答复的回调(readSyncBulkPayload)来append tmp.rdb文件。将状态改为REDIS_REPL_TRANSFER.
8. 在readSyncBulkPayload中，判断到server.repl_transfer_read == server.repl_transfer_size则表示replication传输完成，状态改为REDIS_REPL_CONNECTED.

master端replication流程。
=============
1. master收到slave的 sync命令，判断slave状态是否为REDIS_REPL_CONNECTED， 是，则继续。
2. 如果bgsave正在进行，并且有一个slave状态为REDIS_REPL_WAIT_BGSAVE_END， 则将该slave状态改为REDIS_REPL_WAIT_BGSAVE_END， 否则，状态为REDIS_REPL_WAIT_BGSAVE_START。
3. 如果bgsave不是正在进行的状态，则启动rdbSaveBackground。并将slave状态改为REDIS_REPL_WAIT_BGSAVE_END。
4. 将salve添加到server.slaves中。
5. （后台进程处理）在master的serverCron中，如果判断到server.rdb_child_pid != -1， 则非阻塞的wait子进程.如果wait的返回pid == server.rdb_child_pid. 则进行backgroundSaveDoneHandler。
6. （后台进程处理）在backgroundSaveDoneHandler中， 会进行updateSlavesWaitingBgsave。
7. （后台进程处理）updateSlavesWaitingBgsave则将那些状态为REDIS_REPL_WAIT_BGSAVE_END的slave创建非阻塞异步的写事件。
8. （后台进程处理）7步骤中写事件通过sendBulkToSlave回调，sendBulkToSlave将master的rdb内容分多次同步给slave。
9. 同步期间，master端的修改均通过processInputBuffer（） -> processCommand() -> call() ->propagate() -> replicationFeedSlaves() 来写到客户端的buf里（redisClient->buf）。当8步骤同步完成时，即已经完成的大小和dump的rdb大小一致时，将同步期间master端的修改再次发送给slave。


2.8 Partial Replication的几个点
=============
+ 首先partial replication存在于一个成功的replication后。初次replication若不成功，随后的重新replication会是full replication.
+ master端通过一个backlog来实现增量同步。backlog是一个环形队列，因此只能保存最近min(repl_backlog_size,实际长度）的增量数据。因此slave端的reploff必须在这个增量数据范围内才能满足增量传输。
+ 对于slave端传上来的reploff,当满足上一条中的数据范围时，还需计算出需同步的一个数据字节的位置。
+ slave端通过cached_master缓存了master信息。这一步是replicationCron时检测到master超时，进而freeclient时判断到时master,进而replicationCacheMaster。下一次尝试增量同步时就是利用cached_master缓存的master id.
+ master通过initServerConfig()中getRandomHexChars来生成masterid. getRandomHexChars是通过/dev/urandom来生成随机数的。
+ 增量同步时并不是像全量同步时新建个tmp.rdb，来append。而是通过replicationResurrectCachedMaster()中注册的readQueryFromClient回调来更新数据。




