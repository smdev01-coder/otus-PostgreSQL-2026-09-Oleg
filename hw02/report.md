## Работаем на виртуальной машине с Ubuntu 26.04 (ubuntu01)

### Устанавливаем postgresql-18 и создаем тестовую таблицу

```sh
user@ubuntu01:~/otus$ sudo apt install -y postgresql-18

user@ubuntu01:~/otus$ sudo systemctl status postgresql
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Sun 2026-10-11 18:58:09 UTC; 3min 2s ago
 Invocation: 51d31d7b83464161ab89b691f0e6f779
   Main PID: 6802 (code=exited, status=0/SUCCESS)
   Mem peak: 1.7M
        CPU: 9ms

Oct 11 18:58:09 ubuntu01 systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
Oct 11 18:58:09 ubuntu01 systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.
user@ubuntu01:~/otus$ psql --version
psql (PostgreSQL) 18.6 (Ubuntu 18.6-0ubuntu0.26.04.1)
```

Проверка кластера
```sh 
user@ubuntu01:~/otus$ pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
18  main    5432 online postgres  /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

Подключаемся к postgres и создаем тестовую таблицу
```sh
user@ubuntu01:~/otus$ sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# create database hw02;
CREATE DATABASE
postgres=# \c hw02
You are now connected to database "hw02" as user "postgres".
hw02=# create table test (id int, description text);
CREATE TABLE
hw02=# insert into test values (1, 'description 1'),(2, 'description 2'),(3, 'description 3');
INSERT 0 3
hw02=#
```

Останавливаем кластер
```sh
user@ubuntu01:~/otus$ sudo pg_ctlcluster 18 main stop
user@ubuntu01:~/otus$ pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
18  main    5432 down   postgres  /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

### Создаем дополнительный диск на 10 GB и подключаем к виртуальной машине

Проверяем что диск в системе и не примонтирован (/dev/sdb)
```sh
user@ubuntu01:~$ ls -l /dev/sd*
brw-rw---- 1 root disk 8,  0 Oct 11 19:44 /dev/sda
brw-rw---- 1 root disk 8,  1 Oct 11 19:44 /dev/sda1
brw-rw---- 1 root disk 8,  2 Oct 11 19:44 /dev/sda2
brw-rw---- 1 root disk 8, 16 Oct 11 19:44 /dev/sdb
user@ubuntu01:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0     4K  1 loop /snap/bare/5
...
sda      8:0    0  42.2G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0  42.2G  0 part /
sdb      8:16   0    10G  0 disk
sr0     11:0    1  1024M  1 rom
```

Размечаем диск
```sh
user@ubuntu01:~$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.41.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0xac24e31a.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

user@ubuntu01:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sda      8:0    0  42.2G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0  42.2G  0 part /
sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part
sr0     11:0    1  1024M  1 rom
```

Создаем файловую систему
```sh
user@ubuntu01:~$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: fb505c45-cd3a-4dd8-b5fd-662c8c9f3514
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

Монтирование
```sh
user@ubuntu01:~$ sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
user@ubuntu01:~$ df -hT | grep data
/dev/sdb1      ext4   9.8G  2.1M  9.3G   1% /mnt/data
```

Автомонтирование при старте. \
Прописываем в файле /etc/fstab новую строку \
'/dev/sdb1 /mnt/data ext4 defaults 0 0'

```sh
user@ubuntu01:~$ sudo nano /etc/fstab
  GNU nano 8.7.1                                                                                    /etc/fstab *
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/ba6f490f-ce96-40d1-82bc-ed360f864578 / ext4 defaults 0 1
/dev/sdb1 /mnt/data ext4 defaults 0 0
```

Перезагружаем виртуальную машину и проверяем что диск примонтирован
```sh
user@ubuntu01:~$ df -hT | grep data
/dev/sdb1      ext4   9.8G  2.1M  9.3G   1% /mnt/data
user@ubuntu01:~$ cd /mnt/data
user@ubuntu01:/mnt/data$ ls -l
total 16
drwx------ 2 root root 16384 Oct 11 19:58 lost+found
```

Устанавливаем владельцем каталога пользователя postgres
```sh
user@ubuntu01:/mnt/data$ sudo chown -R postgres:postgres /mnt/data
[sudo: authenticate] Password:
user@ubuntu01:/mnt/data$ ls -l
total 16
drwx------ 2 postgres postgres 16384 Oct 11 19:58 lost+found
```

### Перенос каталог данных PostgreSQL на /mnt/data

Останавливаем кластер и проверяем
```sh
user@ubuntu01:/mnt/data$ sudo pg_ctlcluster 18 main stop
user@ubuntu01:/mnt/data$ pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
18  main    5432 down   postgres  /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

На всякий случай делаем бэкап в виде архива
```sh
user@ubuntu01:/mnt/data$ sudo tar -czpf /home/user/pg18.tar.gz /var/lib/postgresql/18/main
```

Переносим данные
```sh
user@ubuntu01:/mnt/data$ sudo -u postgres mv /var/lib/postgresql/18/main/ /mnt/data/
```

Проверяем что данные перенесены
```sh
user@ubuntu01:/mnt/data$ sudo -u postgres ls -l /var/lib/postgresql/18/main
ls: cannot access '/var/lib/postgresql/18/main': No such file or directory

user@ubuntu01:/mnt/data$ sudo -u postgres ls -l /mnt/data/main
total 80
-rw------- 1 postgres postgres    3 Oct 11 18:58 PG_VERSION
drwx------ 6 postgres postgres 4096 Oct 11 19:26 base
drwx------ 2 postgres postgres 4096 Oct 11 20:06 global
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_commit_ts
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_dynshmem
drwx------ 4 postgres postgres 4096 Oct 11 20:25 pg_logical
drwx------ 4 postgres postgres 4096 Oct 11 18:58 pg_multixact
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_notify
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_replslot
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_serial
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_snapshots
drwx------ 2 postgres postgres 4096 Oct 11 20:25 pg_stat
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_stat_tmp
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_subtrans
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_tblspc
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_twophase
drwx------ 4 postgres postgres 4096 Oct 11 19:29 pg_wal
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_xact
-rw------- 1 postgres postgres   88 Oct 11 18:58 postgresql.auto.conf
-rw------- 1 postgres postgres  130 Oct 11 20:05 postmaster.opts
```

Пытаемся запустить кластер, но он не может стартовать
```sh
user@ubuntu01:/mnt/data$ pg_ctlcluster 18 main start
Error: /var/lib/postgresql/18/main is not accessible or does not exist
```
Причина: Данных по старому пути уже нет, а новый путь мы еще не задали в конфигурации.

Ищем все что связано с '18/main'
```sh
user@ubuntu01:/mnt/data$ sudo -u postgres grep -r '18/main' /etc/postgresql/18/main/*
/etc/postgresql/18/main/postgresql.conf:data_directory = '/var/lib/postgresql/18/main'          # use data in another directory
/etc/postgresql/18/main/postgresql.conf:hba_file = '/etc/postgresql/18/main/pg_hba.conf'        # host-based authentication file
/etc/postgresql/18/main/postgresql.conf:ident_file = '/etc/postgresql/18/main/pg_ident.conf'    # ident configuration file
/etc/postgresql/18/main/postgresql.conf:cluster_name = '18/main'                        # added to process titles if nonempty
```

В файле /etc/postgresql/18/main/postgresql.conf исправляем data_directory c '/var/lib/postgresql/18/main' на '/mnt/data/main'

```sh
sudo -u postgres nano /etc/postgresql/18/main/postgresql.conf

  GNU nano 8.7.1                                                        /etc/postgresql/18/main/postgresql.conf
# with the "SET" SQL command.
#
# Memory units:  B  = bytes            Time units:  us  = microseconds
#                kB = kilobytes                     ms  = milliseconds
#                MB = megabytes                     s   = seconds
#                GB = gigabytes                     min = minutes
#                TB = terabytes                     h   = hours
#                                                   d   = days


#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------

# The default values of these variables are driven from the -D command-line
# option or PGDATA environment variable, represented here as ConfigDir.

data_directory = '/mnt/data/main'               # use data in another directory
                                        # (change requires restart)
hba_file = '/etc/postgresql/18/main/pg_hba.conf'        # host-based authentication file
                                        # (change requires restart)
ident_file = '/etc/postgresql/18/main/pg_ident.conf'    # ident configuration file
                                        # (change requires restart)

# If external_pid_file is not explicitly set, no extra PID file is written.
```

Стартуем кластер 
```sh
user@ubuntu01:/mnt/data$ sudo pg_ctlcluster 18 main start
user@ubuntu01:/mnt/data$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
18  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-18-main.log
```
Сейчас все успешно.

Проверяем что тестовая таблица с данными на месте
```sh
user@ubuntu01:~$ sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# \c hw02
You are now connected to database "hw02" as user "postgres".
hw02=# select * from test;
 id |  description
----+---------------
  1 | description 1
  2 | description 2
  3 | description 3
(3 rows)
```

##  Вторая виртуальная машина с Ubuntu 26.04 (ubuntu02)

### Запуск и настройка
Останавливаем первую виртуальную машину и отсоединяем внешний диск.
Запускаем вторую виртуальную машину с внешним диском, полученным на предыдущих шагах.
На второй машине тоже установлен postgres18

Проверяем что диск в системе
```sh
user1@ubuntu02:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sda      8:0    0    25G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    25G  0 part /
sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part
sr0     11:0    1  1024M  1 rom
```

Останавливаем кластер 
```sh
user1@ubuntu02:~$ sudo pg_ctlcluster 18 main stop
user1@ubuntu02:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

Монтирование и проверка
```sh
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data

user1@ubuntu02:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sda      8:0    0    25G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    25G  0 part /
sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part /mnt/data
sr0     11:0    1  1024M  1 rom

user1@ubuntu02:~$ sudo -u postgres ls -l /mnt/data/main
total 84
-rw------- 1 postgres postgres    3 Oct 11 18:58 PG_VERSION
drwx------ 6 postgres postgres 4096 Oct 11 19:26 base
drwx------ 2 postgres postgres 4096 Oct 11 20:56 global
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_commit_ts
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_dynshmem
drwx------ 4 postgres postgres 4096 Oct 11 21:05 pg_logical
drwx------ 4 postgres postgres 4096 Oct 11 18:58 pg_multixact
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_notify
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_replslot
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_serial
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_snapshots
drwx------ 2 postgres postgres 4096 Oct 11 20:55 pg_stat
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_stat_tmp
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_subtrans
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_tblspc
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_twophase
drwx------ 4 postgres postgres 4096 Oct 11 19:29 pg_wal
drwx------ 2 postgres postgres 4096 Oct 11 18:58 pg_xact
-rw------- 1 postgres postgres   88 Oct 11 18:58 postgresql.auto.conf
-rw------- 1 postgres postgres  117 Oct 11 20:55 postmaster.opts
-rw------- 1 postgres postgres   95 Oct 11 20:55 postmaster.pid
```

Исправляем data_directory c '/var/lib/postgresql/18/main' на '/mnt/data/main' в /etc/postgresql/18/main/postgresql.conf\
Стартуем кластер и проверяем что все на месте
```sh
user1@ubuntu02:~$ sudo pg_ctlcluster 18 main start
user1@ubuntu02:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
18  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-18-main.log

user1@ubuntu02:~$ sudo -u postgres psql
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# \c hw02
You are now connected to database "hw02" as user "postgres".
hw02=# select * from test;
 id |  description
----+---------------
  1 | description 1
  2 | description 2
  3 | description 3
(3 rows)

hw02=#
```
