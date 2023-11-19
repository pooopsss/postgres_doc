## Нагрузочное тестирование и тюнинг PostgreSQL #8

```sql
--смотрим текущие параметры
-- async ? если fsync=on -> no async
show fsync;
 fsync 
-------
 on

-- Совместно используемая память для кэша
show shared_buffers;
 shared_buffers 
----------------
 128MB
(1 строка)

-- размер одного work_mem влияющего на сортировку и обьединения под каждый запрос
show work_mem;
 work_mem 
----------
 4MB

--сбор статистики, сбор мусора, создание индекса
show maintenance_work_mem;
 maintenance_work_mem 
----------------------
 64MB
(1 строка)

-- маскимальный размер кеша файла
show effective_cache_size;
 effective_cache_size 
----------------------
 4GB
(1 строка)

-- Синхронная запись в лог файл
show synchronous_commit;
 synchronous_commit 
--------------------
 on
(1 строка)


-- Види что на сервере включены и checksums
show data_checksums;
 data_checksums 
----------------
 on
(1 строка)


```

> Сервер 2 ядра 4 Гб ОЗУ. Пробуем pgbench на текущих настроках

```bash
pgbench -c8 -P 6 -T 600 -U postgres postgres -p 5432

scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 210396
number of failed transactions: 0 (0.000%)
latency average = 22.812 ms
latency stddev = 30.741 ms
initial connection time = 26.230 ms
tps = 350.660622 (without initial connection time)
```

> Настроаиваем пострес на лучшую производительность

```bash
pg_ctlcluster 16 main stop
-- отключаем checksum
/usr/lib/postgresql/16/bin/pg_checksums --disable -D /mnt/new_disk/16/main
pg_ctlcluster 16 main start
```


```sql
-- асинхронный режим
alter system set fsync = 'off';
-- асинхронная запись комитов
alter system set synchronous_commit='off';
alter system set max_connections = 20;
alter system set shared_buffers = '1GB';
alter system set effective_cache_size = '3GB';
alter system set maintenance_work_mem = '256MB';
alter system set checkpoint_completion_target = '0.9';
alter system set wal_buffers = '16MB';
alter system set work_mem = '13107kB';
alter system set work_mem = '13107kB';


pg_ctlcluster 16 main restart
```


```bash
pgbench -c8 -P 6 -T 600 -U postgres postgres -p 5432

scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2351080
number of failed transactions: 0 (0.000%)
latency average = 2.035 ms
latency stddev = 0.873 ms
initial connection time = 25.911 ms
tps = 3918.527979 (without initial connection time)


number of transactions actually processed: 210396
number of failed transactions: 0 (0.000%)
latency average = 22.812 ms
latency stddev = 30.741 ms
initial connection time = 26.230 ms
tps = 350.660622 (without initial connection time)

```
> Мы видим что намного выросла скорость отдачи результата транзакций. Ну это понятно - ассинхронный режим не ждет фактического завершения

> average намного увеличелся. Т.О мы увеличили производительность во много раз, но забили на такие вещи как гарантия записи при сбое

> Для увеличения производительности можно еще создавать sharding например. Но мы его пока не изучали. 


### sysbench
```bash
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```

```sql
CREATE USER sbtest WITH PASSWORD '123456';
CREATE DATABASE sbtest;
GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;

show hba_file;
              hba_file               
-------------------------------------
 /etc/postgresql/16/main/pg_hba.conf

nano /etc/postgresql/16/main/pg_hba.conf
host    sbtest          sbtest          all         md5
pg_ctlcluster 16 main restart

--insert
sysbench --db-driver=pgsql --threads=2 --pgsql-port=5432 --pgsql-user=sbtest --pgsql-password=123456  --pgsql-db=sbtest  /usr/share/sysbench/bulk_insert.lua run


--OLTP будет интересно когда OLTP смотреть будем
--oltp check read write
sysbench --db-driver=pgsql --threads=2 --pgsql-port=5432 --pgsql-user=sbtest --pgsql-password=123456  --pgsql-db=sbtest  /usr/share/sysbench/oltp_read_write.lua run

```

> sysbench Все в скриптах. Кстати у меня parallel_prepare.lua не появился после установки
