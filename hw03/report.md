## Работаем на виртуальной машине Ubuntu 26.04 (ubuntu01) c postgresql 18


### Cоздаем тестовую базу, схему и таблицу

Проверка кластера
```sh 
user@ubuntu01:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
18  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-18-main.log
```

Подключаемся к postgres и создаем тестовые объекты
```sh
user@ubuntu01:~$ sudo -u postgres psql
[sudo: authenticate] Password:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table t1(c1 int);
CREATE TABLE
testdb=# insert into t1 values (1),(2),(3);
INSERT 0 3
testdb=# select * from t1;
 c1
----
  1
  2
  3
(3 rows)

```

Создаем роль readonly и выдаем соответствующие привилегии.  
Команда GRANT SELECT ON ALL TABLES IN SCHEMA действует только на таблицы, которые уже существуют в момент её выполнения.  
Чтобы права на новые таблицы выдавались автоматически, нужно настроить значения по умолчанию (default privileges) для схемы.  
Для этого используется команда ALTER DEFAULT PRIVILEGES
```sh
testdb=# create role readonly;
CREATE ROLE
testdb=# grant connect on database testdb to readonly;
GRANT
testdb=# grant usage on schema testnm to readonly; 
GRANT
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
```

Создаем пользователя testread и добавляем его в роль readonly
```sh
testdb=# create role testread with login password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
```

Подключаемся к БД testdb под пользователем testread и пытаемся сделать select
```sh
user@ubuntu01:~$ psql -d testdb -U testread
Password for user testread:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

testdb=> select * from t1;
ERROR:  permission denied for table t1
```
У нас нет прав на таблицу t1 в схеме public

### Работаем со схемой testnm

Создаем таблицу в схеме testnm
```sh
user@ubuntu01:~$ sudo -u postgres psql -d testdb
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

testdb=# create table testnm.t1(c1 int);
CREATE TABLE
testdb=# insert into testnm.t1 values (4),(5),(6);
INSERT 0 3
testdb=#
```

Проверяем что данные можно прочитать по пользователем testread
```sh
user@ubuntu01:~$ psql -d testdb -U testread
Password for user testread:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

testdb=> select * from testnm.t1;
 c1
----
  4
  5
  6
(3 rows)
```

Настраиваем search_path для пользователя testread, чтобы обращение шло только к таблицам в схеме testnm
```sh
user@ubuntu01:~$ sudo -u postgres psql -d testdb
[sudo: authenticate] Password:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

testdb=# alter role testread set search_path = testnm;
ALTER ROLE
```

Проверяем что обращение к t1 идет в схеме testnm
```sh
user@ubuntu01:~$ psql -d testdb -U testread
Password for user testread:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

testdb=> select * from t1;
 c1
----
  4
  5
  6
(3 rows)

testdb=> select * from testnm.t1;
 c1
----
  4
  5
  6
(3 rows)
```

Проверяем что testread не имеет прав на создание и изменение таблиц
```sh
user@ubuntu01:~$ psql -d testdb -U testread
Password for user testread:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

testdb=> create table t2(c1 int);
ERROR:  permission denied for schema testnm
LINE 1: create table t2(c1 int);
                     ^
testdb=> create table public.t2(c1 int);
ERROR:  permission denied for schema public
LINE 1: create table public.t2(c1 int);
                     ^
testdb=> insert into t1 values (10),(20),(30);
ERROR:  permission denied for table t1
testdb=> insert into public.t1 values (10),(20),(30);
ERROR:  permission denied for table t1
testdb=> insert into testnm.t1 values (10),(20),(30);
ERROR:  permission denied for table t1
testdb=> create table testnm.t2(c1 int);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 int);
```         
