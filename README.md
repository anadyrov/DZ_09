# DZ_09

-- Создал ВМ на GCP
NAME: test1
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.18
EXTERNAL_IP: 35.232.80.148
STATUS: RUNNING

NAME: test2
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.19
EXTERNAL_IP: 34.136.176.253
STATUS: RUNNING

NAME: test3
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.20
EXTERNAL_IP: 35.239.207.100
STATUS: RUNNING

NAME: test4
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.21
EXTERNAL_IP: 34.135.30.123
STATUS: RUNNING


-- На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. 
-- Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. 

-- На всех ВМ установлена postgresql 14
postgres@test1:/home/anadyrov$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

-- Разрешаем доступ, прописываем адреса в конфиг файлах и рестарутем кластер
/etc/postgresql/14/main/pg_hba.conf
/etc/postgresql/14/main/postgresql.conf

-- Меням на логическую репликацию
postgres=# show wal_level;
 wal_level 
-----------
 logical
(1 row)

-- Создаем базу и таблицу на всех ВМ
postgres=# create database laba9;
CREATE DATABASE
postgres=# \c laba9 
You are now connected to database "laba9" as user "postgres".

CREATE TABLE test(i int);
insert into test values('1');
insert into test values('10');

--  Создаем публикацию:
CREATE TABLE test(i int);
CREATE PUBLICATION test_pub FOR TABLE test;
\dRp+
\password

-- Проверяем:
laba9=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"

--  Создаем подписку:
CREATE TABLE test(i int);
CREATE SUBSCRIPTION test_sub
CONNECTION 'host=test1 port=5432 user=postgres password=postgres dbname=laba9' 
PUBLICATION test_pub WITH (copy_data = true);

-- Проверяем:
laba9=# \dRs
            List of subscriptions
   Name   |  Owner   | Enabled | Publication 
----------+----------+---------+-------------
 test_sub | postgres | t       | {test_pub}
(1 row)


-- На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. 
-- Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. 

--  Создаем публикацию:
CREATE TABLE test2(i int);
CREATE PUBLICATION test_pub2 FOR TABLE test2;
insert into test2 values('2');
insert into test2 values('20');
\dRp+
\password

-- Проверяем:
laba9=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"


--  Создаем подписку:
CREATE TABLE test2(i int);
CREATE SUBSCRIPTION test_sub2
CONNECTION 'host=test2 port=5432 user=postgres password=postgres dbname=laba9' 
PUBLICATION test_pub WITH (copy_data = true);

-- Проверяем:
laba9=# \dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication 
-----------+----------+---------+-------------
 test_sub2 | postgres | t       | {test_pub}
(1 rows)


-- 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). 
-- Небольшое описание, того, что получилось.

--  Создаем подписку к первой ВМ:
create database laba9;
CREATE DATABASE
postgres=# \c laba9 

CREATE TABLE test(i int);
CREATE SUBSCRIPTION test_sub3
CONNECTION 'host=test1 port=5432 user=postgres password=postgres dbname=laba9' 
PUBLICATION test_pub WITH (copy_data = true);

-- Проверяем:
laba9=# \dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication 
-----------+----------+---------+-------------
 test_sub3 | postgres | t       | {test_pub}
(1 row)



--  Создаем подписку ко второй ВМ:
CREATE TABLE test2(i int);
CREATE SUBSCRIPTION test_sub4
CONNECTION 'host=test2 port=5432 user=postgres password=postgres dbname=laba9' 
PUBLICATION test_pub WITH (copy_data = true);
laba9=# \dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication 
-----------+----------+---------+-------------
 test_sub3 | postgres | t       | {test_pub}
 test_sub4 | postgres | t       | {test_pub}
(2 rows)

-- Данные по таблицам среплецированны.


-- Задание со звездочкой*
-- реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

-- Изначально на ВМ3 создал пользователя для репликации
postgres=# create user replicator WITH REPLICATION ENCRYPTED PASSWORD '1KyPiA1';

-- Разрешил подключение
echo "host replication replicator 1.0/24 md5" >> $PGDATA/pg_hba.conf


-- На ВМ4 развернул кластер Postgresql 

mv /var/lib/postgresql/14/main/ /var/lib/postgresql/14/main_old
mkdir /var/lib/postgresql/14/main

-- Запускаю утилиту pg_basebackup помимо переноса базы создаст в папке важный файл $PGDATA/standby.signal
pg_basebackup -h test3 -U replicator -p 5432 -D $PGDATA -Fp -Xs -P -R

-- После стартуем кластер и проверяем:

-- На мастере:
postgres@test3:/home/anadyrov$ psql 
psql (14.2 (Ubuntu 14.2-1.pgdg18.04+1))
Type "help" for help.

postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 18398
usesysid         | 16393
usename          | replicator
application_name | 14/main
client_addr      | 10.128.0.21
client_hostname  | 
client_port      | 42698
backend_start    | 2022-04-19 08:12:19.885315+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/5000060
write_lsn        | 0/5000060
flush_lsn        | 0/5000060
replay_lsn       | 0/5000060
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2022-04-19 08:12:49.921284+00

postgres=# SELECT * FROM pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/5000060
(1 row)

-- На реплике:
postgres=# select * from pg_stat_wal_receiver \gx
-[ RECORD 1 ]---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 17971
status                | streaming
receive_start_lsn     | 0/5000000
receive_start_tli     | 1
written_lsn           | 0/5000060
flushed_lsn           | 0/5000060
received_tli          | 1
last_msg_send_time    | 2022-04-19 08:12:49.922717+00
last_msg_receipt_time | 2022-04-19 08:12:49.92385+00
latest_end_lsn        | 0/5000060
latest_end_time       | 2022-04-19 08:12:19.899116+00
slot_name             | 
sender_host           | test3
sender_port           | 5432
conninfo              | user=replicator password=******** channel_binding=prefer dbname=replication host=test3 port=5432 fallback_application_name=14/main sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any


postgres=# select pg_last_wal_receive_lsn();
 pg_last_wal_receive_lsn 
-------------------------
 0/5000060
(1 row)

psql (14.2 (Ubuntu 14.2-1.pgdg18.04+1))
Type "help" for help.

postgres=# show hot_standby;
 hot_standby 
-------------
 on
(1 row)

