## Работаем на виртуальной машине с Ubuntu 26.04

### 1. Устанавливаем Docker Engine

Инструкция https://docs.docker.com/engine/install/ubuntu/

Добавляем локального пользователя в группу docker
```sh 
sudo usermod -aG docker user
newgrp docker
```

### 2. Загрузка и запуск postgres18

Создаем на host каталог и выдаем права для пользователя postgres 
```sh
sudo mkdir /var/lib/postgresql
sudo chown 999:999 /var/lib/postgresql
sudo chmod 700 /var/lib/postgresql
```

Загрузка образа
```sh
docker pull postgres:18
```
```sh
docker images
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
postgres:18          74935e722416        650MB          169MB
```

Cоздаем файл docker-compose.yml
```
services:
  postgres18:
    container_name: postgres18
    image: postgres:18
    restart: always
    ports:
      - 5432:5432
    volumes:
      - /var/lib/postgresql:/var/lib/postgresql
    networks:
      - hw_net
    environment:
      POSTGRES_PASSWORD: password

networks:
    hw_net:
        name: hw_net

```

Запускаем контейнер
```sh
docker compose up -d
```

На хосте
- проверяем файл /var/lib/postgresql/18/docker/pg_hba.conf на наличие записи (если нет добавляем)
```
host all all all scram-sha-256
```

- проверяем файл /var/lib/postgresql/18/docker/postgresql.conf
строка должна быть раскомментирована
```
listen_addresses = '*'
```

Если данные менялись то перезапустим контейнер
```sh
docker compose restart
```

### 3. Запуск клиента (psql)

Запускаем psql в docker, создаем БД и таблицу
```sh
user@ubuntu01:~/otus/hw01$ docker run -it --rm --network hw_net postgres:18 psql -h postgres18 -U postgres
Password for user postgres:
psql (18.6 (Debian 18.6-1.pgdg13+2))
Type "help" for help.

postgres=# create database hw01;
CREATE DATABASE
postgres=# \c hw01
You are now connected to database "hw01" as user "postgres".
hw01=# create table orders_test (order_id int, order_info text);
CREATE TABLE
hw01=# insert into orders_test values (1, 'order 1 info'),(2, 'order 2 info'),(3, 'order 3 info');
INSERT 0 3
hw01=# quit
```

Запускаем клиента с хоста и проверяем наличие записей
```sh
WSL$ psql -h 192.168.0.62 -U postgres -d hw01
Password for user postgres:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

hw01=# select * from orders_test;
 order_id |  order_info
----------+--------------
        1 | order 1 info
        2 | order 2 info
        3 | order 3 info
(3 rows)
```

### 4. Перезапуск контейнера и проверка

Остановка и запуск
```sh
user@ubuntu01:~/otus/hw01$ docker compose down
[+] down 2/2
 ✔ Container postgres18 Removed                                                                                                                                                             0.6s
 ✔ Network hw_net       Removed                                                                                                                                                             0.1s
user@ubuntu01:~/otus/hw01$ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
user@ubuntu01:~/otus/hw01$

user@ubuntu01:~/otus/hw01$ docker compose up -d
[+] up 2/2
 ✔ Network hw_net       Created                                                                                                                                                             0.1s
 ✔ Container postgres18 Started                                                                                                                                                             0.6s
user@ubuntu01:~/otus/hw01$
```

Проверка из контейнера
```sh
user@ubuntu01:~/otus/hw01$ docker run -it --rm --network hw_net postgres:18 psql -h postgres18 -U postgres -d hw01
Password for user postgres:
psql (18.6 (Debian 18.6-1.pgdg13+2))
Type "help" for help.

hw01=# select * from orders_test;
 order_id |  order_info
----------+--------------
        1 | order 1 info
        2 | order 2 info
        3 | order 3 info
(3 rows)

hw01=#
```

Проверка с хоста
```sh
WSL$ psql -h 192.168.0.62 -U postgres -d hw01
Password for user postgres:
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

hw01=# select * from orders_test;
 order_id |  order_info
----------+--------------
        1 | order 1 info
        2 | order 2 info
        3 | order 3 info
(3 rows)
```
