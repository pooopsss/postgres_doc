### Репликация #10

#### 1. создаем таблицы
```bash
#server_1
docker exec -ti postgres_otus_db_2 bash
su - postgres
psql otus -W -d otus
```


```sql
-- server_1
create table test1 as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;


create table test2 as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;
```

#### Логическая репликация
```sql
-- server_1
ALTER SYSTEM SET wal_level = logical;

-- RESTART SERVER
show wal_level;
 wal_level 
-----------
 logical
(1 row)

-- Создаем публикацию на test1.Теперь на нее можно подписываться
CREATE PUBLICATION test_pub FOR TABLE test1;
--Просмотр созданной публикации 
\dRp+

Publication test_pub
 Owner | All tables | Inserts | Updates | Deletes | Truncates | Via root 
-------+------------+---------+---------+---------+-----------+----------
 otus  | f          | t       | t       | t       | t         | f
Tables:
    "public.test1"

```


```sql
-- server 2
-- создаем такую же таблицу
create table test1 as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;

TRUNCATE TABLE test1;

--создадим подписку
CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=db1 port=5432 user=otus password=123456 dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);



SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 54
usesysid         | 10
usename          | otus
application_name | test_sub
client_addr      | 192.168.224.3
client_hostname  | 
client_port      | 49344
backend_start    | 2023-12-14 19:30:00.871264+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/199A028
write_lsn        | 0/199A028
flush_lsn        | 0/199A028
replay_lsn       | 0/199A028
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2023-12-14 19:33:21.165024+00


```

> Репликация идет все отлично

#### Подвязываем к первой машине test2 со второй

```sql
--server 2
create table test2 as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;


ALTER SYSTEM SET wal_level = logical;
--restart cluster

otus=# show wal_level;
 wal_level 
-----------
 logical
(1 row)


-- Делаем с test2 publication
create publication test2_pub for table test2;
\dRp+
                          Publication test2_pub
 Owner | All tables | Inserts | Updates | Deletes | Truncates | Via root 
-------+------------+---------+---------+---------+-----------+----------
 otus  | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
```

##### Подписываемся на test2 с server2->server1
```sql
--server1
create subscription test2_sub
otus-# connection 'host=db2 port=5432 user=otus password=123456 dbname=otus' 
otus-# publication test2_pub with (copy_data  =true);


```


```sql
server2
select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 51
usesysid         | 10
usename          | otus
application_name | test2_sub
client_addr      | 192.168.224.2
client_hostname  | 
client_port      | 55168
backend_start    | 2023-12-21 14:19:37.874646+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/19F7D20
write_lsn        | 0/19F7D20
flush_lsn        | 0/19F7D20
replay_lsn       | 0/19F7D20
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2023-12-21 14:20:38.012954+00

```

> Подписали таблицу server2.test2 -> server1.test2. Проверим

```sql
--server2
insert into test2 (id, fio) values(11, 'insSrv2');
select * from test2;
 id |    fio     
----+------------
  1 | 5d73472142
  2 | a9e7819f54
  3 | 13e17118cb
  4 | 39aac30f56
  5 | 14caab4f01
  6 | 37c9efc088
  7 | 04fe137517
  8 | ecb175922a
  9 | fcf235b4ad
 10 | 2a11c7e195
 11 | insSrv2   
(11 rows)


--server1
select * from test2;
 id |    fio     
----+------------
  1 | 5d73472142
  2 | a9e7819f54
  3 | 13e17118cb
  4 | 39aac30f56
  5 | 14caab4f01
  6 | 37c9efc088
  7 | 04fe137517
  8 | ecb175922a
  9 | fcf235b4ad
 10 | 2a11c7e195
 11 | insSrv2   
(11 rows)

```

#### Добавлем третюю машину и подписываем server3.test <- server1.test, server2.test2 <- server2.test2

```sql
--server 3
create table test1 as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;

create table test2 as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;


TRUNCATE TABLE test1;
TRUNCATE TABLE test2;


--создадим подписку
CREATE SUBSCRIPTION test3_sub 
CONNECTION 'host=db1 port=5432 user=otus password=123456 dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);


create subscription test3_2_sub
connection 'host=db2 port=5432 user=otus password=123456 dbname=otus' 
publication test2_pub with (copy_data  =true);

```

> Проверяем подписки с сервером 3

```sql
-- server1
insert into test1 (id, fio) values(12, 'srv1_12');

--server 2
select * from test1;
 id |    fio     
----+------------
  1 | e22583e70b
  2 | 38f0d302f7
  3 | 59390e01c8
  4 | 44ab79ac15
  5 | 731e577811
  6 | 5e0249a22e
  7 | d27ccdfb19
  8 | 5bfc7b4864
  9 | 80855a90de
 10 | 8f46c5502b
 12 | srv1_12   
(11 rows)

--server3
select * from test1;
 id |    fio     
----+------------
  1 | e22583e70b
  2 | 38f0d302f7
  3 | 59390e01c8
  4 | 44ab79ac15
  5 | 731e577811
  6 | 5e0249a22e
  7 | d27ccdfb19
  8 | 5bfc7b4864
  9 | 80855a90de
 10 | 8f46c5502b
 12 | srv1_12   
(11 rows)


```


```sql
-- server2
insert into test2 (id, fio) values(13, 'srv2_13');

-- server1
select * from test2;
 id |    fio     
----+------------
  1 | 5d73472142
  2 | a9e7819f54
  3 | 13e17118cb
  4 | 39aac30f56
  5 | 14caab4f01
  6 | 37c9efc088
  7 | 04fe137517
  8 | ecb175922a
  9 | fcf235b4ad
 10 | 2a11c7e195
 11 | insSrv2   
 13 | srv2_13   
(12 rows)


-- server 3
select * from test2;
 id |    fio     
----+------------
  1 | 5d73472142
  2 | a9e7819f54
  3 | 13e17118cb
  4 | 39aac30f56
  5 | 14caab4f01
  6 | 37c9efc088
  7 | 04fe137517
  8 | ecb175922a
  9 | fcf235b4ad
 10 | 2a11c7e195
 11 | insSrv2   
 13 | srv2_13   
(12 rows)

```




#### Физическая репликация server3 -> server4
```sql
-- server4
show wal_level;                        
 wal_level 
-----------
 replica
(1 row)

```

- wal_level = replica
- max_wal_senders = 2
- max_replication_slots = 2
- hot_standby = on
- hot_standby_feedback = on

** где

- **wal_level** указывает, сколько информации записывается в WAL (журнал операций, который используется для репликации). Значение replica указывает на необходимость записывать только данные для поддержки архивирования WAL и репликации.
- **max_wal_senders** — количество планируемых слейвов; 
- **max_replication_slots** — максимальное число слотов репликации (данный параметр не нужен для postgresql 9.2 — с ним сервер не запустится); 
- **hot_standby** — определяет, можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления; 
- **hot_standby_feedback** — определяет, будет или нет сервер slave сообщать мастеру о запросах, которые он выполняет.


```bash
--server3
pg_hba.conf
host    replication     all             all           scram-sha-256

--server4
postgres.conf
hot_standby = on


--server4
rm -rf /var/lib/postgresql/data/
su - postgres -c "pg_basebackup --host=db3 --username=otus --pgdata=/var/lib/postgresql/data --wal-method=stream --write-recovery-conf"

\dt
        List of relations
 Schema |  Name   | Type  | Owner 
--------+---------+-------+-------
 public | persons | table | otus
 public | test1   | table | otus
 public | test2   | table | otus
(3 rows)

select * from test1;
 id |    fio     
----+------------
  1 | e22583e70b
  2 | 38f0d302f7
  3 | 59390e01c8
  4 | 44ab79ac15
  5 | 731e577811
  6 | 5e0249a22e
  7 | d27ccdfb19
  8 | 5bfc7b4864
  9 | 80855a90de
 10 | 8f46c5502b
 12 | srv1_12   
(11 rows)

```

> Проверяем физическую репликацию. Добавим записи в server1.test1, server2.test2

```sql
--server1
insert into test1 (id, fio) values(15, 'srv1_15');

--server2
select * from test1;
 id |    fio     
... 
 15 | srv1_15   
(12 rows)

--server3
select * from test1;
 id |    fio     
... 
 15 | srv1_15   
(12 rows)

--server4
 select * from test1;
 id |    fio     
----+------------
  1 | e22583e70b
  2 | 38f0d302f7
  3 | 59390e01c8
  4 | 44ab79ac15
  5 | 731e577811
  6 | 5e0249a22e
  7 | d27ccdfb19
  8 | 5bfc7b4864
  9 | 80855a90de
 10 | 8f46c5502b
 12 | srv1_12   
 15 | srv1_15   
(12 rows)

```

##### Сложности
- Задание написано сумбурно. Пока в телеграме по факту что сделать не написали, я вообще не понял что делать
- В докере не сразу физика сработала. Там через volumes смонтирован был конфигурационный файл и не давал все удалить. Но в итоге победил. Команду подсмотрел в интернете