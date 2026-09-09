# 1. Lab Architecture
two Rocky Linux 8.10 servers use chesam:
two Rocky Linux 8.10 servers use chesam:
| Server  |       FINAPP IP | Server-ID | Role    |
| ------- | --------------: | --------: | ------- |
| mysql01 | `192.168.2.150` |     `101` | Source  |
| mysql02 | `192.168.2.160` |     `201` | Replica |

FINAPP MySQL instances:
```
mysql01
  └── FINAPP
      └── 192.168.2.150:3306

mysql02
  └── FINAPP
      └── 192.168.2.160:3306
```
Data directory:
```
/data/finapp/mysqldata
```
Logs:
```
/data/finapp/log
```
Temporary/socket files:
```
Temporary/socket files:
```

---
# 2. MySQL Binary Logging Configuration
`my.cnf` lo replication kosam important settings add chesam.
```INI
[mysqld]

server-id=101
port=3306
bind-address=192.168.2.150

datadir=/data/finapp/mysqldata
socket=/data/finapp/temp/finapp.sock
pid-file=/data/finapp/temp/finapp.pid
log-error=/data/finapp/log/mysqld.log

log_bin=/data/finapp/mysqldata/binlog
binlog_format=ROW
sync_binlog=1
binlog_expire_logs_seconds=2592000

gtid_mode=ON
enforce_gtid_consistency=ON
```
Replica lo `server-id` different:
```INI
server-id=201
bind-address=192.168.2.160
```
**Important concepts**
`log_bin`
Binary logging enable chestundi.

Binary log contains database changes which can be used for:

- Replication
- Point-in-time recovery
- Auditing/troubleshooting

`binlog_format=ROW`
Statement ni kaakunda affected row changes ni record chestundi.
eplication environments lo ROW-based replication predictable and safer choice.

`sync_binlog=1`
Commit ayina binlog events disk ki sync cheyyadaniki durability improve chestundi.

`binlog_expire_logs_seconds=2592000`
30 days approximately binlogs retain cheyyadaniki. We can modify it as per our environment
Validate changes:
```Bash
sudo mysqld \
  --defaults-file=/etc/my.cnf.d/finapp/my.cnf \
  --validate-config
```

---
# 3. Verify Binary Logging
Useful checks:
```SQL
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'log_bin_basename';
SHOW BINARY LOGS;
SHOW MASTER STATUS\G
```
Example:
```
File: binlog.000015
Position: 754
```
GTID enabled environment lo `SHOW MASTER STATUS` lo:
```
Executed_Gtid_Set
```
This is also important.

---
# 4. GTID
GTID = Global Transaction Identifier
Example source server UUID:
```
a6154c52-aad8-11f1-b425-0800279d3d35
```
Example GTID:
```
a6154c52-aad8-11f1-b425-0800279d3d35:10
```
Meaning:
```
Server UUID : Transaction number
```
GTID advantages

Traditional replication:
```
binlog file + position
```
GTID replication:
```
source UUID + transaction number
```
Check the position automatically:
```
SOURCE_AUTO_POSITION=1
```

---
# 5. XtraBackup Installation
Percona Server: `8.0.46-37`
XtraBackup: `8.0.35-36`
Verify: `xtrabackup --version`
Output: `xtrabackup version 8.0.35-36`

---
# 6. XtraBackup User
FINAPP source lo dedicated backup user create chesam:
```SQL
CREATE USER 'xtrabackup'@'localhost'
IDENTIFIED BY 'XtraBackupLab#2026';
```
Privileges:
```SQL
GRANT BACKUP_ADMIN,
      PROCESS,
      RELOAD,
      LOCK TABLES,
      REPLICATION CLIENT
ON *.* TO 'xtrabackup'@'localhost';

GRANT SELECT
ON performance_schema.keyring_component_status
TO 'xtrabackup'@'localhost';

GRANT SELECT
ON performance_schema.replication_group_members
TO 'xtrabackup'@'localhost';

GRANT SELECT
ON performance_schema.log_status
TO 'xtrabackup'@'localhost';
```

---
# 7. Online XtraBackup
Source mysql01 lo:
```Bash
sudo xtrabackup \
  --defaults-file=/etc/my.cnf.d/finapp/my.cnf \
  --user=xtrabackup \
  --password='XtraBackupLab#2026' \
  --socket=/data/finapp/temp/finapp.sock \
  --backup \
  --target-dir=/backup/finapp/full_20260908_192915
```
Backup successfully created:
```
/backup/finapp/full_20260908_192915
```

---
# 8. xtrabackup_binlog_info
Backup directory lo:
```Bash
cat /backup/finapp/full_20260908_192915/xtrabackup_binlog_info
```
Manaki:
```
binlog.000015  197  a6154c52-aad8-11f1-b425-0800279d3d35:1-5
```
Important observation:
Backup capture ayye time ki source transactions `1-5` varaku unnayi.
Backup taruvata source lo transactions `6-7` generate ayyayi.
Anduke GTID auto-position use chesinappudu replica:
```
Already has: 1-5
Needs:       6-7
```
ani automatically identify chesindi.
Idi GTID + XtraBackup combination yokka excellent practical example.

---
# 9. XtraBackup Prepare
Restore mundu backup prepare chestam:
```Bash
sudo xtrabackup \
  --prepare \
  --target-dir=/backup/finapp/full_20260908_192915
```
Successful: `backup_type=full-prepared`
Prepare phase:

- redo logs apply chestundi
- uncommitted transactions rollback chestundi
- backup ni restore-ready state ki teesukostundi

Hot backup kabatti prepare time lo:
```
Database was not shutdown normally! Starting crash recovery.
```
ani message vachindi.

Idi corruption kaadu. Hot backup prepare process lo normal behavior.

---
# 10. Transfer Backup to Replica
mysql01 → mysql02:
```Bash
sudo rsync -avh --progress \
  /backup/finapp/full_20260908_192915/ \
  rockylinux@192.168.2.136:/backup/finapp/full_20260908_192915/
```

---
# 11. Restore on mysql02
Replica instance gracefully stop:
```Bash
sudo /usr/local/bin/mysql_init stop finapp
```
Old datadir preserve chesam:
```
/data/finapp/mysqldata_before_replication_20260909
```
Then:
```Bash
sudo xtrabackup \
  --copy-back \
  --datadir=/data/finapp/mysqldata \
  --target-dir=/backup/finapp/full_20260908_192915
```
Ownership:
```Bash
sudo chown -R mysql:mysql /data/finapp/mysqldata
```
SELinux:
```Bash
sudo chown -R mysql:mysql /data/finapp/mysqldata
```

---
# 12. Replica UUID
Backup source `auto.cnf` source UUID contain cheyyachu.
Replica ki unique *UUID* undali.
Mana existing mysql02 UUID: `01d3d93e-ab0a-11f1-82d0-08002725500c`
Existing replica `auto.cnf` preserve chesam:
```INI
[auto]
server-uuid=01d3d93e-ab0a-11f1-82d0-08002725500c
```
Important rule:
| Source and replica ki same server UUID undakoodadu.

---
# 13. GTID Enable on Replica
Restore taruvata replica lo GTID initially OFF.
Safe staged transition:
```SQL
SET GLOBAL enforce_gtid_consistency = ON;

SET GLOBAL gtid_mode = OFF_PERMISSIVE;

SET GLOBAL gtid_mode = ON_PERMISSIVE;

SHOW STATUS
LIKE 'Ongoing_anonymous_gtid_violating_transactions';

SET GLOBAL gtid_mode = ON;
```
Persistent config:
```INI
gtid_mode=ON
enforce_gtid_consistency=ON
```

---
# 14. Replication User
Source mysql01 lo:
```SQL
CREATE USER 'repl'@'192.168.2.160'
IDENTIFIED BY 'ReplLab#2026';
```
Grant:
```SQL
GRANT REPLICATION SLAVE
ON *.*
TO 'repl'@'192.168.2.160';
```
Verify:
```SQL
SHOW GRANTS FOR 'repl'@'192.168.2.160';
```

---
# 15. Firewall
Source mysql01 lo only replica IP ki 3306 allow chesam:
```Bash
sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="192.168.2.160" port protocol="tcp" port="3306" accept'

sudo firewall-cmd --reload
```
Verify:
```Bash
sudo firewall-cmd --list-rich-rules
```
From mysql02:
```Bash
nc -zv 192.168.2.150 3306
```
Successful connection vachindi.

---
# 16. GTID Auto-Position Configuration
mysql02:
```SQL
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='192.168.2.150',
    SOURCE_PORT=3306,
    SOURCE_USER='repl',
    SOURCE_PASSWORD='ReplLab#2026',
    SOURCE_AUTO_POSITION=1;
```
`SOURCE_AUTO_POSITION=1` means:
| GTID-based automatic positioning enabled.
File/position manually specify cheyyalsina requirement ledu.

---
# 17. Authentication Problem
Initial `START REPLICA` taruvata:
```
Replica_IO_Running: Connecting
Replica_SQL_Running: Yes
```
Error:
```
Authentication requires secure connection
```
Reason:
MySQL 8 default authentication plugin `caching_sha2_password`.
Lab lo fix:
```SQL
STOP REPLICA;

CHANGE REPLICATION SOURCE TO
    GET_SOURCE_PUBLIC_KEY=1;

START REPLICA;
```
Then replication healthy:
```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Last_IO_Errno: 0
Last_SQL_Errno: 0
Seconds_Behind_Source: 0
```
Production note: `GET_SOURCE_PUBLIC_KEY=1` lab convenience. Production lo replication connection ki TLS/SSL use cheyyadam better.

---
# 18. First Successful Replication
Backup captured: `GTID 1-5`
Source later had: `GTID 1-7`
Replica status:
```
Retrieved_Gtid_Set:
a6154c52-aad8-11f1-b425-0800279d3d35:6-7

Executed_Gtid_Set:
a6154c52-aad8-11f1-b425-0800279d3d35:1-7
```
This proved that replica automatically fetched only missing transactions.

---
# 19. INSERT Replication Test
mysql01:
```SQL
INSERT INTO customers
(customer_id, customer_name, email)
VALUES
(4, 'Kiran', 'kiran@example.com');
```
mysql02 lo:
```SQL
SELECT * FROM customers
ORDER BY customer_id;
```
`Kiran` row successfully appeared. INSERT replication confirmed.

---
# 20. UPDATE Replication Test
mysql01:
```SQL
UPDATE customers
SET email = 'kiran.updated@example.com'
WHERE customer_id = 4;
```
mysql02:
```SQL
SELECT *
FROM customers
WHERE customer_id = 4;
```
Result: `kiran.updated@example.com`
UPDATE replication confirmed.

---
# 21. Replication Failure — Duplicate Key
Intentional conflict create cheddam.
mysql02:
```SQL
INSERT INTO customers
(customer_id, customer_name, email)
VALUES
(5, 'Priya', 'priya@example.com');
```
Then mysql01 lo same:
```SQL
INSERT INTO customers
(customer_id, customer_name, email)
VALUES
(5, 'Priya', 'priya@example.com');
```
mysql01 insert successful.
mysql02 replication failed:
```
Last_SQL_Errno: 1062
```
And:
```
Replica_IO_Running: Yes
Replica_SQL_Running: No
```
Meaning:
```
IO thread  → receiving binlog → working
SQL thread → applying changes → stopped
```
Detailed worker error:
```
Duplicate entry '5'
for key 'customers.PRIMARY'
```

---
# 22. Correct Recovery of Duplicate-Key Conflict
First conflicting row on replica identify chesam.
mysql02:
```SQL
DELETE FROM finapp_test.customers
WHERE customer_id = 5;
```
Verify:
```SQL
SELECT *
FROM finapp_test.customers
WHERE customer_id = 5;
```
Result:`Empty set`
Then:
```SQL
START REPLICA;
```
Verify:
```SQL
SHOW REPLICA STATUS\G
```
Healthy:
```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Last_SQL_Errno: 0
Last_SQL_Error:
Seconds_Behind_Source: 0
```
And GTID: `...:1-10`
Original source transaction successfully applied.
Preferred recovery principle:
| Fix the underlying data conflict and allow the original transaction to execute whenever possible.

---
# 23. Deliberate Transaction Skip Test
Then intentionally another conflict create chesam.
mysql02:
```SQL
INSERT INTO customers
(customer_id, customer_name, email)
VALUES
(6, 'RaviSkip', 'raviskip@example.com');
```
mysql01:
```SQL
INSERT INTO customers
(customer_id, customer_name, email)
VALUES
(6, 'RaviSource', 'ravisource@example.com');
```
Replication stopped with: `Last_SQL_Errno: 1062`
Failed GTID: `a6154c52-aad8-11f1-b425-0800279d3d35:11`

---
# 24. GTID-Aware Transaction Skip
Replica stop:
```SQL
STOP REPLICA;
```
Then intentionally empty transaction:
```SQL
SET GTID_NEXT='a6154c52-aad8-11f1-b425-0800279d3d35:11';

BEGIN;
COMMIT;

SET GTID_NEXT='AUTOMATIC';
```
Then:
```SQL
START REPLICA;
```
Verify:
```SQL
SHOW REPLICA STATUS\G
```
Result:
```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Last_SQL_Errno: 0
Last_SQL_Error:
Seconds_Behind_Source: 0
```
And:
```
Executed_Gtid_Set:
...:1-11
```
So replication resumed successfully.

---
# 25. But the Dangerous Part — Data Divergence
mysql01:
```
customer_id = 6
customer_name = RaviSource
email = ravisource@example.com
```
mysql02:
```
customer_id = 6
customer_name = RaviSkip
email = raviskip@example.com
```
Yet replication showed:
```
IO = Yes
SQL = Yes
Lag = 0
Error = 0
```
This is the most important lesson.
| Healthy replication status does not always mean identical data.
Because GTID 11 was marked as executed using an empty transaction, the source transaction was skipped and the data remained different.

---
# 26. Production DBA Decision — Which Method Is Best?
My recommended order:
🥇 1. Fix the conflict and let replication continue
Best option whenever possible.
```
Find root cause
      ↓
Correct replica data
      ↓
START REPLICA
      ↓
Original transaction executes
      ↓
Verify consistency
```
🥈 2. Re-seed replica
If replica has significant divergence/corruption or many replication problems:
```
Fresh XtraBackup
      ↓
Prepare
      ↓
Restore
      ↓
Configure GTID replication
      ↓
Verify
```
Mana lab lo idi already practice chesam.
🥉 3. Skip transaction — last resort
GTID-aware empty transaction method:
```SQL
SET GTID_NEXT='source_uuid:transaction_id';
BEGIN;
COMMIT;
SET GTID_NEXT='AUTOMATIC';
```
Use only when you understand exactly what transaction you're skipping.
Because:
```
Replication continuity
        ≠
Data consistency
```

---
# Today's Lab — Final Checklist ✅
```
☑ Multiple MySQL instances
☑ Custom my.cnf
☑ Binary logging
☑ ROW binlog format
☑ sync_binlog
☑ Binlog retention
☑ GTID
☑ XtraBackup installation
☑ XtraBackup user
☑ Online backup
☑ xtrabackup_binlog_info
☑ Backup prepare
☑ Backup transfer
☑ XtraBackup restore
☑ Ownership + SELinux
☑ Replica UUID
☑ GTID enablement
☑ Replication user
☑ Firewall restriction
☑ GTID auto-position
☑ Authentication troubleshooting
☑ INSERT replication
☑ UPDATE replication
☑ Duplicate-key replication failure
☑ Worker-level error diagnosis
☑ Safe conflict recovery
☑ GTID transaction skip
☑ Data divergence demonstration
```
