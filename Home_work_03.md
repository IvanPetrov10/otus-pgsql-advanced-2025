1. Создал виртуальную машину с Ред ОС 8 и установил PostgreSQL 16
```bash
## Создал ВМ и подключился по SSH
[root@vml-terminal:~]# ssh -l admin <IP>

## Установил и запустил standalone PostgreSQL 16 https://www.postgresql.org/download/linux/redhat/
[root@vml-yc-disk:~]# dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@vml-yc-disk:~]# dnf -qy module disable postgresql
[root@vml-yc-disk:~]# dnf install -y postgresql16-server
[root@vml-yc-disk:~]# /usr/pgsql-16/bin/postgresql-16-setup initdb

[postgres@vml-yc-disk:~]$ uname -a
# Linux vml-yc-disk.ru-central1.internal 6.6.51-1.red80.x86_64 #1 SMP PREEMPT_DYNAMIC Sun Sep 15 22:29:03 MSK 2024 x86_64 x86_64 x86_64 GNU/Linux

[root@vml-yc-disk:~]# systemctl enable postgresql-16
# Created symlink /etc/systemd/system/multi-user.target.wants/postgresql-16.service → /usr/lib/systemd/system/postgresql-16.service.
[root@vml-yc-disk:~]# systemctl start postgresql-16
[root@vml-yc-disk:~]# systemctl status postgresql-16
#● postgresql-16.service - PostgreSQL 16 database server
#     Loaded: loaded (/usr/lib/systemd/system/postgresql-16.service; enabled; preset: disabled)
#     Active: active (running) since Mon 2025-05-19 21:43:46 MSK; 11s ago
#       Docs: https://www.postgresql.org/docs/16/static/
#    Process: 14583 ExecStartPre=/usr/pgsql-16/bin/postgresql-16-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
#   Main PID: 14588 (postgres)
#      Tasks: 7 (limit: 2329)
#     Memory: 17.3M
#        CPU: 40ms
#     CGroup: /system.slice/postgresql-16.service
#             ├─14588 /usr/pgsql-16/bin/postgres -D /var/lib/pgsql/16/data/
#             ├─14590 "postgres: logger "
#             ├─14591 "postgres: checkpointer "
#             ├─14592 "postgres: background writer "
#             ├─14594 "postgres: walwriter "
#             ├─14595 "postgres: autovacuum launcher "
#             └─14596 "postgres: logical replication launcher "
#
#мая 19 21:43:46 vml-yc-disk.ru-central1.internal systemd[1]: Starting postgresql-16.service - PostgreSQL 16 database server...
#мая 19 21:43:46 vml-yc-disk.ru-central1.internal postgres[14588]: 2025-05-19 21:43:46.707 MSK [14588] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
#мая 19 21:43:46 vml-yc-disk.ru-central1.internal postgres[14588]: 2025-05-19 21:43:46.707 MSK [14588] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
#мая 19 21:43:46 vml-yc-disk.ru-central1.internal systemd[1]: Started postgresql-16.service - PostgreSQL 16 database server.
```

2. Создал таблицу с данными о перевозках
```bash
[postgres@vml-yc-disk:~]$ psql
postgres=# create database otus_home_work_3;
postgres=# \c otus_home_work_3
otus_home_work_2=# 
```

```sql
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

otus_home_work_3=# SELECT * FROM shipments;
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
(10 строк)
```

3. Добавил внешний диск к виртуальной машине и перенес туда базу данных
```bash
## Добавил в облаке диск
[root@vml-yc-disk:~]# lsblk
#NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
#zram0  251:0    0  1,9G  0 disk [SWAP]
#vda    252:0    0   30G  0 disk
#├─vda1 252:1    0    1M  0 part
#└─vda2 252:2    0   30G  0 part /
#vdb    252:16   0   30G  0 disk

[root@vml-yc-disk:~]# pvcreate /dev/vdb
#  Physical volume "/dev/vdb" successfully created.

[root@vml-yc-disk:~]# pvs
#  PV         VG Fmt  Attr PSize  PFree
#  /dev/vdb      lvm2 ---  30,00g 30,00g

[root@vml-yc-disk:~]# vgcreate vg_db /dev/vdb
#  Volume group "vg_db" successfully created

[root@vml-yc-disk:~]# lvcreate -l 100%FREE -n lv_db vg_db
#  Logical volume "lv_db" created.

[root@vml-yc-disk:~]# lvs
#  LV    VG    Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
#  lv_db vg_db -wi-a----- <30,00g

[root@vml-yc-disk:~]# mkfs.ext4 /dev/vg_db/lv_db
#mke2fs 1.46.3 (27-Jul-2021)
#Creating filesystem with 7863296 4k blocks and 1966080 inodes
#Filesystem UUID: d10d7eaf-4301-4ff5-be83-8d1f44975ba1
#Superblock backups stored on blocks:
#        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
#        4096000
#
#Allocating group tables: done
#Writing inode tables: done
#Creating journal (32768 blocks): done
#Writing superblocks and filesystem accounting information: done

[root@vml-yc-disk:~]# vim /etc/fstab
[root@vml-yc-disk:~]# tail -n 1 /etc/fstab
#/dev/mapper/vg_db-lv_db /pg_data                ext4    defaults        1 2

[root@vml-yc-disk:~]# mkdir /pg_data

[root@vml-yc-disk:~]# mount -a
#mount: (hint) your fstab has been modified, but systemd still uses
#       the old version; use 'systemctl daemon-reload' to reload.

[root@vml-yc-disk:~]# systemctl daemon-reload

[root@vml-yc-disk:~]# df -h
#Файловая система        Размер Использовано  Дост Использовано% Cмонтировано в
#devtmpfs                  4,0M            0  4,0M            0% /dev
#tmpfs                     984M         1,1M  983M            1% /dev/shm
#tmpfs                     394M         7,9M  386M            3% /run
#/dev/vda2                  30G         2,5G   26G            9% /
#tmpfs                     984M         4,0K  984M            1% /tmp
#tmpfs                     197M            0  197M            0% /run/user/1000
#/dev/mapper/vg_db-lv_db    30G          24K   28G            1% /pg_data

[root@vml-yc-disk:~]# lsblk
#NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
#zram0         251:0    0  1,9G  0 disk [SWAP]
#vda           252:0    0   30G  0 disk
#├─vda1        252:1    0    1M  0 part
#└─vda2        252:2    0   30G  0 part /
#vdb           252:16   0   30G  0 disk
#└─vg_db-lv_db 253:0    0   30G  0 lvm  /pg_data

[root@vml-yc-disk:~]# systemctl stop postgresql-16

[root@vml-yc-disk:~]# chown -R postgres:postgres /pg_data/

[root@vml-yc-disk:~]# chmod 700 -R /pg_data/

[root@vml-yc-disk:~]# mv /var/lib/pgsql/16/data/* /pg_data/

[root@vml-yc-disk:~]# ll /pg_data/
#итого 148
#drwx------. 6 postgres postgres  4096 мая 19 21:51 base
#-rw-------. 1 postgres postgres    30 мая 19 21:43 current_logfiles
#drwx------. 2 postgres postgres  4096 мая 19 21:51 global
#drwx------. 2 postgres postgres  4096 мая 19 21:43 log
#drwx------. 2 postgres postgres 16384 мая 19 21:59 lost+found
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_commit_ts
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_dynshmem
#-rw-------. 1 postgres postgres  5499 мая 19 21:41 pg_hba.conf
#-rw-------. 1 postgres postgres  2640 мая 19 21:41 pg_ident.conf
#drwx------. 4 postgres postgres  4096 мая 19 22:01 pg_logical
#drwx------. 4 postgres postgres  4096 мая 19 21:41 pg_multixact
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_notify
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_replslot
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_serial
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_snapshots
#drwx------. 2 postgres postgres  4096 мая 19 22:01 pg_stat
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_stat_tmp
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_subtrans
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_tblspc
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_twophase
#-rw-------. 1 postgres postgres     3 мая 19 21:41 PG_VERSION
#drwx------. 3 postgres postgres  4096 мая 19 21:41 pg_wal
#drwx------. 2 postgres postgres  4096 мая 19 21:41 pg_xact
#-rw-------. 1 postgres postgres    88 мая 19 21:41 postgresql.auto.conf
#-rw-------. 1 postgres postgres 29692 мая 19 21:41 postgresql.conf
#-rw-------. 1 postgres postgres    58 мая 19 21:43 postmaster.opts
```

4. Настроил PostgreSQL для работы с новым диском
```bash
[root@vml-yc-disk:~]# vim /usr/lib/systemd/system/postgresql-16.service
[root@vml-yc-disk:~]# cat /usr/lib/systemd/system/postgresql-16.service | grep -i pgdata
## Note: changing PGDATA will typically require adjusting SELinux
## Note: do not use a PGDATA pathname containing spaces, or you will
##Environment=PGDATA=/var/lib/pgsql/16/data/
#Environment=PGDATA=/pg_data/
#ExecStartPre=/usr/pgsql-16/bin/postgresql-16-check-db-dir ${PGDATA}
#ExecStart=/usr/pgsql-16/bin/postgres -D ${PGDATA}

[root@vml-yc-disk:~]# systemctl daemon-reload
[root@vml-yc-disk:~]# systemctl start postgresql-16
[root@vml-yc-disk:~]# systemctl status postgresql-16
#● postgresql-16.service - PostgreSQL 16 database server
#     Loaded: loaded (/usr/lib/systemd/system/postgresql-16.service; enabled; preset: disabled)
#     Active: active (running) since Mon 2025-05-19 22:04:45 MSK; 7s ago
#       Docs: https://www.postgresql.org/docs/16/static/
#    Process: 15167 ExecStartPre=/usr/pgsql-16/bin/postgresql-16-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
#   Main PID: 15175 (postgres)
#      Tasks: 7 (limit: 2329)
#     Memory: 17.4M
#        CPU: 43ms
#     CGroup: /system.slice/postgresql-16.service
#             ├─15175 /usr/pgsql-16/bin/postgres -D /pg_data/
#             ├─15179 "postgres: logger "
#             ├─15180 "postgres: checkpointer "
#             ├─15181 "postgres: background writer "
#             ├─15183 "postgres: walwriter "
#             ├─15184 "postgres: autovacuum launcher "
#             └─15185 "postgres: logical replication launcher "
#мая 19 22:04:44 vml-yc-disk.ru-central1.internal systemd[1]: Starting postgresql-16.service - PostgreSQL 16 database server...
#мая 19 22:04:45 vml-yc-disk.ru-central1.internal postgres[15175]: 2025-05-19 22:04:45.084 MSK [15175] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
#мая 19 22:04:45 vml-yc-disk.ru-central1.internal postgres[15175]: 2025-05-19 22:04:45.084 MSK [15175] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
#мая 19 22:04:45 vml-yc-disk.ru-central1.internal systemd[1]: Started postgresql-16.service - PostgreSQL 16 database server.
```

5. Проверил, что данные сохранились и доступны
```bash
[root@vml-yc-disk:~]# su - postgres
[postgres@vml-yc-disk:~]$ psql
```
```sql
postgres=# \c otus_home_work_3

otus_home_work_3=# SELECT * FROM shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
```
