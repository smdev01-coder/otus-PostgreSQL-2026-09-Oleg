## Работаем на виртуальной машине Ubuntu 26.04 (2 ядра ЦПУ 4 ГБ оперативной памяти) c postgresql 18


### Cоздаем тестовую базу и инициализируем

Создаем базу данных hw06 
```sh
user@ubuntu01:~$ sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# create database hw06;
CREATE DATABASE
```

Инициализируем базу для pgbench. 
```sh
root@ubuntu01:~# sudo -u postgres pgbench -i hw06
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.38 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.18 s, vacuum 0.13 s, primary keys 0.06 s).
user@ubuntu01:~$
```

Прогоняем нагрузку 
```sh
root@ubuntu01:~# sudo -u postgres pgbench -c8 -P 6 -T 60 hw06
pgbench (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
starting vacuum...end.
progress: 6.0 s, 512.7 tps, lat 15.497 ms stddev 14.711, 0 failed
progress: 12.0 s, 530.0 tps, lat 15.086 ms stddev 13.790, 0 failed
progress: 18.0 s, 535.8 tps, lat 14.913 ms stddev 13.882, 0 failed
progress: 24.0 s, 540.2 tps, lat 14.789 ms stddev 13.852, 0 failed
progress: 30.0 s, 537.0 tps, lat 14.888 ms stddev 13.395, 0 failed
progress: 36.0 s, 529.3 tps, lat 15.101 ms stddev 14.163, 0 failed
progress: 42.0 s, 534.0 tps, lat 14.964 ms stddev 14.566, 0 failed
progress: 48.0 s, 519.8 tps, lat 15.352 ms stddev 15.539, 0 failed
progress: 54.0 s, 531.8 tps, lat 15.018 ms stddev 14.206, 0 failed
progress: 60.0 s, 529.8 tps, lat 15.131 ms stddev 14.883, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31811
number of failed transactions: 0 (0.000%)
latency average = 15.075 ms
latency stddev = 14.311 ms
initial connection time = 24.362 ms
tps = 529.937204 (without initial connection time)
```

Фиксируем состояние
```sh
root@ubuntu01:~# sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# \c hw06
You are now connected to database "hw06" as user "postgres".
hw06=# select relname, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables;
     relname      | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+------------+-------------------------------+-------------------------------
 pgbench_accounts |       3918 | 2026-10-25 22:24:50.535005+00 | 2026-10-25 22:24:11.554086+00
 pgbench_history  |          0 | 2026-10-25 22:30:51.513711+00 | 2026-10-25 22:24:11.626841+00
 pgbench_tellers  |          0 | 2026-10-25 22:30:51.510027+00 | 2026-10-25 22:29:17.742495+00
 pgbench_branches |          0 | 2026-10-25 22:30:51.506415+00 | 2026-10-25 22:29:17.74202+00
(4 rows)
```

Создаем таблицу test и наполняем ее 1000000 строк.  
Размер таблицы ~42 MB (мёртвые кортежи отсутствуют)
```sh
hw06=# create table test(id integer, name text);
CREATE TABLE
hw06=# insert into test select id, 'field' || id::text from (select generate_series(1, 1000000) as id);
INSERT 0 1000000
hw06=# select relname, pg_total_relation_size(relname::regclass) as tablesize, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables;
     relname      | tablesize | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+-----------+------------+-------------------------------+-------------------------------
 pgbench_accounts |  16097280 |       3918 | 2026-10-25 22:24:50.535005+00 | 2026-10-25 22:24:11.554086+00
 pgbench_history  |   1728512 |          0 | 2026-10-25 22:30:51.513711+00 | 2026-10-25 22:24:11.626841+00
 pgbench_tellers  |     90112 |          0 | 2026-10-25 22:30:51.510027+00 | 2026-10-25 22:29:17.742495+00
 test             |  44335104 |          0 | 2026-10-25 23:39:52.996758+00 |
 pgbench_branches |    106496 |          0 | 2026-10-25 22:30:51.506415+00 | 2026-10-25 22:29:17.74202+00
(5 rows)
```

Выполняем 5 полных обновлений строк (добавление символа к текстовому полю)  
Размер таблицы ~288 MB, мёртвых кортежей 4968977, последний автовакуум 2026-10-25 23:39:52.996758+00 
```sh
hw06=# do $$
    declare
		num_iter integer = 5;
		start integer;
		i integer;
		checkdate timestamptz;
		group_cnt integer = 10000;
    begin
		i = 1;
		while i <= num_iter loop
			checkdate = current_timestamp;
			raise notice 'Итерация: %; начало %', i, checkdate;
			start = 1;

			while start < 1000000 loop
				update test set name = name || 'a' where id >= start and id < start + group_cnt;
				start = start + group_cnt;
				commit;
			end loop;
			raise notice 'Итерация: %; конец %; длительность %', i, current_timestamp, current_timestamp - checkdate;
			i = i + 1;
		end loop;
    end;
$$;
NOTICE:  Итерация: 1; начало 2026-10-25 23:45:35.157428+00
NOTICE:  Итерация: 1; конец 2026-10-25 23:45:40.943044+00; длительность 00:00:05.785616
NOTICE:  Итерация: 2; начало 2026-10-25 23:45:40.943044+00
NOTICE:  Итерация: 2; конец 2026-10-25 23:45:47.405472+00; длительность 00:00:06.462428
NOTICE:  Итерация: 3; начало 2026-10-25 23:45:47.405472+00
NOTICE:  Итерация: 3; конец 2026-10-25 23:45:56.537376+00; длительность 00:00:09.131904
NOTICE:  Итерация: 4; начало 2026-10-25 23:45:56.537376+00
NOTICE:  Итерация: 4; конец 2026-10-25 23:46:09.672019+00; длительность 00:00:13.134643
NOTICE:  Итерация: 5; начало 2026-10-25 23:46:09.672019+00
NOTICE:  Итерация: 5; конец 2026-10-25 23:46:24.612698+00; длительность 00:00:14.940679
DO
hw06=# select relname, pg_total_relation_size(relname::regclass) as tablesize, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables;
     relname      | tablesize | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+-----------+------------+-------------------------------+-------------------------------
 pgbench_accounts |  16097280 |       3918 | 2026-10-25 22:24:50.535005+00 | 2026-10-25 22:24:11.554086+00
 pgbench_history  |   1728512 |          0 | 2026-10-25 22:30:51.513711+00 | 2026-10-25 22:24:11.626841+00
 pgbench_tellers  |     90112 |          0 | 2026-10-25 22:30:51.510027+00 | 2026-10-25 22:29:17.742495+00
 test             | 302047232 |    4968977 | 2026-10-25 23:39:52.996758+00 |
 pgbench_branches |    106496 |          0 | 2026-10-25 22:30:51.506415+00 | 2026-10-25 22:29:17.74202+00
(5 rows)
```

Ждем завершения автовакуума  
Размер таблицы ~288 MB (даже немного увеличился с прошлого раза), мёртвых кортежей нет, последний автовакуум 2026-10-25 23:47:04.604024+00 
```sh
hw06=# select relname, pg_total_relation_size(relname::regclass) as tablesize, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables;
     relname      | tablesize | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+-----------+------------+-------------------------------+-------------------------------
 pgbench_accounts |  16097280 |       3918 | 2026-10-25 22:24:50.535005+00 | 2026-10-25 22:24:11.554086+00
 pgbench_history  |   1728512 |          0 | 2026-10-25 22:30:51.513711+00 | 2026-10-25 22:24:11.626841+00
 pgbench_tellers  |     90112 |          0 | 2026-10-25 22:30:51.510027+00 | 2026-10-25 22:29:17.742495+00
 test             | 302055424 |          0 | 2026-10-25 23:47:04.604024+00 |
 pgbench_branches |    106496 |          0 | 2026-10-25 22:30:51.506415+00 | 2026-10-25 22:29:17.74202+00
(5 rows)

```

Отключаем автовакуум для нашей тестовой таблицы  
И выполняем 10 полных обновлений строк (добавление символа к текстовому полю), через анонимный блок  
Размер таблицы ~584 MB, мёртвых кортежей 9728952
```sh
hw06=# alter table test set (autovacuum_enabled = false);
ALTER TABLE
hw06=# select reloptions from pg_class where relname = 'test';
         reloptions
----------------------------
 {autovacuum_enabled=false}
(1 row)

hw06=# do $$
    declare
        num_iter integer = 10;
        start integer;
        i integer;
        checkdate timestamptz;
        group_cnt integer = 10000;
    begin
        i = 1;
        while i <= num_iter loop
            checkdate = current_timestamp;
            raise notice 'Итерация: %; начало %', i, checkdate;
            start = 1;

            while start < 1000000 loop
                update test set name = name || 'a' where id >= start and id < start + group_cnt;
                start = start + group_cnt;
                commit;
            end loop;
            raise notice 'Итерация: %; конец %; длительность %', i, current_timestamp, current_timestamp - checkdate;
            i = i + 1;
        end loop;
    end;
$$;
NOTICE:  Итерация: 1; начало 2026-10-25 23:56:59.489635+00
NOTICE:  Итерация: 1; конец 2026-10-25 23:57:13.911026+00; длительность 00:00:14.421391
NOTICE:  Итерация: 2; начало 2026-10-25 23:57:13.911026+00
NOTICE:  Итерация: 2; конец 2026-10-25 23:57:29.929239+00; длительность 00:00:16.018213
NOTICE:  Итерация: 3; начало 2026-10-25 23:57:29.929239+00
NOTICE:  Итерация: 3; конец 2026-10-25 23:57:46.260443+00; длительность 00:00:16.331204
NOTICE:  Итерация: 4; начало 2026-10-25 23:57:46.260443+00
NOTICE:  Итерация: 4; конец 2026-10-25 23:58:03.976065+00; длительность 00:00:17.715622
NOTICE:  Итерация: 5; начало 2026-10-25 23:58:03.976065+00
NOTICE:  Итерация: 5; конец 2026-10-25 23:58:22.389594+00; длительность 00:00:18.413529
NOTICE:  Итерация: 6; начало 2026-10-25 23:58:22.389594+00
NOTICE:  Итерация: 6; конец 2026-10-25 23:58:41.681495+00; длительность 00:00:19.291901
NOTICE:  Итерация: 7; начало 2026-10-25 23:58:41.681495+00
NOTICE:  Итерация: 7; конец 2026-10-25 23:59:02.804751+00; длительность 00:00:21.123256
NOTICE:  Итерация: 8; начало 2026-10-25 23:59:02.804751+00
NOTICE:  Итерация: 8; конец 2026-10-25 23:59:26.88388+00; длительность 00:00:24.079129
NOTICE:  Итерация: 9; начало 2026-10-25 23:59:26.88388+00
NOTICE:  Итерация: 9; конец 2026-10-25 23:59:54.332308+00; длительность 00:00:27.448428
NOTICE:  Итерация: 10; начало 2026-10-25 23:59:54.332308+00
NOTICE:  Итерация: 10; конец 2026-10-26 00:00:24.227154+00; длительность 00:00:29.894846
DO
hw06=# select relname, pg_total_relation_size(relname::regclass) as tablesize, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables;
     relname      | tablesize | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+-----------+------------+-------------------------------+-------------------------------
 pgbench_accounts |  16097280 |       3918 | 2026-10-25 22:24:50.535005+00 | 2026-10-25 22:24:11.554086+00
 pgbench_history  |   1728512 |          0 | 2026-10-25 22:30:51.513711+00 | 2026-10-25 22:24:11.626841+00
 pgbench_tellers  |     90112 |          0 | 2026-10-25 22:30:51.510027+00 | 2026-10-25 22:29:17.742495+00
 test             | 611893248 |    9728952 | 2026-10-25 23:47:04.604024+00 |
 pgbench_branches |    106496 |          0 | 2026-10-25 22:30:51.506415+00 | 2026-10-25 22:29:17.74202+00
(5 rows)

```
Как видим растет и размер таблицы и число мертвых кортежей.  
Все как и ожидается: автовакуум отключен, старые версии кортежей не помечаются свободными для использования 
и поэтому для новых версий кортежей выделяется новое место на диске


Включаем автовакуум для таблицы и ждем
```sh
hw06=# alter table test set (autovacuum_enabled = true);
ALTER TABLE
hw06=# select reloptions from pg_class where relname = 'test';
        reloptions
---------------------------
 {autovacuum_enabled=true}
(1 row)

hw06=# select relname, pg_total_relation_size(relname::regclass) as tablesize, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables;
     relname      | tablesize | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+-----------+------------+-------------------------------+-------------------------------
 pgbench_accounts |  16097280 |       3918 | 2026-10-25 22:24:50.535005+00 | 2026-10-25 22:24:11.554086+00
 pgbench_history  |   1728512 |          0 | 2026-10-25 22:30:51.513711+00 | 2026-10-25 22:24:11.626841+00
 pgbench_tellers  |     90112 |          0 | 2026-10-25 22:30:51.510027+00 | 2026-10-25 22:29:17.742495+00
 test             | 611893248 |    9728952 | 2026-10-25 23:47:04.604024+00 |
 pgbench_branches |    106496 |          0 | 2026-10-25 22:30:51.506415+00 | 2026-10-25 22:29:17.74202+00
(5 rows)
...
...
hw06=# select relname, pg_total_relation_size(relname::regclass) as tablesize, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables;
     relname      | tablesize | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+-----------+------------+-------------------------------+-------------------------------
 pgbench_accounts |  16097280 |       3918 | 2026-10-25 22:24:50.535005+00 | 2026-10-25 22:24:11.554086+00
 pgbench_history  |   1728512 |          0 | 2026-10-25 22:30:51.513711+00 | 2026-10-25 22:24:11.626841+00
 pgbench_tellers  |     90112 |          0 | 2026-10-25 22:30:51.510027+00 | 2026-10-25 22:29:17.742495+00
 test             | 611901440 |          0 | 2026-10-26 00:08:32.404817+00 |
 pgbench_branches |    106496 |          0 | 2026-10-25 22:30:51.506415+00 | 2026-10-25 22:29:17.74202+00
(5 rows)

```
Место на диске не освободилось, это и предсказуемо, а вот мертвые кортежи почистились.   
Для возврата места в систему можно использовать vacuum full (или pg_repack), если это реально требуется.  
```sh
hw06=# vacuum full test;
VACUUM
hw06=# select relname, pg_total_relation_size(relname::regclass) as tablesize, n_dead_tup from pg_stat_user_tables;
     relname      | tablesize | n_dead_tup
------------------+-----------+------------
 pgbench_accounts |  16097280 |          0
 pgbench_history  |   1728512 |          0
 pgbench_tellers  |     90112 |          0
 test             |  60243968 |          0
 pgbench_branches |    106496 |          0
(5 rows)
```