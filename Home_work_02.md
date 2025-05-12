1. Создал ВМ в Yandex Cloud.

2. Установил Docker Engine. https://docs.docker.com/engine/install/rhel/

```bash
[root@vml-yc-docker-01:~]# sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc

[root@vml-yc-docker-01:~]# dnf -y install dnf-plugins-core
[root@vml-yc-docker-01:~]# dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
[root@vml-yc-docker-01:~]# dnf install docker-ce docker-ce-cli containerd.io
[root@vml-yc-docker-01:~]# systemctl enable --now docker

[root@vml-yc-docker-01:~]# docker run hello-world

```

3. Создал каталог /var/lib/postgres для хранения данных.

```bash
[root@vml-yc-docker-01:~]# mkdir -p /var/lib/postgres
```

4. Развернул контейнер с PostgreSQL 14, смонтировал в него /var/lib/postgres

```bash
[root@vml-yc-docker-01:~]# docker pull postgres:14
[root@vml-yc-docker-01:~]# docker run -d -e POSTGRES_PASSWORD=postgres --name postgres14 -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

[root@vml-yc-docker-01:~]# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                      PORTS                                       NAMES
de920c673d24   postgres:14   "docker-entrypoint.s…"   5 seconds ago    Up 5 seconds                0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   postgres14
f92d49b3ef16   hello-world   "/hello"                 32 minutes ago   Exited (0) 32 minutes ago                                               frosty_jones

[root@vml-yc-docker-01:~]# docker run -it --rm postgres:14 bash
root@ebb8a59cfe58:/# su - postgres
postgres@ebb8a59cfe58:~$ psql -p 5432 -U postgres -h 10.129.0.10 -d postgres -W
Password:
psql (14.18 (Debian 14.18-1.pgdg120+1))
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres

```

5. Развернул контейнер с клиентом PostgreSQL

```bash
[root@vml-yc-docker-01:~]# mkdir psql-client
[root@vml-yc-docker-01:~]# cd psql-client
[root@vml-yc-docker-01:~/psql-client]# vim Dockerfile

[root@vml-yc-docker-01:~/psql-client]# cat Dockerfile
FROM alpine:latest

RUN apk add --no-cache postgresql-client

ENTRYPOINT ["psql"]

[root@vml-yc-docker-01:~/psql-client]# docker build -t psql-client .
[+] Building 6.2s (6/6) FINISHED                                                                                                                                                                                                          docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                                                                0.1s
 => => transferring dockerfile: 119B                                                                                                                                                                                                                0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                                                                                                                                    2.2s
 => [internal] load .dockerignore                                                                                                                                                                                                                   0.2s
 => => transferring context: 2B                                                                                                                                                                                                                     0.0s
 => [1/2] FROM docker.io/library/alpine:latest@sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c                                                                                                                              1.1s
 => => resolve docker.io/library/alpine:latest@sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c                                                                                                                              0.1s
 => => sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c 9.22kB / 9.22kB                                                                                                                                                      0.0s
 => => sha256:1c4eef651f65e2f7daee7ee785882ac164b02b78fb74503052a26dc061c90474 1.02kB / 1.02kB                                                                                                                                                      0.0s
 => => sha256:aded1e1a5b3705116fa0a92ba074a5e0b0031647d9c315983ccba2ee5428ec8b 581B / 581B                                                                                                                                                          0.0s
 => => sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870 3.64MB / 3.64MB                                                                                                                                                      0.5s
 => => extracting sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870                                                                                                                                                           0.1s
 => [2/2] RUN apk add --no-cache postgresql-client                                                                                                                                                                                                  2.1s
 => exporting to image                                                                                                                                                                                                                              0.2s
 => => exporting layers                                                                                                                                                                                                                             0.1s
 => => writing image sha256:33c83f6d3e4ba1f42e345db3b6afd154096749af8f8401ba76799b8f9308a88a                                                                                                                                                        0.0s
 => => naming to docker.io/library/psql-client     
                                                                                                                                                                                                  0.0s
[root@vml-yc-docker-01:~/psql-client]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
psql-client   latest    33c83f6d3e4b   3 seconds ago   12.9MB
postgres      14        fd70450ab5f7   4 days ago      426MB
hello-world   latest    74cc54e27dc4   3 months ago    10.1kB
```

6. Подключился из контейнера с клиентом к контейнеру с сервером и создал таблицу с данными о перевозках

```bash
[root@vml-yc-docker-01:~/psql-client]# docker run -it --rm psql-client -U postgres -h 10.129.0.10 -d postgres -W
Password:
psql (17.4, server 14.18 (Debian 14.18-1.pgdg120+1))
Type "help" for help.

postgres=#
```

```sql
postgres=# create database otus_home_work_2;
postgres=# \c otus_home_work_2
otus_home_work_2=# 

create table shipments(id serial, product_name text, quantity int, destination text);

insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('bananas', 1500, 'Asia');
insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
insert into shipments(product_name, quantity, destination) values('coffee', 700, 'Canada');
insert into shipments(product_name, quantity, destination) values('coffee', 300, 'Japan');
insert into shipments(product_name, quantity, destination) values('sugar', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('sugar', 800, 'Asia');
insert into shipments(product_name, quantity, destination) values('sugar', 600, 'Africa');
insert into shipments(product_name, quantity, destination) values('sugar', 400, 'USA');
```

7. Подключился к контейнеру с сервером с домашнего компьютера

```bash
[root@vml-terminal:~]# pgcli -p 5432 -U postgres -h 89.169.175.197 -d postgres -W
Password for postgres: 
Server: PostgreSQL 14.18 (Debian 14.18-1.pgdg120+1)
Version: 4.1.0
Home: http://pgcli.com
postgres>
postgres> \l
+------------------+----------+----------+------------+------------+-----------------------+
| Name             | Owner    | Encoding | Collate    | Ctype      | Access privileges     |
|------------------+----------+----------+------------+------------+-----------------------|
| otus_home_work_2 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | <null>                |
| postgres         | postgres | UTF8     | en_US.utf8 | en_US.utf8 | <null>                |
| template0        | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres           |
|                  |          |          |            |            | postgres=CTc/postgres |
| template1        | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres           |
|                  |          |          |            |            | postgres=CTc/postgres |
+------------------+----------+----------+------------+------------+-----------------------+
```

8. Удалил контейнер с сервером и создал его заново.

```bash
[root@vml-yc-docker-01:~]# docker rm -f postgres14

[root@vml-yc-docker-01:~/psql-client]# docker ps -a
CONTAINER ID   IMAGE         COMMAND    CREATED             STATUS                         PORTS     NAMES
f92d49b3ef16   hello-world   "/hello"   About an hour ago   Exited (0) About an hour ago             frosty_jones

[root@vml-yc-docker-01:~]# docker run -d -e POSTGRES_PASSWORD=postgres --name postgres14 -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
```

9. Проверил, что данные остались на месте

```sql
otus_home_work_2=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 rows)
```



