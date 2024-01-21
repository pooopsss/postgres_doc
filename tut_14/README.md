### Секционирование таблицы 13

```bash
wget https://edu.postgrespro.ru/demo_big.zip
unzip demo_big.zip
psql < demo_big.sql
```

```sql
\connect demo
\dt bookings.*
                Список отношений
  Схема   |       Имя       |   Тип   | Владелец 
----------+-----------------+---------+----------
 bookings | aircrafts       | таблица | postgres
 bookings | airports        | таблица | postgres
 bookings | boarding_passes | таблица | postgres
 bookings | bookings        | таблица | postgres
 bookings | flights         | таблица | postgres
 bookings | seats           | таблица | postgres
 bookings | ticket_flights  | таблица | postgres
 bookings | tickets         | таблица | postgres
(8 строк)


select pg_size_pretty(pg_table_size('bookings.ticket_flights'));
 pg_size_pretty 
----------------
 547 MB
(1 строка)

select pg_size_pretty(pg_table_size('bookings.flights'));
 pg_size_pretty 
----------------
 21 MB
(1 строка)

show enable_partition_pruning;
 enable_partition_pruning 
--------------------------
 on
(1 строка)
```


### Секционирование по списку
```sql
select * from bookings.flights limit 2 \gx
-[ RECORD 1 ]-------+-----------------------
flight_id           | 1
flight_no           | PG0403
scheduled_departure | 2016-08-11 07:25:00+00
scheduled_arrival   | 2016-08-11 08:20:00+00
departure_airport   | DME
arrival_airport     | LED
status              | Arrived
aircraft_code       | 321
actual_departure    | 2016-08-11 07:29:00+00
actual_arrival      | 2016-08-11 08:24:00+00
-[ RECORD 2 ]-------+-----------------------
flight_id           | 2
flight_no           | PG0404
scheduled_departure | 2016-08-11 15:05:00+00
scheduled_arrival   | 2016-08-11 16:00:00+00
departure_airport   | DME
arrival_airport     | LED
status              | Arrived
aircraft_code       | 321
actual_departure    | 2016-08-11 15:11:00+00
конкретной

--Возьмем поле статус
select status from bookings.flights group by status; 
  status   
-----------
 Arrived
 Cancelled
 Delayed
 Departed
 On Time
 Scheduled
(6 строк)

-- Создаем секционированную таблицу унаследованную от flights
create table bookings.flights_s1 (like bookings.flights including all) inherits (bookings.flights); 
ЗАМЕЧАНИЕ:  слияние столбца "flight_id" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "flight_no" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "scheduled_departure" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "scheduled_arrival" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "departure_airport" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "arrival_airport" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "status" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "aircraft_code" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "actual_departure" с наследованным определением
ЗАМЕЧАНИЕ:  слияние столбца "actual_arrival" с наследованным определением
ЗАМЕЧАНИЕ:  слияние ограничения "flights_check" с унаследованным определением
ЗАМЕЧАНИЕ:  слияние ограничения "flights_check1" с унаследованным определением
ЗАМЕЧАНИЕ:  слияние ограничения "flights_status_check" с унаследованным определением
CREATE TABLE

--Если запросить к первой таблице то искать будет в обеих
explain analyze select * from bookings.flights limit 3;
                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..0.08 rows=3 width=63) (actual time=1.282..1.285 rows=3 loops=1)
   ->  Append  (cost=0.00..5911.16 rows=215277 width=63) (actual time=1.280..1.282 rows=3 loops=1)
         ->  Seq Scan on flights flights_1  (cost=0.00..4820.67 rows=214867 width=63) (actual time=1.279..1.280 rows=3 loops=1)
         ->  Seq Scan on flights_s1 flights_2  (cost=0.00..14.10 rows=410 width=170) (never executed)
 Planning Time: 3.114 ms
 Execution Time: 1.311 ms
(6 строк)

--Но и можем без партиции взять
explain analyze select * from only bookings.flights limit 3;
                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..0.07 rows=3 width=63) (actual time=0.015..0.015 rows=3 loops=1)
   ->  Seq Scan on flights  (cost=0.00..4820.67 rows=214867 width=63) (actual time=0.014..0.014 rows=3 loops=1)
 Planning Time: 0.065 ms
 Execution Time: 0.025 ms
(4 строки)


--Добавим секционирование по статусам. Добавим также default что бы туда попадали все остальные
create table bookings.flights_list (like bookings.flights) partition by list(status);

CREATE TABLE bookings.flights_list_arrived PARTITION OF bookings.flights_list FOR VALUES IN ('Arrived');

CREATE TABLE bookings.flights_list_cancelled PARTITION OF bookings.flights_list FOR VALUES IN ('Cancelled');


insert into bookings.flights_list select * from bookings.flights where status in ('Arrived', 'Cancelled');


-- Все секции видит
 explain analyze select * from bookings.flights_list;
                                                                    QUERY PLAN                                                                    
--------------------------------------------------------------------------------------------------------------------------------------------------
 Append  (cost=0.00..5438.01 rows=198867 width=63) (actual time=0.015..54.125 rows=198867 loops=1)
   ->  Seq Scan on flights_list_arrived flights_list_1  (cost=0.00..4434.30 rows=198430 width=63) (actual time=0.015..27.202 rows=198430 loops=1)
   ->  Seq Scan on flights_list_cancelled flights_list_2  (cost=0.00..9.37 rows=437 width=65) (actual time=0.010..0.093 rows=437 loops=1)
 Planning Time: 0.360 ms
 Execution Time: 67.978 ms
(5 строк)
--Выбираем по конкретному статусу
explain analyze select * from bookings.flights_list where status='Arrived';
                                                                QUERY PLAN                                                                
------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on flights_list_arrived flights_list  (cost=0.00..4930.38 rows=198430 width=63) (actual time=0.012..58.575 rows=198430 loops=1)
   Filter: ((status)::text = 'Arrived'::text)
 Planning Time: 0.112 ms
 Execution Time: 73.056 ms
(4 строки)

```
> list по двум статусам сделали. При поиске видет как если бы по одной секции конкретной так и по всем


### Секционирование по дате

```sql
create table bookings.flights_range (like bookings.flights) partition by range(actual_arrival);


-- Data берется как 2016-02-01 >= and < 2016-03-01
create table bookings.flights_range_06_02_2016 PARTITION OF bookings.flights_range FOR VALUES FROM ('2016-02-01') TO ('2016-03-01');

insert into bookings.flights_range select * from bookings.flights where actual_arrival between '2016-02-01' and '2016-02-29';


explain analyze select * from bookings.flights_range;
                                                                QUERY PLAN                                                                 
-------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on flights_range_06_02_2016 flights_range  (cost=0.00..339.92 rows=15192 width=63) (actual time=0.004..0.965 rows=15192 loops=1)
 Planning Time: 0.227 ms
 Execution Time: 1.579 ms
(3 строки)

```

> Недостатки первых двух только в одном, нужно следить за тем что бы секция была на нужный диапазон ну или элемент списка. (cron, bash)


### Секционирование ппо хешу
- Когда необходимо равномерное распределение;
- Нет явного ключа, по которому можно разбить таблицу;
- Для равномерного распределения необходимо уникалþное или почти уникалþное поле.

```sql
create table bookings.flights_hash (like bookings.flights) partition by hash(flight_id);

-- делаем три части
create table bookings.flights_hash_p1
partition of bookings.flights_hash
for values with (modulus 3, remainder 0);
 
create table bookings.flights_hash_p2
partition of bookings.flights_hash
for values with (modulus 3, remainder 1);

create table bookings.flights_hash_p3
partition of bookings.flights_hash
for values with (modulus 3, remainder 2);

insert into bookings.flights_hash select * from bookings.flights;



select count(*) from bookings.flights_hash_p1;
 count 
-------
 71904
(1 строка)

select count(*) from bookings.flights_hash_p2;
 count 
-------
 71494
(1 строка)


select count(*) from bookings.flights_hash_p3;
 count 
-------
 71469
(1 строка)



explain analyze select * from bookings.flights_hash limit 3;
                                                                 QUERY PLAN                                                                 
--------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..0.08 rows=3 width=63) (actual time=0.007..0.009 rows=3 loops=1)
   ->  Append  (cost=0.00..5848.01 rows=214867 width=63) (actual time=0.006..0.007 rows=3 loops=1)
         ->  Seq Scan on flights_hash_p1 flights_hash_1  (cost=0.00..1597.04 rows=71904 width=63) (actual time=0.005..0.006 rows=3 loops=1)
         ->  Seq Scan on flights_hash_p2 flights_hash_2  (cost=0.00..1588.94 rows=71494 width=63) (never executed)
         ->  Seq Scan on flights_hash_p3 flights_hash_3  (cost=0.00..1587.69 rows=71469 width=63) (never executed)
 Planning Time: 0.447 ms
 Execution Time: 0.038 ms
(7 строк)


```




### P.S. Для себя
Декларативнýй способ:
- Можно создаватþ дефолтную секцию:
○ create table part_name partition of main_table default;
- Можно исполþзоватþ минималþное, максималþное значение для range partition
○ create table part_name partition of main_table for values from (MINVALUE) to (MAXVALUE);
- Можно отсоединять секции:
○ alter table main_table detach partition part_name;
- Можно добавлāть секции:
○ alter table main_table attach partition part_name for values from (‘2020-01-01’) to
(‘2020-02-01’);
- Не забыть включить enable_partition_pruning длā оптимизации (default on).

Трудности:
- В partition by range не получится использовать в ключе секционирования null значения;
- Не умеет создавать секции самостоāтельно (можно использовать cron и прочее);
- Не получится создать уникальное ограничение на часть.