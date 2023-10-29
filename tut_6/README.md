## Настройка autovacuum с учетом особеностей производительности #5


```sql
CREATE TABLE student(
  id serial,
  fio char(100)
);

INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,500000);
```

```bash
postgres@pooopsssvd1:~$  pgbench -i postgres -p 5434
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.08 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.27 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.10 s, vacuum 0.05 s, primary keys 0.10 s).

```


```bash
pgbench -c8 -P 6 -T 60 -U postgres postgres -p 5434

pgbench (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 405.3 tps, lat 19.627 ms stddev 10.424, 0 failed
progress: 12.0 s, 401.5 tps, lat 19.930 ms stddev 10.251, 0 failed
progress: 18.0 s, 393.5 tps, lat 20.330 ms stddev 10.127, 0 failed
progress: 24.0 s, 436.2 tps, lat 18.343 ms stddev 8.023, 0 failed
progress: 30.0 s, 397.0 tps, lat 20.145 ms stddev 9.608, 0 failed
progress: 36.0 s, 414.2 tps, lat 19.318 ms stddev 9.801, 0 failed
progress: 42.0 s, 375.0 tps, lat 21.312 ms stddev 11.023, 0 failed
progress: 48.0 s, 382.5 tps, lat 20.936 ms stddev 10.664, 0 failed
progress: 54.0 s, 397.7 tps, lat 20.105 ms stddev 10.144, 0 failed
progress: 60.0 s, 376.8 tps, lat 21.220 ms stddev 9.556, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 23886
number of failed transactions: 0 (0.000%)
latency average = 20.093 ms
latency stddev = 10.008 ms
initial connection time = 19.546 ms
tps = 398.049726 (without initial connection time)

```



```bash
sudo nano /etc/postgresql/15/main/postgresql.conf

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB


sudo pg_ctlcluster 15 main restart

pgbench -c8 -P 6 -T 60 -U postgres postgres -p 5434

progress: 6.0 s, 383.8 tps, lat 20.736 ms stddev 10.538, 0 failed
progress: 12.0 s, 383.3 tps, lat 20.880 ms stddev 10.399, 0 failed
progress: 18.0 s, 401.7 tps, lat 19.926 ms stddev 10.535, 0 failed
progress: 24.0 s, 398.0 tps, lat 20.087 ms stddev 10.469, 0 failed
progress: 30.0 s, 385.2 tps, lat 20.774 ms stddev 9.968, 0 failed
progress: 36.0 s, 391.3 tps, lat 20.435 ms stddev 10.854, 0 failed
progress: 42.0 s, 397.8 tps, lat 20.121 ms stddev 10.311, 0 failed
progress: 48.0 s, 419.8 tps, lat 19.050 ms stddev 9.290, 0 failed
progress: 54.0 s, 392.3 tps, lat 20.386 ms stddev 11.034, 0 failed
progress: 60.0 s, 386.0 tps, lat 20.724 ms stddev 11.056, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 23644
number of failed transactions: 0 (0.000%)
latency average = 20.300 ms
latency stddev = 10.464 ms
initial connection time = 14.217 ms
tps = 394.000475 (without initial connection time)

```

> Изменился initial connection time = 14.217 ms

> По параметрам изменили shared_buffers и логично растянули max_wal_size чтобы растянуть процесс записи большого объёма на более продолжительное время

> Большую часть параметров мы не изучали, но посмотрев по интернету вроде ничего в глаза не попало раздражающего. 


> Идем дальше. Создаем таблицу и вставляем илион записей

```sql
CREATE TABLE b_table(
  fio text
);

INSERT INTO b_table(fio) SELECT vl from generate_series(1,1000000), gen_random_uuid() as vl;
```

```sql
#Размер
SELECT pg_size_pretty(pg_total_relation_size('b_table'));
 pg_size_pretty 
----------------
 65 MB
(1 строка)


#5 Обновлеем и добавлем значение к строчки
update b_table set fio = fio || random();
update b_table set fio = fio || random();
update b_table set fio = fio || random();
update b_table set fio = fio || random();
update b_table set fio = fio || random();

SELECT pg_size_pretty(pg_total_relation_size('b_table'));
 pg_size_pretty 
----------------
 595 MB
(1 строка)

```

```sql
#Добавляем extensions
CREATE EXTENSION pageinspect;
CREATE EXTENSION pgstattuple;

#См0трим мертвые tuple
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'b_table';

 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 b_table |    1511015 |          0 |      0 | 2023-10-29 19:43:01.714034+00
(1 строка)

#Уже автовакум вычестил все

#Тут тоже впорядке все
SELECT * FROM pgstattuple('b_table') \gx
-[ RECORD 1 ]------+----------
table_len          | 623984640
tuple_count        | 1000000
tuple_len          | 154496800
tuple_percent      | 24.76
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 459888580
free_percent       | 73.7

# Смотрим когда последний раз autovacuum приходил
SELECT schemaname, relname, last_autovacuum
FROM pg_stat_user_tables
WHERE schemaname = 'public' AND relname = 'b_table';

schemaname | relname |        last_autovacuum        
------------+---------+-------------------------------
 public     | b_table | 2023-10-29 19:43:01.714034+00
(1 строка)



# Еще 5 раз обновлем
 update b_table set fio = fio || '_' || (random() * 9 + 1);
...

SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'b_table';

 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 b_table |    1511015 |    1000000 |     66 | 2023-10-29 19:43:01.714034+00
(1 строка)


#n_dead_tup 1 млн
postgres=# SELECT pg_size_pretty(pg_total_relation_size('b_table'));
 pg_size_pretty 
----------------
 595 MB
(1 строка)

```

> Размер такой же - не успели - автовакуум сработал. Отключаем автовакуум

```sql
ALTER TABLE b_table SET (autovacuum_enabled = off);

#10 не получилость - на 7 вставкe сервер упал - закончилось место.
update b_table set fio = fio || '_' || (random() * 9 + 1);
...{7}

SELECT pg_size_pretty(pg_total_relation_size('b_table'));
 pg_size_pretty 
----------------
 1634 MB
(1 строка)

```

> Ну все просто. Автовакум на таблице выключен. Он перестал чистить старые записи и размер пропорционально рос. Все транзакции сохраняются. Во всех транзакциях данные помечаются флагами как например удаление. В итоге наростился dead_tuples. Это устаревшие версии строк в результате установки транзакций xmin, xmaх. Когда xmax меньше xmin. То такие помечаются устаревшими.

При включении Autovacuum обратно система автоматически начнет все чистить

```sql
ALTER TABLE b_table SET (autovacuum_enabled = on);
# При включении ничего не произошло автоматически. 
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'b_table';
 relname | n_live_tup | n_dead_tup | ratio% | last_autovacuum 
---------+------------+------------+--------+-----------------
 b_table |          0 |          0 |      0 | 
(1 строка)

#Запускаем VACUUM
vacuum;
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'b_table';
 relname | n_live_tup | n_dead_tup | ratio% | last_autovacuum 
---------+------------+------------+--------+-----------------
 b_table |    1000000 |          0 |      0 | 
(1 строка)



SELECT * FROM pgstattuple_approx('b_table') \gx
-[ RECORD 1 ]--------+------------------
table_len            | 1667743744
scanned_percent      | 0
approx_tuple_count   | 1000000
approx_tuple_len     | 297491552
approx_tuple_percent | 17.83796539907752
dead_tuple_count     | 0
dead_tuple_len       | 0
dead_tuple_percent   | 0
approx_free_space    | 1370252192
approx_free_percent  | 82.16203460092248
```


### Задание со *

```bash
#Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
#Не забыть вывести номер шага цикла.
DO $$
BEGIN
    FOR r IN 1..10
    LOOP
        RAISE info 'update %', r;
        update b_table set fio = fio || '_' || (random() * 9 + 1);        
    END LOOP;
END$$;
```