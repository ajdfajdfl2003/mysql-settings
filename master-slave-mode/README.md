#### 初始化和啟動資料庫

一個空的資料庫需要執行

```bash
mysqld --defaults-file=my.cnf --initialize-insecure
## --defaults-file => 給定一個設定檔來啟動 mysql 資料庫
## --initialize-insecure => 代表我們的 root 密碼可為空
```

進行初始化的動作
分別啟動 `主`和 `從`，在 command line 下執行

```bash
mysqld --defaults-file=my.cnf 
```

#### 設定 主 節點
mysql指令登入到 `主` 節點

```bash
mysql -h127.0.0.1 -P 33016 -uroot
```

```sql
-- 確認是否有連到正確的 instance
show variables like '%port%';

-- 創建一個帳號 repl，密碼為 123456
-- 待會兒會用來做主從的資料同步
CREATE USER 'repl'@'%' IDENTIFIED BY '123456';

-- 改一下權限
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

flush privileges;
 
-- 檢視一下 master 這台 instance 的狀態
show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+

--- 新建一個資料庫：
create schema db;
```

#### 設定 從 節點

mysql指令登入到 `從` 節點

```bash
mysql -h127.0.0.1 -P 33026 -uroot
```

```sql
-- 確認是否有連到正確的 instance
show variables like '%port%';

-- 告訴 slave 節點，master 在哪
CHANGE MASTER TO
    MASTER_HOST='localhost',  
    MASTER_PORT = 33016,
    MASTER_USER='repl',      
    MASTER_PASSWORD='123456',   
-- 下面這兩個參數會根據主節點的 status 而有所不同
    MASTER_LOG_FILE='mysql-bin.000002',
    MASTER_LOG_POS=1036;
    
    //MASTER_AUTO_POSITION = 1;
	
--- 新建一個資料庫：
create schema db;

```

##### TroubleShooting

```sql
Query OK, 0 rows affected, 8 warnings (0.01 sec)

-- 我們發現有 warnings
-- 可以用下面指令來看 warnings
show warnings;

+---------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Warning | 1287 | 'CHANGE MASTER' is deprecated and will be removed in a future release. Please use CHANGE REPLICATION SOURCE instead                                                                                                                                                                  |
| Warning | 1287 | 'MASTER_HOST' is deprecated and will be removed in a future release. Please use SOURCE_HOST instead                                                                                                                                                                                  |
| Warning | 1287 | 'MASTER_PORT' is deprecated and will be removed in a future release. Please use SOURCE_PORT instead                                                                                                                                                                                  |
| Warning | 1287 | 'MASTER_USER' is deprecated and will be removed in a future release. Please use SOURCE_USER instead                                                                                                                                                                                  |
| Warning | 1287 | 'MASTER_PASSWORD' is deprecated and will be removed in a future release. Please use SOURCE_PASSWORD instead                                                                                                                                                                          |
| Warning | 1287 | 'MASTER_LOG_FILE' is deprecated and will be removed in a future release. Please use SOURCE_LOG_FILE instead                                                                                                                                                                          |
| Warning | 1287 | 'MASTER_LOG_POS' is deprecated and will be removed in a future release. Please use SOURCE_LOG_POS instead                                                                                                                                                                            |
| Note    | 1760 | Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information. |
+---------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
8 rows in set (0.00 sec)

-- deprecated 的 warning 先不看，我們先看一下 1760 這條警告
-- 他是說我們的 slave 狀態不對，需要 start slave
-- 我們在用下面指令觀察 slave 狀態
show slave status\G

-- 發現我們 slave 真的還沒有啟動
-- 透過下面指令啟動 slave
start slave
```

---

```sql
show slave status\G
-- 發現有以下錯誤
Last_IO_Error: error connecting to master 'repl@localhost:33016' - retry-time: 60 retries: 1 message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.

-- 驗證請求不正確，caching_sha2_password 這個是 MySQL 8 新加的
-- 我們來調整一下 
-- 可以在 my.cnf 加上
-- default_authentication_plugin=mysql_native_password
-- 或者去修改目前使用者的驗證方式
-- 我們在 主 節點上觀察一下 使用者 驗證使用的套件是什麼
select host, user, plugin from mysql.user;
+-----------+------------------+-----------------------+
| host      | user             | plugin                |
+-----------+------------------+-----------------------+
| %         | repl             | caching_sha2_password |
| localhost | mysql.infoschema | caching_sha2_password |
| localhost | mysql.session    | caching_sha2_password |
| localhost | mysql.sys        | caching_sha2_password |
| localhost | root             | caching_sha2_password |
+-----------+------------------+-----------------------+
5 rows in set (0.00 sec)

-- 修改該使用者的加密方式
alter user 'repl'@'%' IDENTIFIED with mysql_native_password by '123456';
flush privileges;

-- 我們在 主 節點上觀察狀態一下
show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |     1485 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

-- 可以透過下面指令來停止複製
stop slave;

-- 清除 slave 設定
reset slave all;

-- 我們再重新設定一下 replica(slave)
CHANGE MASTER TO
    MASTER_HOST='localhost',  
    MASTER_PORT = 33016,
    MASTER_USER='repl',      
    MASTER_PASSWORD='123456',   
    MASTER_LOG_FILE='mysql-bin.000002',
    MASTER_LOG_POS=1485;
	
start replica;
```


#### 驗證操作

在 master 上執行

```sql
use db

create table t1(id int);

insert into t1(id) values(1),(2);

```

在 slave 觀察資料同步的情況

```sql
use db

show tables;
+--------------+
| Tables_in_db |
+--------------+
| t1           |
+--------------+

select * from t1;
+------+
| id   |
+------+
|    1 |
|    2 |
+------+

```

#### 只希望在 master 上做，但 replica 不要
```sql
-- 先暫時關閉 binlog
set SQL_LOG_BIN=0
.
.
-- 中間先做一下 master 想做的事情
.
.
-- 開啟 binlog
set SQL_LOG_BIN=1
```

#### 觀察的指令

```sql
-- 可以通過下面指令觀察狀態
show master status;
show slave status;

show master status\G
show slave status\G

-- 可以透過下面指令來停止複製
stop slave;

-- 清除 slave 設定
reset slave all;
```
