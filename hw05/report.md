## Работаем на виртуальной машине Ubuntu 26.04 (ubuntu01) c postgresql 18


### Cоздаем тестовую базу, схему и таблицу

Подключаемся к postgres и создаем тестовые объекты
Отключаем AUTOCOMMIT.

Сессия 1
```sh
root@ubuntu01:~# sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# create database hw05;
CREATE DATABASE
postgres=# \c hw05
You are now connected to database "hw05" as user "postgres".
hw05=# \set AUTOCOMMIT off
hw05=# \echo :AUTOCOMMIT
off
hw05=# create table orders(id serial, created_at timestamptz, amount numeric);
CREATE TABLE
hw05=# select current_setting('transaction_isolation');
 current_setting
-----------------
 read committed
(1 row)
```

Сессия 2
```sh
root@ubuntu01:~# sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

hw05=# \set AUTOCOMMIT off
hw05=# \echo :AUTOCOMMIT
off
You are now connected to database "hw05" as user "postgres".
hw05=# select current_setting('transaction_isolation');
 current_setting
-----------------
 read committed
(1 row)

hw05=# select count(*), sum(amount) from orders where created_at >= current_timestamp - interval '1 minute';
ERROR:  relation "orders" does not exist

hw05=#
```

В первой сессии выполним commit, что бы таблица стала видна во второй. 
Во второй сессии в транзакции возникла ошибка - поэтому она завершится откатом в любом случае. 
```sh
hw05=!# select count(*), sum(amount) from orders where created_at >= current_timestamp - interval '1 minute';
ERROR:  current transaction is aborted, commands ignored until end of transaction block
hw05=!# commit;
ROLLBACK
hw05=# select count(*), sum(amount) from orders where created_at >= current_timestamp - interval '1 minute';
 count | sum
-------+-----
     0 |
(1 row)

```

### Тесты с изоляцией read committed

Вставим 2 строки в первой сессии
```sh
hw05=# insert into orders(created_at, amount) values(current_timestamp, 100);
INSERT 0 1
hw05=*# insert into orders(created_at, amount) values(current_timestamp, 100);
INSERT 0 1
```

Выполним селект во второй - записей нет
```sh
hw05=*# select count(*), sum(amount) from orders where created_at >= current_timestamp - interval '1 minute';
 count | sum
-------+-----
     0 |
(1 row)
```

Вставим еще одну строку в первой сессии и выполним commit
```sh
hw05=*# insert into orders(created_at, amount) values(current_timestamp, 100);
INSERT 0 1
hw05=*# commit;
```

Во второй сессии видны все вставленные строки
```sh
hw05=*# select count(*), sum(amount) from orders where created_at >= current_timestamp - interval '1 minute';
 count | sum
-------+-----
     3 | 300
(1 row)
```

Повторим еще вставку 2х строк
1 Сессия
```sh
hw05=# insert into orders(created_at, amount) values(current_timestamp, 100);
INSERT 0 1
hw05=*# insert into orders(created_at, amount) values(current_timestamp, 100);
INSERT 0 1
```

Вторая сессия
```sh
hw05=*# select count(*), sum(amount) from orders where created_at >= current_timestamp - interval '1 minute';
 count | sum
-------+-----
     3 | 300
(1 row)
```
После коммита в первой сессии, увидим все строки во второй

Вторая сессия
```sh
hw05=*# select count(*), sum(amount) from orders where created_at >= current_timestamp - interval '1 minute';
 count | sum
-------+-----
     5 | 500
(1 row)
```

### Тесты с изоляцией repeatable read

Сменим уровень изоляции транзакций на repeatable read. 
Но не через команду set transaction isolation level repeatable read 
(иначе после коммита уровень перейдет на дефолтный), а через alter database 
В 2х сессиях
```sh
hw05=*# alter database hw05 set default_transaction_isolation to 'repeatable read';
ALTER DATABASE
hw05=*# commit;
COMMIT
hw05=# show transaction_isolation;
 transaction_isolation
-----------------------
 read committed
(1 row)

hw05=*# \c postgres
You are now connected to database "postgres" as user "postgres".
postgres=# \c hw05
You are now connected to database "hw05" as user "postgres".
hw05=# show transaction_isolation;
 transaction_isolation
-----------------------
 repeatable read
```
Также будем использовать clock_timestamp() вместо current_timestamp (это время начала транзакции, а транзакции сейчас долгие в открытых терминалах), 
во избежание временных искажениий.

В первой сессии вставим 2 строки и выполним селект 
Данные видны в текущей транзакции 
```sh
hw05=# insert into orders(created_at, amount) values(clock_timestamp(), 100);
INSERT 0 1
hw05=*# insert into orders(created_at, amount) values(clock_timestamp(), 100);
INSERT 0 1
hw05=# select count(*), sum(amount) from orders where created_at >= clock_timestamp() - interval '1 minute';
 count | sum
-------+-----
     2 | 200
(1 row)
```

Во второй сессии данные не видны 
```sh
hw05=*# select count(*), sum(amount) from orders where created_at >= clock_timestamp() - interval '1 minute';
 count | sum
-------+-----
     0 |
(1 row)
```
Сделаем коммит в первой сесиии и select во второй.
Данные во второй сессии по прежнему не видны.


Сделаем коммит во второй сессии и снова сделаем select 
Теперь данные видны 
```sh
hw05=*# commit;
COMMIT
hw05=# select count(*), sum(amount) from orders where created_at >= clock_timestamp() - interval '1 minute';
 count | sum
-------+-----
     2 | 200
(1 row)
```

Добавим еще строку в первой сессии и коммит 
```sh
hw05=# insert into orders(created_at, amount) values(clock_timestamp(), 100);
INSERT 0 1
hw05=*# commit;
```

Во второй сессии изменения видны только после начала новой транзакции
```sh
hw05=*# select count(*), sum(amount) from orders where created_at >= clock_timestamp() - interval '1 minute';
 count | sum
-------+-----
     2 | 200
(1 row)

hw05=*# commit; 
COMMIT
hw05=# select count(*), sum(amount) from orders where created_at >= clock_timestamp() - interval '1 minute';
 count | sum
-------+-----
     3 | 300
(1 row)
```

Выводы:
На момент начала транзакции (первого оператора в транзакции) при изоляции 'repeatable read' 
фиксируется состояние данных и изменения, совершенные другими транзакциями, на него не влияют.

Если для отчета важно чтобы на момент старта операций его формирования данные были неизменны, 
т.е другие процессы ни на что не влияли, то самая походящая для этого изоляция 'repeatable read'.
