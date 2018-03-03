
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


## Roberto

## Configuración MYSQL Maestro


```console

alu5906@maestro2:~$ cd /etc/mysql/mysql.conf.d/
alu5906@maestro2:/etc/mysql/mysql.conf.d$ ls
mysqld.cnf  mysqld_safe_syslog.cnf
alu5906@maestro2:/etc/mysql/mysql.conf.d$

```

- Configuramos el fichero `mysql.conf`

```console

alu5906@maestro2:/etc/mysql/mysql.conf.d$ sudo cat mysqld.cnf
[sudo] password for alu5906:
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

# This will be passed to all mysql clients
# It has been reported that passwords should be enclosed with ticks/quotes
# escpecially if they contain "#" chars...
# Remember to edit /etc/mysql/debian.cnf when changing the socket location.

# Here is entries for some specific programs
# The following values assume you have at least 32M ram

[mysqld_safe]
socket		= /var/run/mysqld/mysqld.sock
nice		= 0

[mysqld]
#
# * Basic Settings
#
user		= mysql
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
port		= 3306
basedir		= /usr
datadir		= /var/lib/mysql
tmpdir		= /tmp
lc-messages-dir	= /usr/share/mysql
skip-external-locking
#bind-address		= 127.0.0.1
key_buffer_size		= 16M
max_allowed_packet	= 16M
thread_stack		= 192K
thread_cache_size       = 8
myisam-recover-options  = BACKUP
query_cache_limit	= 1M
query_cache_size        = 16M

# Configuración para el master

log_error = /var/log/mysql/error.log
server-id		= 1
log_bin			= /var/log/mysql/mysql-bin.log
expire_logs_days	= 10
max_binlog_size         = 100M
binlog_do_db		= cluster
binlog_ignore_db	= mysql
binlog_ignore_db	= test

```

- Reiniciamos el servicio.

```console
alu5906@maestro2:~$ sudo systemctl restart mysql.service
```

- Entramos a la base de datos de el maestro.

```console

alu5906@maestro2:/etc/mysql/mysql.conf.d$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.21-0ubuntu0.16.04.1-log (Ubuntu)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database cluster;
Query OK, 1 row affected (1,65 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'esclavo'@'172.18.22.2' IDENTIFIED BY '1234';
Query OK, 0 rows affected, 1 warning (0,00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (1,87 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      770 | cluster      | mysql,test       |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0,00 sec)

mysql>
```

## Configuración Mysql Slave


```console

alu5906@maestro2:~$ cd /etc/mysql/mysql.conf.d/
alu5906@maestro2:/etc/mysql/mysql.conf.d$ ls
mysqld.cnf  mysqld_safe_syslog.cnf
alu5906@maestro2:/etc/mysql/mysql.conf.d$

```

- Configuramos el fichero `mysql.conf`

```console

alu5906@maestro2:/etc/mysql/mysql.conf.d$ sudo cat mysqld.cnf
[sudo] password for alu5906:
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

# This will be passed to all mysql clients
# It has been reported that passwords should be enclosed with ticks/quotes
# escpecially if they contain "#" chars...
# Remember to edit /etc/mysql/debian.cnf when changing the socket location.

# Here is entries for some specific programs
# The following values assume you have at least 32M ram

[mysqld_safe]
socket		= /var/run/mysqld/mysqld.sock
nice		= 0

[mysqld]
#
# * Basic Settings
#
user		= mysql
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
port		= 3306
basedir		= /usr
datadir		= /var/lib/mysql
tmpdir		= /tmp
lc-messages-dir	= /usr/share/mysql
skip-external-locking
#bind-address		= 127.0.0.1
key_buffer_size		= 16M
max_allowed_packet	= 16M
thread_stack		= 192K
thread_cache_size       = 8
myisam-recover-options  = BACKUP
query_cache_limit	= 1M
query_cache_size        = 16M

# Configuración para el master

log_error = /var/log/mysql/error.log
server-id		= 2
log_bin			= /var/log/mysql/mysql-bin.log
expire_logs_days	= 10
max_binlog_size         = 100M
binlog_do_db		= cluster
binlog_ignore_db	= mysql
binlog_ignore_db	= test

```

- Reiniciamos el servicio.

```console
alu5906@maestro2:~$ sudo systemctl restart mysql.service
```

- Entramos a la base de datos

```console

alu5906@maestro2:~$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.21-0ubuntu0.16.04.1-log (Ubuntu)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE DATABASE cluster;
Query OK, 1 row affected (0,12 sec)

mysql> CHANGE MASTER TO MASTER_HOST='172.18.22.1', MASTER_USER='esclavo', MASTER_PASSWORD='1234', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=770;
Query OK, 0 rows affected, 2 warnings (0,01 sec)

mysql> START SLAVE;
Query OK, 0 rows affected (0,01 sec)

mysql> SHOW SLAVE STATUS \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.18.22.1
                  Master_User: esclavo
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 770
               Relay_Log_File: maestro2-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
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
          Exec_Master_Log_Pos: 770
              Relay_Log_Space: 530
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: 341c4457-1f11-11e8-a758-080027e6f90f
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
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

mysql>
```

## Vamos a crear una Tabla en la base de datos cluster En el Server Maestro

```console
mysql> use cluster;
Database changed
mysql> CREATE TABLE usuarios (id int(11) AUTO_INCREMENT, PRIMARY KEY (id)) Engine=MyISAM;
Query OK, 0 rows affected (0,00 sec)

mysql> show tables;
+-------------------+
| Tables_in_cluster |
+-------------------+
| usuarios          |
+-------------------+
1 row in set (0,00 sec)

mysql>
```

![](img/006.png)

- Metemos datos a la tabla de usuarios

![](img/007.png)

## Comprobamos en el servidor slave que tenemos creado la tabla en la base de datos cluster


![](img/005.png)

- Comprobamos los datos que estan en la tabla de usuarios

![](img/008.png)



## Configuración MASTER a Master

## Configuración MASTER2

Comprobamos con el comando show master status

![](img/009.png)

Para luego ir al master1 y darle permisos al usuario esclavo.

```console
mysql> GRANT REPLICATION SLAVE ON *.* TO 'esclavo'@'172.18.22.1' IDENTIFIED BY '1234';
Query OK, 0 rows affected, 1 warning (0,01 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0,01 sec)

mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      784 | cluster      | mysql,test       |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0,00 sec)
```

![](img/010.png)

- vamos al master1, tenemos que decirle cual es nuestro servidor maestro y con la ip y el usuario.

```console
mysql> CHANGE MASTER TO MASTER_HOST='172.18.22.2', MASTER_USER='esclavo', MASTER_PASSWORD='1234', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=784;
Query OK, 0 rows affected, 2 warnings (5,40 sec)

mysql> START SLAVE;
Query OK, 0 rows affected (0,00 sec)

mysql> SHOW SLAVE STATUS \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.18.22.2
                  Master_User: esclavo
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 784
               Relay_Log_File: maestro1-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
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
          Exec_Master_Log_Pos: 784
              Relay_Log_Space: 530
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 2
                  Master_UUID: ebd4fc8a-1f0f-11e8-9a5e-0800277e8971
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
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

mysql>

```

Entramos en la base de datos en el servidor master2 y ahora podemos insertar datos a la tabla de usuarios.


```console
mysql> INSERT INTO usuarios (id) VALUES(3);
Query OK, 1 row affected (0,00 sec)

mysql> select * from usuarios;
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
3 rows in set (0,00 sec)

mysql>
```
![](img/011.png)
