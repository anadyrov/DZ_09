# DZ_09

-- Создал ВМ на GCP
NAME: test1
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
INTERNAL_IP: 10.128.0.15
EXTERNAL_IP: 35.239.207.100
STATUS: RUNNING

NAME: test2
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
INTERNAL_IP: 10.128.0.16
EXTERNAL_IP: 34.136.176.253
STATUS: RUNNING

NAME: test3
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
INTERNAL_IP: 10.128.0.17
EXTERNAL_IP: 35.232.80.148
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
CREATE PUBLICATION test_pub FOR TABLE test2;
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
 test_sub  | postgres | t       | {test_pub}
 test_sub2 | postgres | t       | {test_pub}
(2 rows)


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

