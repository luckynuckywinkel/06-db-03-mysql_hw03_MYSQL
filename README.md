# Домашнее задание к занятию 3. «MySQL»


## Задача 1

Используя Docker, поднимите инстанс MySQL (версию 8). Данные БД сохраните в volume.

Изучите [бэкап БД](https://github.com/netology-code/virt-homeworks/tree/virt-11/06-db-03-mysql/test_data) и 
восстановитесь из него.

Перейдите в управляющую консоль `mysql` внутри контейнера.

Используя команду `\h`, получите список управляющих команд.

Найдите команду для выдачи статуса БД и **приведите в ответе** из её вывода версию сервера БД.

Подключитесь к восстановленной БД и получите список таблиц из этой БД.

**Приведите в ответе** количество записей с `price` > 300.

В следующих заданиях мы будем продолжать работу с этим контейнером.  

### Решение: 

- Поднимем виртуалку используя Vagrant и Ansibl'ом установим все необходимые компоненты, также, как мы делали в [предыдущей работе](https://github.com/luckynuckywinkel/06-db-02-sql_hw02_PGSQL1/blob/main/ansible/main.yaml)

- **docker-compose.yaml**:


```
version: "3"

services:
  mysql_db:
    image: mysql:8
    container_name: mysql8
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: test_db
      MYSQL_USER: admin
      MYSQL_PASSWORD: strongpassword
    volumes:
      - ./dbdata:/var/lib/mysql/
      - ./backup:/tmp/mysql/backup
```

- зайдем в контейнер, затем в mysql и посмотрим вывод команды **\h**, сразу же найдем нужную нам команду и посмотрим версию сервера:

```
mysql> \h

For information about MySQL products and services, visit:
   http://www.mysql.com/
For developer information, including the MySQL Reference Manual, visit:
   http://dev.mysql.com/
To buy MySQL Enterprise support, training, or other products, visit:
   https://shop.mysql.com/

List of all MySQL commands:
Note that all text commands must be first on line and end with ';'
?         (\?) Synonym for `help'.
clear     (\c) Clear the current input statement.
connect   (\r) Reconnect to the server. Optional arguments are db and host.
delimiter (\d) Set statement delimiter.
edit      (\e) Edit command with $EDITOR.
ego       (\G) Send command to mysql server, display result vertically.
exit      (\q) Exit mysql. Same as quit.
go        (\g) Send command to mysql server.
help      (\h) Display this help.
nopager   (\n) Disable pager, print to stdout.
notee     (\t) Don't write into outfile.
pager     (\P) Set PAGER [to_pager]. Print the query results via PAGER.
print     (\p) Print current command.
prompt    (\R) Change your mysql prompt.
quit      (\q) Quit mysql.
rehash    (\#) Rebuild completion hash.
source    (\.) Execute an SQL script file. Takes a file name as an argument.
status    (\s) Get status information from the server.
system    (\!) Execute a system shell command.
tee       (\T) Set outfile [to_outfile]. Append everything into given outfile.
use       (\u) Use another database. Takes database name as argument.
charset   (\C) Switch to another charset. Might be needed for processing binlog with multi-byte charsets.
warnings  (\W) Show warnings after every statement.
nowarning (\w) Don't show warnings after every statement.
resetconnection(\x) Clean session context.
query_attributes Sets string parameters (name1 value1 name2 value2 ...) for the next query to pick up.
ssl_session_data_print Serializes the current SSL session data to stdout or file

For server side help, type 'help contents'

mysql> status;
--------------
mysql  Ver 8.1.0 for Linux on x86_64 (MySQL Community Server - GPL)

Connection id:          13
Current database:
Current user:           admin@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         8.1.0 MySQL Community Server - GPL  <---- Версия сервера
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    latin1
Conn.  characterset:    latin1
UNIX socket:            /var/run/mysqld/mysqld.sock
Binary data as:         Hexadecimal
Uptime:                 17 min 54 sec

Threads: 2  Questions: 61  Slow queries: 0  Opens: 161  Flush tables: 3  Open tables: 79  Queries per second avg: 0.056
--------------
```

- А, забыл. Я скачал дамп нужной базы, засунул его в волум "backup" и залил в созданную пустую базу test_db командой *mysql -u admin -p test_db < /tmp/mysql/backup/test_dump.sql*

- Зайдем в базу и посмотрим таблицы (не густо...):

```
mysql> use test_db
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-------------------+
| Tables_in_test_db |
+-------------------+
| orders            |
+-------------------+
1 row in set (0.01 sec)
```

- Посмотрим, что стоит более 300, видим всего одну запись и я не смог ее не просмотреть:

```
mysql> select count(*) from orders where price > 300;
+----------+
| count(*) |
+----------+
|        1 |
+----------+
1 row in set (0.01 sec)

mysql> select * from orders where price > 300;
+----+----------------+-------+
| id | title          | price |
+----+----------------+-------+
|  2 | My little pony |   500 |
+----+----------------+-------+
1 row in set (0.00 sec)
```

---





## Задача 2

Создайте пользователя test в БД c паролем test-pass, используя:

- плагин авторизации mysql_native_password
- срок истечения пароля — 180 дней 
- количество попыток авторизации — 3 
- максимальное количество запросов в час — 100
- аттрибуты пользователя:
    - Фамилия "Pretty"
    - Имя "James".

Предоставьте привелегии пользователю `test` на операции SELECT базы `test_db`.
    
Используя таблицу INFORMATION_SCHEMA.USER_ATTRIBUTES, получите данные по пользователю `test` и 
**приведите в ответе к задаче**.  

### Решение:  

- Прежде чем начать создавать нового пользователя и задавать ему различные параметры, выдвдим пользователю admin все необходимые для этого права (зайдя под рутом):

```
mysql> SELECT user, host FROM mysql.user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| admin            | %         |
| root             | %         |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+
6 rows in set (0.00 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)
```

- Продолжим под админом:

```
bash-4.4# mysql -u admin -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 18
Server version: 8.1.0 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'test'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY 'test-pass';
Query OK, 0 rows affected (0.01 sec)

mysql> ALTER USER 'test'@'localhost' PASSWORD EXPIRE INTERVAL 180 DAY;
Query OK, 0 rows affected (0.01 sec)

mysql> ALTER USER 'test'@'localhost' FAILED_LOGIN_ATTEMPTS 3;
Query OK, 0 rows affected (0.00 sec)

mysql> ALTER USER 'test'@'localhost' WITH MAX_QUERIES_PER_HOUR 100;
Query OK, 0 rows affected (0.00 sec)

mysql> ALTER USER 'test'@'localhost' ATTRIBUTE '{"Lastname": "Pretty", "Firstname": "James"}';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT SELECT ON test_db.* TO 'test'@'localhost';
Query OK, 0 rows affected, 1 warning (0.01 sec)
```

- Получим данные по юзеру **test**:

```
mysql> SELECT * FROM INFORMATION_SCHEMA.USER_ATTRIBUTES where user='test';
+------+-----------+----------------------------------------------+
| USER | HOST      | ATTRIBUTE                                    |
+------+-----------+----------------------------------------------+
| test | localhost | {"Lastname": "Pretty", "Firstname": "James"} |
+------+-----------+----------------------------------------------+
1 row in set (0.00 sec)
```

---



## Задача 3

Установите профилирование `SET profiling = 1`.
Изучите вывод профилирования команд `SHOW PROFILES;`.

Исследуйте, какой `engine` используется в таблице БД `test_db` и **приведите в ответе**.

Измените `engine` и **приведите время выполнения и запрос на изменения из профайлера в ответе**:
- на `MyISAM`,
- на `InnoDB`.

### Решение:   

- Сделаем необходимые действия, в качестве запроса используем уже используемый ранее запрос:

```
mysql> SET profiling = 1;
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> SHOW PROFILES;
Empty set, 1 warning (0.00 sec)

mysql> use test_db
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> SELECT COUNT(*) FROM orders WHERE price > 300;
+----------+
| COUNT(*) |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)

mysql> SHOW PROFILES;
+----------+------------+-----------------------------------------------+
| Query_ID | Duration   | Query                                         |
+----------+------------+-----------------------------------------------+
|        1 | 0.00025225 | SELECT DATABASE()                             |
|        2 | 0.00142675 | show databases                                |
|        3 | 0.00188725 | show tables                                   |
|        4 | 0.00063400 | SELECT COUNT(*) FROM orders WHERE price > 300 |
+----------+------------+-----------------------------------------------+
4 rows in set, 1 warning (0.00 sec)
```

- Посмотрим, какой используется движок:

```
mysql> SHOW TABLE STATUS;
+--------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| Name   | Engine | Version | Row_format | Rows | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time         | Update_time         | Check_time | Collation          | Checksum | Create_options | Comment |
+--------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| orders | InnoDB |      10 | Dynamic    |    5 |           3276 |       16384 |               0 |            0 |         0 |              6 | 2023-10-23 09:48:28 | 2023-10-23 09:48:28 | NULL       | utf8mb4_0900_ai_ci |     NULL |                |         |
+--------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
1 row in set (0.01 sec)
```

- Поменяем движок на таблице orders (да, сначала я пытался менять прямо на базу, что не правильно) на MyISAM и выполним тот же запрос, запрос занял меньше времени:

```
mysql> show tables;
+-------------------+
| Tables_in_test_db |
+-------------------+
| orders            |
+-------------------+
1 row in set (0.01 sec)

mysql> ALTER TABLE test_db ENGINE = MyISAM;
ERROR 1146 (42S02): Table 'test_db.test_db' doesn't exist
mysql> ALTER TABLE orders ENGINE = MyISAM;
Query OK, 5 rows affected (0.05 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> SELECT COUNT(*) FROM orders WHERE price > 300;
+----------+
| COUNT(*) |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)

mysql> SHOW PROFILES;
+----------+------------+-----------------------------------------------+
| Query_ID | Duration   | Query                                         |
+----------+------------+-----------------------------------------------+
|        7 | 0.00607175 | SHOW TABLE STATUS                             |
|        8 | 0.00065250 | ALTER TABLE test_db ENGINE = MyISAM           |
|        9 | 0.00008275 | ALTER TABLE ENGINE = MyISAM                   |
|       10 | 0.00016950 | SELECT DATABASE()                             |
|       11 | 0.00108800 | show databases                                |
|       12 | 0.00422900 | show tables                                   |
|       13 | 0.00129150 | ALTER TABLE test_db ENGINE = MyISAM           |
|       14 | 0.00019250 | SELECT DATABASE()                             |
|       15 | 0.00111500 | show databases                                |
|       16 | 0.00207250 | show tables                                   |
|       17 | 0.00132550 | ALTER TABLE test_db ENGINE = MyISAM           |
|       18 | 0.00218600 | show tables                                   |
|       19 | 0.00097625 | ALTER TABLE test_db ENGINE = MyISAM           |
|       20 | 0.05795550 | ALTER TABLE orders ENGINE = MyISAM            |
|       21 | 0.00039650 | SELECT COUNT(*) FROM orders WHERE price > 300 |
+----------+------------+-----------------------------------------------+
15 rows in set, 1 warning (0.00 sec)
```

---





## Задача 4 

Изучите файл `my.cnf` в директории /etc/mysql.

Измените его согласно ТЗ (движок InnoDB):

- скорость IO важнее сохранности данных;
- нужна компрессия таблиц для экономии места на диске;
- размер буффера с незакомиченными транзакциями 1 Мб;
- буффер кеширования 30% от ОЗУ;
- размер файла логов операций 100 Мб.

Приведите в ответе изменённый файл `my.cnf`.  

### Решение:  

- После долгих поисков и манипуляций, нескольких заваливаний моего контейнера с mysql, вот мои параметры в my.cnf:

```
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.1/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M

# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.1/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
secure-file-priv=/var/lib/mysql-files
user=mysql

pid-file=/var/run/mysqld/mysqld.pid

innodb_flush_log_at_trx_commit = 2  <--- как часто буферы журнала транзакций должны быть сброшены на диск. 2 - компромисс, между 0 и 1(дефолт).
innodb_file_per_table = 1 <--- каждая таблица InnoDB будет храниться в отдельном файле
innodb_log_buffer_size = 1M
innodb_buffer_pool_size = 2576980377 <--- цифровое значение 30% от 8Гб РАМа
innodb_log_file_size = 100M

[client]
socket=/var/run/mysqld/mysqld.sock

!includedir /etc/mysql/conf.d/
```




---


