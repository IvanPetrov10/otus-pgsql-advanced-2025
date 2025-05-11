Создал ВМ в облаке Yandex Cloud на РЕД ОС 8

Настроил доступ по SSH

Подключился по ssh к виртуалке с домашнего компьютера
[root@vml-terminal:~]# ssh -l admin <IP>
[admin@vml-yc-pgsql-db-01 ~]$

Подключился под рутом
[admin@vml-yc-pgsql-db-01 ~]$ sudo -s

Установил и запустил standalone PostgreSQL 16

Переключился на пользователя ОС postgres
[postgres@vml-yc-pgsql-db-01:~]$ 

Создал БД otus_postgresql_advanced_2025
postgres=# CREATE DATABASE otus_postgresql_advanced_2025;



Ответы на домашнее задание:
Первая сессия
postgres=# \c otus_postgresql_advanced_2025;
otus_postgresql_advanced_2025=# create table shipments(id serial, product_name text, quantity int, destination text);
otus_postgresql_advanced_2025=# insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
otus_postgresql_advanced_2025=# insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
otus_postgresql_advanced_2025=# commit;

otus_postgresql_advanced_2025=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed

otus_postgresql_advanced_2025=# \set AUTOCOMMIT off
otus_postgresql_advanced_2025=# insert into shipments(product_name, quantity, destination) values('sugar', 300, 'Asia');



Вторая сессия
otus_postgresql_advanced_2025=# \set AUTOCOMMIT off
| otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA

Во второй сессии не видно новой строки поскольку транзакция не была закоммичена в первой сессии, 
а в PostgreSQL невозможно грязное чтение, мимнимальный уровень изоляции read committed (стандартный).
Грязное чтение в целом допустимо при уровне изоляции read uncommitted, которого нет в СУБД PostgreSQL (точнее он ведет себя как read committed)


Первая сессия
otus_postgresql_advanced_2025=# commit;


Вторая сессия
| otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia

Строка видна поскольку при данном уровне изоляции снимок делается при начале выполнения ОПЕРАТОРА,
и на этот момент транзакция в первой сессии уже была закоммичена.


Первая сессия
otus_postgresql_advanced_2025=# set transaction isolation level repeatable read;
otus_postgresql_advanced_2025=# insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');


Вторая сессия
| otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia

Строка во второй сессии не видна, поскольку коммита еще не было


Первая сессия
otus_postgresql_advanced_2025=# commit;


Вторая сессия
| otus_postgresql_advanced_2025=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia


Во второй сессии запись по-прежнему не видна, поскольку при данном уровне изоляции снимок выполняется на момент начала ТРАНЗАКЦИИ, 
а на момент начала транзакции во второй сессии коммит еще не был выполнен










