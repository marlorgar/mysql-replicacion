
## 1. Configuración de Maestro

```console
alu5906@maestro:~$ cd /etc/mysql/
alu5906@maestro:/etc/mysql$ ls
conf.d        fabric.cfg       my.cnf.wba.bak  mysql-fabric-doctrine-1.4.0.zip
debian.cnf    my.cnf           mysql.cnf
debian-start  my.cnf.fallback  mysql.conf.d
alu5906@maestro:/etc/mysql$ sudo nano my.cnf
[sudo] password for alu5906:
alu5906@maestro:/etc/mysql$ sudo nano my.cnf
alu5906@maestro:/etc/mysql$ sudo cat my.cnf
#
# The MySQL database server configuration file.
#
# You can copy this to one of:
# - "/etc/mysql/my.cnf" to set global options,
# - "~/.my.cnf" to set user-specific options.
#
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

#
# * IMPORTANT: Additional settings that can override those from this file!
#   The files must end with '.cnf', otherwise they'll be ignored.
#

!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/

[mysqld]
bind-address = 0.0.0.0
skip-external-locking
lc-messages = es_ES
language = "spanish"

port = 3306

...
...

# Configuración Maestro

log-bin=mysql-bin
server-id=1
binlog-ignore-db=mysql
binlog-ignore-db=test
alu5906@maestro:/etc/mysql$
```

- Creamos un usuario con permisos de replicación.

~~~console
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%' IDENTIFIED BY '78619841e';
Query OK, 0 rows affected, 1 warning (0,00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0,00 sec)

mysql> show grants for replication;
+-----------------------------------------------------+
| Grants for replication@%                            |
+-----------------------------------------------------+
| GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%' |
+-----------------------------------------------------+
1 row in set (0,00 sec)

mysql>


~~~

- Creamos el fichero dump.

~~~console
mysql> use jardineria;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0,00 sec)

mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      154 | jardineria   | mysql,test       |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0,00 sec)

mysql> ^DBye
alu5906@maestro:~$ mysqldump -u root -p --opt jardineria > mysqldump.sql
Enter password:
alu5906@maestro:~$ ls | grep mysql
mysqldump.sql
mysqld-verbose-help.txt
alu5906@maestro:~$



~~~

- Volvemos a desbloquear la escritura en las tablas.

~~~console
mysql> UNLOCK TABLES;
Query OK, 0 rows affected (0,00 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'replication'@'%' IDENTIFIED BY '78619841e' WITH GRANT OPTION;
Query OK, 0 rows affected, 1 warning (0,01 sec)

~~~
## 2. Configuración esclavo

```console
alu5906@esclavo:~$ cd /etc/mysql/
alu5906@esclavo:/etc/mysql$ ls
conf.d        fabric.cfg       my.cnf.wba.bak  mysql-fabric-doctrine-1.4.0.zip
debian.cnf    my.cnf           mysql.cnf
debian-start  my.cnf.fallback  mysql.conf.d
alu5906@esclavo:/etc/mysql$ cd mysql.conf.d/
alu5906@esclavo:/etc/mysql/mysql.conf.d$ ls
mysqld.cnf  mysqld_safe_syslog.cnf
alu5906@esclavo:/etc/mysql/mysql.conf.d$ cd ..
alu5906@esclavo:/etc/mysql$ sudo nano my.cnf
[sudo] password for alu5906:
alu5906@esclavo:/etc/mysql$ cat my.cnf
#
# The MySQL database server configuration file.
#
# You can copy this to one of:
# - "/etc/mysql/my.cnf" to set global options,
# - "~/.my.cnf" to set user-specific options.
#
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

#
# * IMPORTANT: Additional settings that can override those from this file!
#   The files must end with '.cnf', otherwise they'll be ignored.
#

!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/

[mysqld]
bind-address = 0.0.0.0
skip-external-locking
lc-messages = es_ES
language = "spanish"

port = 3306

...
...

# Configuración Esclavo

server-id=2
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = jardineria
relay-log = /var/log/mysql/mysql-relay-bin.log  

alu5906@esclavo:/etc/mysql$
```

Debemos reiniciar el servicio

```console
alu5906@esclavo:/etc/mysql$ sudo systemctl restart mysql
alu5906@esclavo:/etc/mysql$ sudo systemctl status mysql
● mysql.service - MySQL Community Server
   Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: en
   Active: active (running) since vie 2018-03-02 19:10:16 WET; 6s ago
  Process: 2389 ExecStartPost=/usr/share/mysql/mysql-systemd-start post (code=ex
  Process: 2380 ExecStartPre=/usr/share/mysql/mysql-systemd-start pre (code=exit
 Main PID: 2388 (mysqld)
   CGroup: /system.slice/mysql.service
           └─2388 /usr/sbin/mysqld

mar 02 19:10:15 esclavo systemd[1]: Stopped MySQL Community Server.
mar 02 19:10:15 esclavo systemd[1]: Starting MySQL Community Server...
mar 02 19:10:16 esclavo systemd[1]: Started MySQL Community Server.
```

- Vamos a usar `mysqldump` para crear la base de datos jardineria en el esclavo.

~~~console
mysql> create database jardineria;
Query OK, 1 row affected (0,00 sec)


...

alu5906@esclavo:~$ mysql -u root -p jardineria < mysqldump.sql;
Enter password:
alu5906@esclavo:~$


...


mysql> use jardineria
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------------+
| Tables_in_jardineria |
+----------------------+
| Clientes             |
| Empleados            |
| GamasProductos       |
| Oficinas             |
| Pedidos              |
| vista1               |
+----------------------+
6 rows in set (0,00 sec)

mysql>

~~~

## 3. Replicando el esclavo

- MASTER_HOST=’192.168.0.18′: ip del servidor maestro (master).
- MASTER_USER=’xulescode’: nombre del usuario utilizado para la sincronización.
- MASTER_PASSWORD=’xulescode’: clave para el usuario definido.
- MASTER_LOG_FILE=’mysql-bin.000001′: fichero log que hemos obtenido al hacer la copia y consultar el servidor maestro (master).
- MASTER_LOG_POS= 402: posición de inicio de la sincronización, que hemos obtenido del maestro al igual que el fichero.

- Vamos a ejecutar el siguiente comando para cambiar el master de nuestro esclavo.

~~~console
mysql> CHANGE MASTER TO MASTER_HOST='172.18.22.1',MASTER_USER='replication', MASTER_PASSWORD='78619841e', MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=154;
Query OK, 0 rows affected, 2 warnings (0,00 sec)

mysql>
~~~

- Por último iniciamos y comprobamos que funciona correctamente el servidor slave.

~~~console
mysql> START SLAVE;
Query OK, 0 rows affected (5,34 sec)

mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 172.18.22.1
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 154
               Relay_Log_File: mysql-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: No
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 154
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 1593
                Last_IO_Error: Fatal error: The slave I/O thread stops because master and slave have equal MySQL server UUIDs; these UUIDs must be different for replication to work.
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID:
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp: 180302 20:26:55
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0,00 sec)

ERROR:
No query specified

mysql>

~~~
