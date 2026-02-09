### Create user for DB replication

CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

Tạo user tên repl
'%' = cho phép slave kết nối từ bất kỳ IP nào
  - Trong Docker / cloud → IP thường thay đổi
GRANT REPLICATION SLAVE ON *.*
  - Quyền này cho phép:
  Slave:
    - Kết nối tới Master
    - Đọc binary log (binlog)
    - Đồng bộ dữ liệu

Replication => Slave DB need a valid user to connect to MasterDB and read binlog - binary log (1)
Mọi thay đổi trong DB sẽ được log trong binlog

* INSERT sinh binlog → slave đọc binlog → data được sync *

***Implement config***
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      827 | app_db       |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
Kết nối Slave với Master

### Slave 1
docker exec -it mysql-slave-1 mysql -uroot -proot

CHANGE MASTER TO
MASTER_HOST='mysql-master',
MASTER_USER='repl',
MASTER_PASSWORD='replpass',
MASTER_LOG_FILE='mysql-bin.000003',
MASTER_LOG_POS=827;
START SLAVE;

### Slave 2
docker exec -it mysql-slave-2 mysql -uroot -proot
SAME AS Slave 1

SHOW SLAVE STATUS\G

Slave_IO_Running: Yes
Slave_SQL_Running: Yes

ISSUE:
Trường hợp phổ biến
  - Master đã INSERT/UPDATE
  - Binlog đã sinh
  - Slave chưa kịp apply
  - User query vào Slave
  User sẽ thấy data cũ (stale data) => ***Replication Lag*** (độ trễ giữa master và slave)

SOLUTION:
  Read-after-write → đọc từ Master
  Cách 1:
    Theo request lifecycle (đơn giản nhất)
      HTTP request
        ├─ POST /orders (write)
        ├─ GET /orders/last (read)
        └─ response
  Cách 2:
* Promote slave to master?
