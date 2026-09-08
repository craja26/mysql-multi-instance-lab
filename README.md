GitHub repo architecture:
```
mysql-multi-instance-lab/
├── mysql_init
├── configs/
│   ├── finapp-my.cnf
│   └── hrapp-my.cnf
└── README.md
```
My plan is to create two MySQL instances in one Linux machine. Each one has a dedicated ip address and hostname but want to use default port.
I will update complete architecture later.

I am using `mysql_init` file to stop or start mysql instance.

## mysql02 - Percona Server installation
```Bash
sudo dnf install -y dnf-utils

sudo dnf module disable mysql -y

sudo dnf install -y \
  https://repo.percona.com/yum/percona-release-latest.noarch.rpm

sudo percona-release setup ps80
sudo percona-release enable pxb-80
```
Install Percona Server:
```Bash
sudo dnf install -y \
  percona-server-server \
  percona-server-client \
  percona-server-shared \
  percona-server-shared-compat \
  percona-icu-data-files
```
Verify:
```Bash
mysqld --version
```

## FINAPP directories
```Bash
sudo mkdir -p \
  /data/finapp/mysqldata \
  /data/finapp/log \
  /data/finapp/temp
```
Ownership:
```Bash
sudo chown -R mysql:mysql /data/finapp
```
SELinux:
```Bash
sudo semanage fcontext -a -t mysqld_db_t "/data/finapp/mysqldata(/.*)?"

sudo semanage fcontext -a -t mysqld_log_t "/data/finapp/log(/.*)?"

sudo semanage fcontext -a -t mysqld_var_run_t "/data/finapp/temp(/.*)?"

sudo restorecon -Rv /data/finapp
```

---
## FINAPP clone from mysql01
```Bash
mysql01 lo instance graceful ga stop:
```
Archive:
```Bash
sudo tar czpf /tmp/finapp.tar.gz -C /data finapp
```
Copy:
```Bash
sudo scp /tmp/finapp.tar.gz rockylinux@192.168.2.136:/tmp/
```
mysql02:
```Bash
sudo tar xzpf /tmp/finapp.tar.gz -C /data
```
Restore SELinux contexts:
```Bash
sudo restorecon -Rv /data/finapp
```
Important: cloned `auto.cnf` remove cheyyali:
```Bash
sudo rm -f /data/finapp/mysqldata/auto.cnf
```

---
## FINAPP config on mysql02
mysql01 config copy:
```Bash
sudo scp /etc/my.cnf.d/finapp/my.cnf \
rockylinux@192.168.2.136:/tmp/finapp.my.cnf
```
mysql02:
```Bash
sudo mkdir -p /etc/my.cnf.d/finapp

sudo mv /tmp/finapp.my.cnf \
  /etc/my.cnf.d/finapp/my.cnf

sudo chown root:root /etc/my.cnf.d/finapp/my.cnf
sudo chmod 644 /etc/my.cnf.d/finapp/my.cnf
```
Then change:
```Bash
server-id=201
bind-address=192.168.2.160
```
Validation:
```Bash
sudo mysqld \
  --defaults-file=/etc/my.cnf.d/finapp/my.cnf \
  --validate-config
```

---
## HRAPP
Same process:
```Bash
sudo mkdir -p \
  /data/hrapp/mysqldata \
  /data/hrapp/log \
  /data/hrapp/temp

sudo chown -R mysql:mysql /data/hrapp
```
SELinux:
```Bash
sudo semanage fcontext -a -t mysqld_db_t "/data/hrapp/mysqldata(/.*)?"

sudo semanage fcontext -a -t mysqld_log_t "/data/hrapp/log(/.*)?"

sudo semanage fcontext -a -t mysqld_var_run_t "/data/hrapp/temp(/.*)?"

sudo restorecon -Rv /data/hrapp
```
mysql01 HRAPP graceful stop:
```Bash
sudo /usr/local/bin/mysql_init stop hrapp
```
Archive:
```Bash
sudo tar czpf /tmp/hrapp.tar.gz -C /data hrapp
```
Copy:
```Bash
sudo scp /tmp/hrapp.tar.gz \
rockylinux@192.168.2.136:/tmp/
```
mysql02:
```Bash
sudo tar xzpf /tmp/hrapp.tar.gz -C /data
sudo restorecon -Rv /data/hrapp
```
Remove cloned UUID:
```Bash
sudo rm -f /data/hrapp/mysqldata/auto.cnf
```
Config:
```INI
server-id=202
port=3306
bind-address=192.168.2.161

mysqlx-port=33061
mysqlx-bind-address=192.168.2.161
```
Validation:
```Bash
sudo mysqld \
  --defaults-file=/etc/my.cnf.d/hrapp/my.cnf \
  --validate-config
```
`mysql_init` copy:
```Bash
sudo scp /usr/local/bin/mysql_init \
rockylinux@192.168.2.136:/tmp/mysql_init
```
Then:
```Bash
sudo mv /tmp/mysql_init /usr/local/bin/mysql_init
sudo chown root:root /usr/local/bin/mysql_init
sudo chmod 755 /usr/local/bin/mysql_init
sudo bash -n /usr/local/bin/mysql_init
```
Start:
```Bash
sudo /usr/local/bin/mysql_init start finapp
sudo /usr/local/bin/mysql_init start hrapp
```
Verify:
```Bash
sudo /usr/local/bin/mysql_init status finapp
sudo /usr/local/bin/mysql_init status hrapp
```

---






























































