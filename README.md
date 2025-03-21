# MySQL Master-Slave Replication

### Step 1: Start with your master configuration.
#### This configuration use only in master side database.

```bash
# first run this command to go bash of master
cd /etc/mysql

# if my.cnf not created then create it first and then add this lines with this code
echo "[mysqld]
server-id=1
log-bin=mysql-bin 
binlog-do-db=supersee
binlog-format=ROW
bind-address=0.0.0.0
gtid_mode=ON
enforce-gtid-consistency=ON
" > my.cnf

# Restart your mysql service
sudo systemctl restart mysql
```
- Note for master config file ` my.cnf ` | `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini` use this configuration <br>
  ```
  [mysqld]
  server-id=1 
  log-bin=mysql-bin 
  binlog-do-db=supersee 
  binlog-format=ROW 
  bind-address=0.0.0.0 
  gtid_mode=ON
  enforce-gtid-consistency=ON
  ```

---

### Step 2: Configure On Slave MySQL
```bash
# first run this command to go bash of master
cd /etc/mysql

# if my.cnf not created then create it first and then add this lines with this code
echo "[mysqld]
server-id=2
relay-log=mysql-relay-bin
log-bin=mysql-bin
gtid_mode=ON
enforce_gtid_consistency=ON
" > my.cnf

# Restart your mysql service
sudo systemctl restart mysql
```
- Note for slave/replica config file ` my.cnf ` | `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini` use this configuration <br>
  ```
  [mysqld]
  server-id=2
  relay-log=mysql-relay-bin
  log-bin=mysql-bin
  gtid_mode=ON
  enforce_gtid_consistency=ON
  ```

---

### Step 3: Now create user for Slave | Replica connections.
#### For master side mysql cli create user with this code
```sql
CREATE USER 'username'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'username'@'%';
FLUSH PRIVILEGES;
```

#### Check Master Status
```sql
SHOW MASTER STATUS;
```

### Step 4: Configure Slave for Replication
#### Configure Slave
```sql
-- Stop slave if start
STOP SLAVE;
-- On Slave Side Use This Configuration.
-- With GTID Configuration.
CHANGE MASTER TO
  MASTER_HOST='Master Host Name Ex. supersee',
  MASTER_USER='<username>',
  MASTER_PASSWORD='<password>',
  MASTER_AUTO_POSITION=1;

-- If start slave stop it not start.
STOP SLAVE;
```

#### Verify Slave Status
```sql
SHOW SLAVE STATUS\G
```

#### After configured successfully user export backup database of current in master side.
```bash
# For taking backup without lock the table and maintain consistency use this way
mysqldump -uroot -proot --single-transaction --master-data=2 --flush-logs --hex-blob --set-gtid-purged=ON --databases supersee > /your-path/backup.sql
```
- Note : <br>
`--single-transaction` is for without lock database it can create a backup files. <br>
`--master-data=2` for getting log_file_name log_file_pos in backup sql file. <br>
`--set-gtid-purged=ON` for set the gtid status on <br>
`--databases` After this flag use database name <br>

#### Now we first import backup of master database of master-data.sql in slave side.
```bash
mysql -uroot -proot database_name < /your-path/backup.sql

# After successfull import database we show status of slave and start slave if not configured use above step configure slave.
START SLAVE;
SHOW SLAVE STATUS\G;
```

---

#### Check status of connection
```bash
# Check this configuration 
Master_Host: supersee
Master_User: raoinfotech
Master_Port: 3306
Connect_Retry: 60
Master_Log_File: mysql-bin.000002
Read_Master_Log_Pos: 157
Relay_Log_File: 2eac3f2cee88-relay-bin.000001
Relay_Log_Pos: 4
Relay_Master_Log_File: mysql-bin.000002



# Check the connection status
Slave_IO_Running: Connecting # THIS IS FOR IO CONNECTION Yes if all is set
Slave_SQL_Running: Yes # THIS IS FOR SQL DB CONNECTION Yes if all is set

# If any of the above is Not yes then check the log for error

# THIS WILL CHECK FOR LOGS OF IO
Last_IO_Errno: 0 # THIS WILL SHOW'S ERROR NO
Last_IO_Error: '' # THIS WILL SHOW'S ERROR CAUSE

# THIS WILL CHECK FOR LOGS OF SQL CONNECTION
Last_SQL_Errno: 0 # THIS WILL SHOW'S ERROR NO
Last_SQL_Error: '' # THIS WILL SHOW'S ERROR CAUSE

# FOR EX: CHECK THIS ERROR.
Last_IO_Errno: 2061
Last_IO_Error: Error connecting to source 'raoinfotech@supersee:3306'. This was attempt 1/86400, with a delay of 60 seconds between attempts. Message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
  - NOTE : THIS ERROR CAUSE THE MASTER CONFIG IS SET AS A 'caching_sha2_password' TO FIX USE THIS

# ON MASTER CHANGE CONFIG TO GO IN MASTER DOCKER CONTAINER AND LOGIN IN MYSQL AND RUN THIS QUERY.
SELECT user, host, plugin FROM mysql.user WHERE user ='raoinfotech';

# THIS WILL PRINT SOMTHING LIKE THIS WAY
+-------------+------+-----------------------+
| user        | host | plugin                |
+-------------+------+-----------------------+
| raoinfotech | %    | caching_sha2_password |
+-------------+------+-----------------------+

# ON MASTER SIDE
# ON THIS U CAN SEE THE PLUGIN IS `caching_sha2_password` CHNAGE TO `mysql_native_password` BY THIS QUERY ON MASTER
ALTER USER 'raoinfotech'@'%' IDENTIFIED WITH mysql_native_password BY 'raoinfotech';
FLUSH PRIVILEGES;

# RESTART SLAVE OR REPLICA(WINDOWS)
STOP SLAVE;
START SLAVE;

```

---

### Step 5: Test Replication
#### Insert Data into Master
```sql
USE employees;
INSERT INTO departments (dept_no, dept_name) VALUES ('d010', 'New Department');
```

#### Check Data in Slave
```sql
USE employees;
SELECT * FROM departments;
```

If the data appears, replication is working correctly!

---

### For GTID Information use this document :
- https://dev.mysql.com/doc/refman/8.4/en/replication-gtids-howto.html
