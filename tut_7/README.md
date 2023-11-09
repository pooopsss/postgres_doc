## Работа с журналами #6


```sql
--  выполнение контрольной точки раз в 30 секунд.
alter system set checkpoint_timeout = '30s';
select pg_reload_conf();

show checkpoint_timeout;
 checkpoint_timeout 
--------------------
 30s
(1 строка)
```



```sql
-- $PGDATA/pg_wal/
-- Размер текущего wal
select * from pg_ls_waldir();
           name           |   size   |      modification      
--------------------------+----------+------------------------
 000000010000000000000001 | 16777216 | 2023-11-09 14:48:04+00
(1 строка)

--  10 минут c помощью утилиты pgbench подавайте нагрузку.
pgbench -c8 -P 6 -T 600 -U postgres postgres -p 5432

caling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 213388
number of failed transactions: 0 (0.000%)
latency average = 22.493 ms
latency stddev = 35.436 ms
initial connection time = 22.108 ms
tps = 355.642058 (without initial connection time)


select * from pg_ls_waldir();
           name           |   size   |      modification      
--------------------------+----------+------------------------
 00000001000000000000001A | 16777216 | 2023-11-09 15:34:52+00
 00000001000000000000001D | 16777216 | 2023-11-09 15:33:29+00
 00000001000000000000001B | 16777216 | 2023-11-09 15:32:33+00
 00000001000000000000001C | 16777216 | 2023-11-09 15:33:01+00

```

> Можно посчитать самому по этим данным, но есть способ приятнее если ранее получим позицию в журнале до и после. 

```sql
-- Сбросим статистику 
-- SELECT pg_stat_reset_shared('bgwriter');
-- Повторим но с сохранением позицию в журнале
SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/1A289AE8
(1 строка)


-- Запускаем pgbench снова
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 222586
number of failed transactions: 0 (0.000%)
latency average = 21.563 ms
latency stddev = 29.483 ms
initial connection time = 21.465 ms
tps = 370.974517 (without initial connection time)

select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/31D4E190
(1 строка)

```

> Теперь Посмотрим размер до и после зная lsn указатели

```sql
select pg_size_pretty('0/31D4E190'::pg_lsn - '0/1A289AE8'::pg_lsn);
 pg_size_pretty 
----------------
 379 MB
(1 строка)

```

```sql
SELECT checkpoints_timed, checkpoints_req FROM pg_stat_bgwriter;
 checkpoints_timed | checkpoints_req 
-------------------+-----------------
                97 |               3
(1 строка)

```

> checkpoints_timed - По расписанию сколько выполнено
checkpoints_req - по требованию (max_wal_size)
Видим что впринцепе большенство создалось по расписанию, что говорит о более менне правильной настройке. 

```sql
show max_wal_size;
 max_wal_size 
--------------
 1GB
(1 строка)
```
> checkpoints_req значит сработал не по переполнению буфера, значит смотреть нужно в сторону  checkpoint_completion_target который позволяет оттянуть запись и сделать меннее агресивное сохранение

``` sql
show checkpoint_completion_target;
 checkpoint_completion_target 
------------------------------
 0.9
(1 строка)

```


> Для эксперимента возьмем и обратно поставим checkpoint_timeout = 5min и повторим статистику

```sql
-- сброс настроек
SELECT pg_stat_reset_shared('bgwriter');
alter system set checkpoint_timeout = '5min';
select pg_reload_conf();

SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/31D51D18
(1 строка)


--pgbanch

SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/3B09ABB0
(1 строка)

select pg_size_pretty('0/3B09ABB0'::pg_lsn - '0/31D51D18'::pg_lsn);
 pg_size_pretty 
----------------
 147 MB
(1 строка)

SELECT checkpoints_timed, checkpoints_req FROM pg_stat_bgwriter;
 checkpoints_timed | checkpoints_req 
-------------------+-----------------
                 4 |               0
(1 строка)

--Полностью по расписанию

--Возвращаем
alter system set checkpoint_timeout = '30s';
select pg_reload_conf();

```


#### Сравните tps в синхронном/асинхронном режиме

```sql

show fsync;
 fsync 
-------
 on
(1 строка)

show wal_sync_method;
 wal_sync_method 
-----------------
 fdatasync
(1 строка)

show synchronous_commit;
 synchronous_commit 
--------------------
 on
(1 строка)


--Установить асинхронный режим
alter system set fsync = 'off';

pg_ctlcluster 16 main restart

SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/3B09F118
(1 строка)

--pgbanch
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2325666
number of failed transactions: 0 (0.000%)
latency average = 2.058 ms
latency stddev = 0.930 ms
initial connection time = 17.971 ms
tps = 3876.119438 (without initial connection time)

```

> TPS увеличен за счет того что асинхронный режим. Скорость увеличилась, но за счет того что postgres не ждет записи wal и дает ответ клиенту что все хорошо. Данные могут потеяться в таком режиме


#### SRC32

```bash
pg_lscluster
16  main    5432 down   postgres /mnt/new_disk/16/main /var/log/postgresql/postgresql-16-main.log


sudo pg_ctlcluster 16 main stop
sudo /usr/lib/postgresql/16/bin/pg_checksums --enable -D /mnt/new_disk/16/main

sudo -i -u postgres psql
show data_checksums;
 data_checksums 
----------------
 on
(1 строка)


create table after_src(i integer);
insert into after_src (i) VALUES(2);
insert into after_src (i) VALUES(3);


# Смотрим где располагается таблица
select pg_relation_filepath('after_src');
 pg_relation_filepath 
----------------------
 base/5/16419
(1 строка)

#Вновим ерунду
echo "BAD FACE SRC" >> /mnt/new_disk/16/main/base/5/16419 
```


```sql
select * from after_src
;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 39421, а ожидалась - 25715
ОШИБКА:  неверная страница в блоке 0 отношения base/5/16419
```

> Логично. Мы внесли изменения в файл таблицы и суммы src32 не сошлись. Вот и ругается. Нужно или в файле посмотреть, ну или включить игнорирование сообщений

```sql
alter system set ignore_checksum_failure = 'on';
select pg_reload_conf();


```

> ignore_checksum_failure на 16 версии не сработал. и кластер перезапускал. Флаг стоит on. Вообщем исправлять можно вручную. Еще есть механизм xxd или pg_hexedit. Но туда не полез

