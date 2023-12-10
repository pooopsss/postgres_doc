### Виды индексов. Работа с индексами и оптимизация запросов #11


```sql
create database test11;
\c test11;

--Создаем таблицу
create table test as 
select generate_series as id
	, generate_series::text || (random() * 10)::text as col2 
    , (array['Yes', 'No', 'Maybe'])[floor(random() * 3 + 1)] as is_okay
from generate_series(1, 50000);

-- Создаем индекс
create index idx_test_id on test(id);
```

```sql
explain select  * from test;
                        QUERY PLAN                         
-----------------------------------------------------------
 Seq Scan on test  (cost=0.00..884.00 rows=50000 width=31)
(1 строка)

```

> В данном примере использовался seq scan без индекса. Для некоторых случаев postgres сам в некоторых ситуациях может принимать, а может и нет индекс. Для небольших таблиц postgres часто индексы и не включает.В данном пример не использовалось индексированное поле id в where или order

```sql
test11=# explain select  * from test order by id desc;
                                       QUERY PLAN                                        
-----------------------------------------------------------------------------------------
 Index Scan Backward using idx_test_id on test  (cost=0.29..1693.29 rows=50000 width=31)
(1 строка)
```

> Вот и получили аналитику на idx_test_id который ранее создавали, из-за **order by id**


```sql
-- Примеры получения списков индекса из урока
--Размер таблиц вместе с индексами
SELECT
    TABLE_NAME,
    pg_size_pretty(table_size) AS table_size,
    pg_size_pretty(indexes_size) AS indexes_size,
    pg_size_pretty(total_size) AS total_size
FROM (
    SELECT
        TABLE_NAME,
        pg_table_size(TABLE_NAME) AS table_size,
        pg_indexes_size(TABLE_NAME) AS indexes_size,
        pg_total_relation_size(TABLE_NAME) AS total_size
    FROM (
        SELECT ('"' || table_schema || '"."' || TABLE_NAME || '"') AS TABLE_NAME
        FROM information_schema.tables
    ) AS all_tables
    ORDER BY total_size DESC

    ) AS pretty_sizes;


---see defenition index
SELECT
    tablename,
    indexname,
    indexdef
FROM
    pg_indexes
WHERE
    schemaname = 'public'
ORDER BY
    tablename,
    indexname;

 tablename |  indexname  |                         indexdef                         
-----------+-------------+----------------------------------------------------------
 test      | idx_test_id | CREATE INDEX idx_test_id ON public.test USING btree (id)
(1 строка)


--Неиспользуемые индексы
SELECT s.schemaname,
       s.relname AS tablename,
       s.indexrelname AS indexname,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size,
       s.idx_scan
FROM pg_catalog.pg_stat_user_indexes s
   JOIN pg_catalog.pg_index i ON s.indexrelid = i.indexrelid
WHERE s.idx_scan < 10      -- has never been scanned
  AND 0 <>ALL (i.indkey)  -- no index column is an expression
  AND NOT i.indisunique   -- is not a UNIQUE index
  AND NOT EXISTS          -- does not enforce a constraint
         (SELECT 1 FROM pg_catalog.pg_constraint c
          WHERE c.conindid = s.indexrelid)
ORDER BY pg_relation_size(s.indexrelid) DESC;

 schemaname | tablename |  indexname  | index_size | idx_scan 
------------+-----------+-------------+------------+----------
 public     | test      | idx_test_id | 1112 kB    |        0
(1 строка)
```

```sql
-- Так можно не только размер таблицы, но и размер индекса получить
select pg_size_pretty(pg_table_size('idx_test_id')); 
```


#### Реализовать индекс для полнотекстового поиска

> Для ускорения полнотекстового поиска можно использовать индексы двух видов: **GIN** и **GiST**

> Более предпочтительными для текстового поиска являются индексы GIN. Будучи инвертированными индексами, они содержат записи для всех отдельных слов (лексем) с компактным списком мест их вхождений

> Построение индекса **GIN** часто можно ускорить, увеличив **maintenance_work_mem**

```sql
-- таблицу берем с практики 
create table orders (
    id int,
    user_id int,
    order_date date,
    status text,
    some_text text
);

insert into orders(id, user_id, order_date, status, some_text)
select generate_series, (random() * 70), date'2019-01-01' + (random() * 300)::int as order_date
        , (array['returned', 'completed', 'placed', 'shipped'])[(random() * 4)::int]
        , concat_ws(' ', (array['go', 'space', 'sun', 'London'])[(random() * 5)::int]
            , (array['the', 'capital', 'of', 'Great', 'Britain'])[(random() * 6)::int]
            , (array['some', 'another', 'example', 'with', 'words'])[(random() * 6)::int]
            )
from generate_series(100001, 1000000);

create index idx_ord_id on orders(id);


explain
select *
from orders
where id < 100;
QUERY PLAN                                
--------------------------------------------------------------------------
 Index Scan using idx_ord_id on orders  (cost=0.42..4.44 rows=1 width=34)
   Index Cond: (id < 100)
(2 строки)



explain
select *
from orders
where id > 100;
                           QUERY PLAN                           
----------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..18461.00 rows=900000 width=34)
   Filter: (id > 100)
(2 строки)

```

> Для больше значения используется Filter: (id > 100). В то время как < используется Index Scan using idx_ord_id on orders


```sql
explain
select *
from orders
order by id;
                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Index Scan using idx_ord_id on orders  (cost=0.42..30598.42 rows=900000 width=34)
(1 строка)

```

> При order by используется индекс


```sql
explain select * from orders where some_text ilike 'g%';
----------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..18461.00 rows=213825 width=34)
   Filter: (some_text ~~* 'g%'::text)
(2 строки)
```

```sql
CREATE INDEX idx_some_text ON orders USING GIN (some_text);
ОШИБКА:  для типа данных text не определён класс операторов по умолчанию для метода доступа "gin"
ПОДСКАЗКА:  Вы должны указать класс операторов для индекса или определить класс операторов по умолчанию для этого типа данных.
```

> Можно конечно тип поля перевести в tsvector - set some_text_lexeme = to_tsvector(some_text);

```sql
alter table orders drop column if exists some_text_lexeme;
alter table orders add column some_text_lexeme tsvector;
update orders
set some_text_lexeme = to_tsvector(some_text);
explain
select some_text
from orders
where some_text_lexeme @@ to_tsquery('britains');


QUERY PLAN                                    
---------------------------------------------------------------------------------
 Gather  (cost=1000.00..131874.50 rows=148020 width=14)
   Workers Planned: 2
   ->  Parallel Seq Scan on orders  (cost=0.00..116072.50 rows=61675 width=14)
         Filter: (some_text_lexeme @@ to_tsquery('britains'::text))
 JIT:
   Functions: 4
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(7 строк)

```

```sql
create extension pg_trgm; 
CREATE INDEX idx_some_text ON orders USING GIN (some_text gin_trgm_ops);

explain select * from orders where some_text ilike 'g%';
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=1452.87..11336.68 rows=213825 width=34)
   Recheck Cond: (some_text ~~* 'g%'::text)
   ->  Bitmap Index Scan on idx_some_text  (cost=0.00..1399.41 rows=213825 width=0)
         Index Cond: (some_text ~~* 'g%'::text)
(4 строки)

select pg_size_pretty(pg_table_size('idx_some_text'));
 pg_size_pretty 
----------------
 14 MB
(1 строка)
```

> Использовали специальное расширения для Gin индекса по строке. Удалим этот индекс и создадим станадртный **btree**

```sql
drop index if exists idx_some_text;
create index idx_some_text on orders(some_text);

\d orders
                           Таблица "public.orders"
  Столбец   |   Тип   | Правило сортировки | Допустимость NULL | По умолчанию 
------------+---------+--------------------+-------------------+--------------
 id         | integer |                    |                   | 
 user_id    | integer |                    |                   | 
 order_date | date    |                    |                   | 
 status     | text    |                    |                   | 
 some_text  | text    |                    |                   | 
Индексы:
    "idx_ord_id" btree (id)
    "idx_some_text" btree (some_text)


```

> Видим что создался **btree**

```sql
explain select * from orders where some_text ilike 'g%';
                           QUERY PLAN                           
----------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..18461.00 rows=213825 width=34)
   Filter: (some_text ~~* 'g%'::text)
(2 строки)


select pg_size_pretty(pg_table_size('idx_some_text'));
 pg_size_pretty 
----------------
 6304 kB
(1 строка)

```

> Итого **btree** через анализатор по тексту не виден, даже если order by по этому полю сделать. Размер индекса маленький. Индекс gin имеет больший размер, но при этом выборка произошла быстрее.



```sql
-- частичный индекс status='returned'
CREATE INDEX idx_status_only_returned ON orders (status) where status='returned';


--индекс на несколько полей
create index idx_orders_order_date_and_status on orders(order_date) include (status);

```
> Индекс по нескольким поля работает как по отдельности, так и вместе по полям. order_date по нему сработает индекс и если только status - по нему тоже


#### Добавляем коменты к индексам

```sql
--list index
SELECT
    tablename,
    indexname,
    indexdef
FROM
    pg_indexes
WHERE
    schemaname = 'public'
ORDER BY
    tablename,
    indexname;

 orders    | idx_ord_id                       | CREATE INDEX idx_ord_id ON public.orders USING btree (id)
 orders    | idx_orders_order_date_and_status | CREATE INDEX idx_orders_order_date_and_status ON public.orders USING btree (order_date) INCLUDE (status)
 orders    | idx_some_text                    | CREATE INDEX idx_some_text ON public.orders USING btree (some_text)
 orders    | idx_status_only_returned         | CREATE INDEX idx_status_only_returned ON public.orders USING btree (status) WHERE (status = 'returned'::text)
 test      | idx_test_id                      | CREATE INDEX idx_test_id ON public.test USING btree (id)
(5 строк)

```


```sql
-- ADD COMMENT
COMMENT ON INDEX idx_ord_id IS 'Индекс на order.id';
COMMENT ON INDEX idx_orders_order_date_and_status IS 'Индекс на date , status два поля';
COMMENT ON INDEX idx_some_text IS 'Индекс на some_text';
COMMENT ON INDEX idx_status_only_returned IS 'Индекс частичный на status=returned';
COMMENT ON INDEX idx_test_id IS 'Индекс на test.id';

```


```sql
\di+ idx_ord_id
                                            Список отношений
 Схема  |    Имя     |  Тип   | Владелец | Таблица |  Хранение  | Метод доступа | Размер |   Описание   
--------+------------+--------+----------+---------+------------+---------------+--------+--------------
 public | idx_ord_id | индекс | postgres | orders  | постоянное | btree         | 39 MB  | Индекс на id
(1 строка)
```





#### Оставшиеся вопросы

1. select * from orders where id > 100. Для больше значения используется Filter: (id > 100). В то время как < используется Index Scan using idx_ord_id. Тут я не понял почему так происходит. Нам на уроки показывали операторы конечно поддерживаемые. Возможно в этом дело
2. На счет полнотекстового поиска - есть ли смысл вообще использовать btree character vareing, text например?
3. Пока не нашел способ через ьаблицы получить описание COMMENT. Хотя он есть точно
