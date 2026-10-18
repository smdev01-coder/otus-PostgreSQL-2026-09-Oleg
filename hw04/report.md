## Работаем на виртуальной машине Ubuntu 26.04 (6 ядер ЦПУ 16 ГБ оперативной памяти) c postgresql 18


### Cоздаем тестовую базу, и инициализируем

Проверка кластера
```sh 
user@ubuntu01:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
18  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-18-main.log
```


Создаем базу данных benchtest
```sh
user@ubuntu01:~$ sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# create database benchtest;
CREATE DATABASE
```

Инициализируем базу для pg_bench, scale_factor = 10.
```sh
user@ubuntu01:~$ sudo -u postgres pgbench -i -s 10 -d benchtest
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 2.42 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 1.67 s, vacuum 0.22 s, primary keys 0.52 s).
user@ubuntu01:~$
```

Проверяем что таблицы присутствуют и проинициализированы
```sh
user@ubuntu01:~$ sudo -u postgres psql -d benchtest
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

benchtest=# \dt+
                                           List of tables
 Schema |       Name       | Type  |  Owner   | Persistence | Access method |  Size   | Description
--------+------------------+-------+----------+-------------+---------------+---------+-------------
 public | pgbench_accounts | table | postgres | permanent   | heap          | 128 MB  |
 public | pgbench_branches | table | postgres | permanent   | heap          | 40 kB   |
 public | pgbench_history  | table | postgres | permanent   | heap          | 0 bytes |
 public | pgbench_tellers  | table | postgres | permanent   | heap          | 40 kB   |
(4 rows)

benchtest=# select count(*) from pgbench_accounts;
  count
---------
 1000000
(1 row)

benchtest=# select count(*) from pgbench_branches;
 count
-------
    10
(1 row)

benchtest=# select count(*) from pgbench_history;
 count
-------
     0
(1 row)

benchtest=# select count(*) from pgbench_tellers;
 count
-------
   100
(1 row)
```

### Изменение параметров, и прогон pgbench

Запускаем pgbench с настройками 60 секунд, 10 клиентов, 4 потока
```sh
user@ubuntu01:~$ sudo -u postgres pgbench -T 60 -c 10 -j 4 -P 10 benchtest
pgbench (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
starting vacuum...end.
progress: 10.0 s, 969.4 tps, lat 10.264 ms stddev 7.859, 0 failed
progress: 20.0 s, 1029.5 tps, lat 9.699 ms stddev 6.347, 0 failed
progress: 30.0 s, 959.3 tps, lat 10.409 ms stddev 9.436, 0 failed
progress: 40.0 s, 948.3 tps, lat 10.527 ms stddev 11.013, 0 failed
progress: 50.0 s, 961.4 tps, lat 10.385 ms stddev 10.517, 0 failed
progress: 60.0 s, 965.5 tps, lat 10.336 ms stddev 6.971, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 58343
number of failed transactions: 0 (0.000%)
latency average = 10.263 ms
latency stddev = 8.836 ms
initial connection time = 12.821 ms
tps = 972.334198 (without initial connection time)
```

Зафиксируем текущие настройки, которые будем изменять
```sh
benchtest=# select current_setting('max_worker_processes') max_worker_processes,
        current_setting('max_parallel_workers_per_gather') max_parallel_workers_per_gather,
        current_setting('max_parallel_workers') max_parallel_workers,
        current_setting('max_parallel_maintenance_workers') max_parallel_maintenance_workers,
        current_setting('shared_buffers') shared_buffers,
        current_setting('effective_cache_size') effective_cache_size,
        current_setting('work_mem') work_mem,
        current_setting('maintenance_work_mem') maintenance_work_mem;
 max_worker_processes | max_parallel_workers_per_gather | max_parallel_workers | max_parallel_maintenance_workers | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem
----------------------+---------------------------------+----------------------+----------------------------------+----------------+----------------------+----------+----------------------
 8                    | 2                               | 8                    | 2                                | 128MB          | 4GB                  | 4MB      | 64MB

```

Меняем параметры, перезагружаем конфигурацию и видим что сервер нуждается в перезагрузке
```sh
benchtest=# alter system set max_worker_processes=6;
alter system set max_parallel_workers=6;
alter system set max_parallel_workers_per_gather=3;
alter system set max_parallel_maintenance_workers=3;
alter system set shared_buffers='4GB';
alter system set effective_cache_size='10GB';
alter system set work_mem='14MB';
alter system set maintenance_work_mem='1GB';
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
benchtest=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

benchtest=# select name from pg_settings where pending_restart=true;
         name
----------------------
 max_worker_processes
 shared_buffers
(2 rows)

```

Перезагружаем сервер
```sh
user@ubuntu01:~$ sudo pg_ctlcluster 18  main stop
[sudo: authenticate] Password:
user@ubuntu01:~$ sudo pg_ctlcluster 18  main start
user@ubuntu01:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
18  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-18-main.log

```

Видим что настройки применены успешно
```sh
user@ubuntu01:~$ sudo -u postgres psql -d benchtest
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

benchtest=# select name from pg_settings where pending_restart=true;
 name
------
(0 rows)

benchtest=# select current_setting('max_worker_processes') max_worker_processes,
        current_setting('max_parallel_workers_per_gather') max_parallel_workers_per_gather,
        current_setting('max_parallel_workers') max_parallel_workers,
        current_setting('max_parallel_maintenance_workers') max_parallel_maintenance_workers,
        current_setting('shared_buffers') shared_buffers,
        current_setting('effective_cache_size') effective_cache_size,
        current_setting('work_mem') work_mem,
        current_setting('maintenance_work_mem') maintenance_work_mem;
 max_worker_processes | max_parallel_workers_per_gather | max_parallel_workers | max_parallel_maintenance_workers | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem
----------------------+---------------------------------+----------------------+----------------------------------+----------------+----------------------+----------+----------------------
 6                    | 3                               | 6                    | 3                                | 4GB            | 10GB                 | 14MB     | 1GB
(1 row)
```

Запускаем pgbench
```sh
user@ubuntu01:~$ sudo -u postgres pgbench -T 60 -c 10 -j 4 -P 10 benchtest
pgbench (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
starting vacuum...end.
progress: 10.0 s, 971.4 tps, lat 10.240 ms stddev 7.063, 0 failed
progress: 20.0 s, 997.6 tps, lat 10.014 ms stddev 6.544, 0 failed
progress: 30.0 s, 1017.6 tps, lat 9.804 ms stddev 6.126, 0 failed
progress: 40.0 s, 1022.7 tps, lat 9.761 ms stddev 6.018, 0 failed
progress: 50.0 s, 1093.3 tps, lat 9.129 ms stddev 5.170, 0 failed
progress: 60.0 s, 1096.5 tps, lat 9.093 ms stddev 5.134, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 62000
number of failed transactions: 0 (0.000%)
latency average = 9.656 ms
latency stddev = 6.037 ms
initial connection time = 12.688 ms
tps = 1033.082346 (without initial connection time)
```

Проверим параметры дисковой подсистемы (работаем c SSD)


```sh
user@ubuntu01:~$ sudo -u postgres psql -d benchtest
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

benchtest=# select current_setting('fsync') fsync,
        current_setting('checkpoint_completion_target') checkpoint_completion_target,
        current_setting('effective_io_concurrency') effective_io_concurrency,
        current_setting('synchronous_commit') synchronous_commit,
        current_setting('random_page_cost') random_page_cost;
 fsync | checkpoint_completion_target | effective_io_concurrency | synchronous_commit | random_page_cost
-------+------------------------------+--------------------------+--------------------+------------------
 on    | 0.9                          | 16                       | on                 | 4
```

Изменим 
effective_io_concurrency
synchronous_commit
random_page_cost

```sh
benchtest=# alter system set effective_io_concurrency=200;
alter system set synchronous_commit=off;
alter system set random_page_cost=1.1;
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
benchtest=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

benchtest=# select name from pg_settings where pending_restart=true;
 name
------
(0 rows)

```
Запускаем pgbench
```sh
user@ubuntu01:~$ sudo -u postgres pgbench -T 60 -c 10 -j 4 -P 10 benchtest
pgbench (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
starting vacuum...end.
progress: 10.0 s, 3664.8 tps, lat 2.707 ms stddev 1.262, 0 failed
progress: 20.0 s, 3660.8 tps, lat 2.713 ms stddev 1.165, 0 failed
progress: 30.0 s, 3659.6 tps, lat 2.715 ms stddev 1.181, 0 failed
progress: 40.0 s, 3670.0 tps, lat 2.706 ms stddev 1.227, 0 failed
progress: 50.0 s, 3676.9 tps, lat 2.703 ms stddev 1.178, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 219614
number of failed transactions: 0 (0.000%)
latency average = 2.714 ms
latency stddev = 1.207 ms
initial connection time = 15.268 ms
tps = 3660.436915 (without initial connection time)
```

Отключим fsync для теста, на продакшн системах делать этого не рекомендуется. 
Это создаёт серьёзный риск: при аварийном завершении работы часть данных может быть потеряна или повреждена.

```sh
benchtest=# alter system set fsync=off;
ALTER SYSTEM
benchtest=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

benchtest=# select name from pg_settings where pending_restart=true;
 name
------
(0 rows)

benchtest=# select current_setting('fsync') fsync,
        current_setting('checkpoint_completion_target') checkpoint_completion_target,
        current_setting('effective_io_concurrency') effective_io_concurrency,
        current_setting('synchronous_commit') synchronous_commit,
        current_setting('random_page_cost') random_page_cost;
 fsync | checkpoint_completion_target | effective_io_concurrency | synchronous_commit | random_page_cost
-------+------------------------------+--------------------------+--------------------+------------------
 off   | 0.9                          | 200                      | off                | 1.1
(1 row)
```

Запускаем pgbench
```sh
user@ubuntu01:~$ sudo -u postgres pgbench -T 60 -c 10 -j 4 -P 10 benchtest
pgbench (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
starting vacuum...end.
progress: 10.0 s, 5308.9 tps, lat 1.870 ms stddev 0.959, 0 failed
progress: 20.0 s, 5580.8 tps, lat 1.783 ms stddev 0.963, 0 failed
progress: 30.0 s, 5687.4 tps, lat 1.749 ms stddev 0.970, 0 failed
progress: 40.0 s, 5472.9 tps, lat 1.818 ms stddev 1.151, 0 failed
progress: 50.0 s, 5521.6 tps, lat 1.802 ms stddev 1.131, 0 failed
progress: 60.0 s, 5643.3 tps, lat 1.763 ms stddev 0.952, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 332152
number of failed transactions: 0 (0.000%)
latency average = 1.797 ms
latency stddev = 1.026 ms
initial connection time = 15.208 ms
tps = 5534.968341 (without initial connection time)

```

### Таблица и выводы после прогонов pgbench (настройки scale_factor = 10, 60 секунд, 10 клиентов, 4 потока)

Таблица измененных параметров для системы с SSD, 6 ЦПУ и 16 ГБ ОЗУ

| Параметр                         | До изменений | После изменений | Описание                                    |
| -------------------------------- | ------------ | --------------- | --------------------------------------------|
| max_worker_processes             | 8            | 6               | <= числу ядер ЦПУ                           |
| max_parallel_workers             | 8            | 6               | <= числу ядер ЦПУ                           |
| max_parallel_workers_per_gather  | 2            | 3               | max_parallel_workers / 2                    |
| max_parallel_maintenance_workers | 2            | 3               | max_parallel_workers / 2                    |
| shared_buffers                   | 128MB        | 4GB             | 25% – 40% ОЗУ                               |
| effective_cache_size             | 4GB          | 10GB            | ~ 2/3 ОЗУ                                   |
| work_mem                         | 4MB          | 14MB            | Рекомендация PGTune                         |
| maintenance_work_mem             | 64MB         | 1GB             | Рекомендация PGTune                         |
| effective_io_concurrency         | 16           | 200             | Рекомендация PGTune для SSD                 |
| synchronous_commit               | on           | off             | Это значительно повышает производительность |
| random_page_cost                 | 4            | 1.1             | Рекомендация PGTune для SSD                 |
| fsync                            | on           | off             | Это значительно повышает производительность |


tps - сколько транзакций база данных успевает обработать за одну секунду.
1) tps (базовый прогон) ~972
2) tps (прогон после смены параметров, касающихся ОЗУ и ЦПУ) ~1033
3) tps (прогон после смены параметров, касающихся дисковых операций, fsync=on) ~3660
4) tps (прогон после смены параметров, касающихся дисковых операций, fsync=off) ~5535

Выводы:
Обычная подстройка параметров, связанных с памятью и количеством ядер, дает прирост производительности.
Но максимальный эффект дают настройки связанные с подсистемой ввода/вывода: synchronous_commit, fsync.

synchronous_commit=off — Cинхронная обработка полностью отключается.
Сервер сразу возвращает подтверждение клиенту, не дожидаясь записи WAL на диск.
Отключение synchronous_commit не приводит к несогласованному состоянию базы данных — при сбое система восстановится корректно, просто часть транзакций может отсутствовать.

fsync=off - Отключение этого параметра может значительно повысить производительность, сервер перестаёт ждать записи данных на диск.
Однако это создаёт серьёзный риск: при аварийном завершении работы часть данных может быть потеряна или повреждена.
