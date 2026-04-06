# 1. 初始数据同步

当建立一个全新的从库时，你需要先让从库的数据与主库的某个时间点保持一致。这通常通过以下方式完成，这个过程通常被认为是“全量同步”：

1. 在主库上执行 `FLUSH TABLES WITH READ LOCK;` 获取一个全局读锁（或使用不锁表的备份工具，如 `mysqldump --single-transaction`）。
2. 记录下主库当前的 binlog 文件名和位置点（`SHOW MASTER STATUS;`）。
3. 备份主库的数据（使用 `mysqldump`、`Xtrabackup` 等工具）。
4. 在主库上解锁 (`UNLOCK TABLES;`)。
5. 将备份文件恢复到从库。
6. 在从库上配置 `CHANGE MASTER TO`，使用步骤2中记录的 binlog 文件名和位置点。
7. 启动复制 (`START SLAVE`)。从库会从该位置点开始，进行增量同步。

# 2. 主从连接建立

1. 从服务器配置：在从服务器上执行 `CHANGE MASTER TO` 命令，指定主服务器的 `HOST`, `PORT`, `USER`, `PASSWORD`，以及复制的起点（使用 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS`，或使用 `MASTER_AUTO_POSITION=1` 启用 GTID）。
2. 启动从服务器复制：执行 `START SLAVE;`（或在 MySQL 8.0+ 中为 `START REPLICA;`）。
3. 从服务器 I/O 线程启动：从服务器上的 I/O 线程根据配置信息，与主服务器建立连接。
4. 主服务器创建 Dump 线程：主服务器接受从服务器的连接后，会为每个连接的从服务器创建一个 `Binlog Dump` 线程。

# 3. 数据同步（binlog 传输与重放）

这个过程完美地对应了上图中的三个核心线程：

1. I/O 线程拉取 binlog (数据拉取)：

- 从服务器的 I/O 线程向主服务器的 `Binlog Dump` 线程请求 binlog 数据。
- `Binlog Dump` 线程读取主服务器的 binlog，并根据从服务器请求的位置点，将新的 binlog 事件发送给从服务器。
- 从服务器的 I/O 线程接收这些 binlog 事件。

2. 写入中继日志 (本地存储)：

- 从服务器的 I/O 线程将接收到的 binlog 事件顺序写入到本地的中继日志 (relay log) 文件中。
- 关键点：写入中继日志的操作与 SQL 线程重放日志是解耦的。即使 SQL 线程重放较慢，I/O 线程也能持续从主库拉取数据，确保有完整的复制缓冲。

3. SQL 线程重放日志 (数据应用)：

- 从服务器的 SQL 线程会持续监控中继日志文件。
- 一旦发现新的日志事件，它会立即读取这些事件。
- SQL 线程解析并执行这些事件所代表的 SQL 命令（或行变更），从而更新从服务器上的数据。
- 执行完成后，SQL 线程会更新从服务器上的复制位点信息（如 `Relay_Master_Log_File` 和 `Exec_Master_Log_Pos`，或在 GTID 模式下更新 `Retrieved_Gtid_Set` 和 `Executed_Gtid_Set`）。