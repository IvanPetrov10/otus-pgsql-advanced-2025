1. Создал ВМ в облаке Yandex Cloud на РЕД ОС 8


2. Настроил доступ по SSH c домашнего компьютера (создал ключи и добавил публичный в облако)
```bash
[root@vml-terminal:~/.ssh]# ssh-keygen -t rsa -b 4096
```

   Подключился по SSH к виртуалке в Yandex Cloud с домашнего компьютера
```bash
[root@vml-terminal:~]# ssh -l admin <IP>
[admin@vml-yc-pgsql-db-01 ~]$
```

   Подключился под root
```bash
[admin@vml-yc-pgsql-db-01 ~]$ sudo -s
[root@vml-yc-pgsql-db-01:/home/admin]# 
```

3. Установил и запустил standalone PostgreSQL 16 
```bash
[root@vml-yc-pgsql-db-01:~]# dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@vml-yc-pgsql-db-01:~]# dnf -qy module disable postgresql
[root@vml-yc-pgsql-db-01:~]# dnf install -y postgresql16-server
[root@vml-yc-pgsql-db-01:~]# /usr/pgsql-16/bin/postgresql-16-setup initdb
```

4. Переключился на пользователя ОС postgres
```bash
[root@vml-yc-pgsql-db-01:/home/admin]# su - postgres
[postgres@vml-yc-pgsql-db-01:~]$ 
```

5. Работа с транзакциями

   Создал БД otus_postgresql_advanced_2025
```sql
postgres=# CREATE DATABASE otus_postgresql_advanced_2025;
postgres=# \c otus_postgresql_advanced_2025
```

```sql
otus_postgresql_advanced_2025=# create table shipments(id serial, product_name text, quantity int, destination text);
insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
commit;
```

6. Изучение уровней изоляции

```sql
otus_postgresql_advanced_2025=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed

otus_postgresql_advanced_2025=# \set AUTOCOMMIT off
otus_postgresql_advanced_2025=# insert into shipments(product_name, quantity, destination) values('sugar', 300, 'Asia');
```

Вторая сессия
```sql
otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
```

Во второй сессии не видно новой строки поскольку транзакция не была закоммичена в первой сессии, 
а в PostgreSQL невозможно грязное чтение, мимнимальный уровень изоляции read committed (стандартный).
Грязное чтение в целом допустимо при уровне изоляции read uncommitted, которого нет в СУБД PostgreSQL (точнее он ведет себя как read committed)


Первая сессия
```sql
otus_postgresql_advanced_2025=# commit;
COMMIT
```


Вторая сессия
```sql
otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
```

Строка видна поскольку при данном уровне изоляции снимок делается при начале выполнения ОПЕРАТОРА,
и на этот момент транзакция в первой сессии уже была закоммичена.



7. Эксперименты с уровнем изоляции Repeatable Read

```sql
otus_postgresql_advanced_2025=# set transaction isolation level repeatable read;
otus_postgresql_advanced_2025=# insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');

| otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
```
Строка во второй сессии не видна, поскольку коммита еще не было


```sql
otus_postgresql_advanced_2025=# commit;
```

Вторая сессия
```sql
otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
```

Во второй сессии запись по-прежнему не видна, поскольку при данном уровне изоляции снимок выполняется на момент начала ТРАНЗАКЦИИ, 
а на момент начала транзакции во второй сессии, коммит в первой сессии еще не был выполнен





