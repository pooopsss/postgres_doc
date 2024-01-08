### Работа с join'ами, статистикой #12

```sql
-- Создаем таблицы
create table bus (id serial,route text,id_model int,id_driver int);
create table model_bus (id serial,name text);
create table driver (id serial,first_name text,second_name text);
insert into bus values (1,'Москва-Болшево',1,1),(2,'Москва-Пушкино',1,2),(3,'Москва-Ярославль',2,3),(4,'Москва-Кострома',2,4),(5,'Москва-Волгорад',3,5),(6,'Москва-Иваново',null,null);
insert into model_bus values(1,'ПАЗ'),(2,'ЛИАЗ'),(3,'MAN'),(4,'МАЗ'),(5,'НЕФАЗ');
insert into driver values(1,'Иван','Иванов'),(2,'Петр','Петров'),(3,'Савелий','Сидоров'),(4,'Антон','Шторкин'),(5,'Олег','Зажигаев'),(6,'Аркадий','Паровозов');
```

```sql
select * from bus
postgres-# ;
 id |      route       | id_model | id_driver 
----+------------------+----------+-----------
  1 | Москва-Болшево   |        1 |         1
  2 | Москва-Пушкино   |        1 |         2
  3 | Москва-Ярославль |        2 |         3
  4 | Москва-Кострома  |        2 |         4
  5 | Москва-Волгорад  |        3 |         5
  6 | Москва-Иваново   |          |          
(6 строк)
```

```sql
select * from model_bus;
 id | name  
----+-------
  1 | ПАЗ
  2 | ЛИАЗ
  3 | MAN
  4 | МАЗ
  5 | НЕФАЗ
(5 строк)
```

```sql
select * from driver;
 id | first_name | second_name 
----+------------+-------------
  1 | Иван       | Иванов
  2 | Петр       | Петров
  3 | Савелий    | Сидоров
  4 | Антон      | Шторкин
  5 | Олег       | Зажигаев
  6 | Аркадий    | Паровозов
(6 строк)
```

### 1. Реализовать прямое соединение

```sql
select bus.route, model_bus.name from bus 
inner join model_bus on (model_bus.id = bus.id_model);
      route       | name 
------------------+------
 Москва-Болшево   | ПАЗ
 Москва-Пушкино   | ПАЗ
 Москва-Ярославль | ЛИАЗ
 Москва-Кострома  | ЛИАЗ
 Москва-Волгорад  | MAN
(5 строк)


select bus.route, model_bus.name, driver.first_name, driver.second_name 
from bus 
inner join model_bus on (model_bus.id = bus.id_model) 
inner join driver on (driver.id = bus.id_driver);
      route       | name | first_name | second_name 
------------------+------+------------+-------------
 Москва-Болшево   | ПАЗ  | Иван       | Иванов
 Москва-Пушкино   | ПАЗ  | Петр       | Петров
 Москва-Ярославль | ЛИАЗ | Савелий    | Сидоров
 Москва-Кострома  | ЛИАЗ | Антон      | Шторкин
 Москва-Волгорад  | MAN  | Олег       | Зажигаев
(5 строк)
```

> Прямое соединение отсекает все результаты в которых нет хотя бы одного совпадение с inner join

### 2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

```sql
elect bus.route, model_bus.name from bus 
left join model_bus on (model_bus.id = bus.id_model);
      route       | name 
------------------+------
 Москва-Болшево   | ПАЗ
 Москва-Пушкино   | ПАЗ
 Москва-Ярославль | ЛИАЗ
 Москва-Кострома  | ЛИАЗ
 Москва-Волгорад  | MAN
 Москва-Иваново   | 
(6 строк)


select bus.route, model_bus.name, driver.first_name, driver.second_name 
from bus 
left join model_bus on (model_bus.id = bus.id_model) 
left join driver on (driver.id = bus.id_driver);

      route       | name | first_name | second_name 
------------------+------+------------+-------------
 Москва-Болшево   | ПАЗ  | Иван       | Иванов
 Москва-Пушкино   | ПАЗ  | Петр       | Петров
 Москва-Ярославль | ЛИАЗ | Савелий    | Сидоров
 Москва-Кострома  | ЛИАЗ | Антон      | Шторкин
 Москва-Волгорад  | MAN  | Олег       | Зажигаев
 Москва-Иваново   |      |            | 
(6 строк)



select bus.route, model_bus.name, driver.first_name, driver.second_name 
from bus 
right join model_bus on (model_bus.id = bus.id_model) 
right join driver on (driver.id = bus.id_driver);

route       | name | first_name | second_name 
------------------+------+------------+-------------
 Москва-Болшево   | ПАЗ  | Иван       | Иванов
 Москва-Пушкино   | ПАЗ  | Петр       | Петров
 Москва-Ярославль | ЛИАЗ | Савелий    | Сидоров
 Москва-Кострома  | ЛИАЗ | Антон      | Шторкин
 Москва-Волгорад  | MAN  | Олег       | Зажигаев
                  |      | Аркадий    | Паровозов
(6 строк)

```

> Левостороннее или правостороннее соединение в отличае от join показывает все записи left table и есть или нет соединяет right table. Если right table нет, то выводятся вместо ее записей null

### 3. Реализовать кросс соединение двух или более таблиц

```sql
select *
from bus
cross join model_bus;

id |      route       | id_model | id_driver | id | name  
----+------------------+----------+-----------+----+-------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ
  3 | Москва-Ярославль |        2 |         3 |  1 | ПАЗ
  4 | Москва-Кострома  |        2 |         4 |  1 | ПАЗ
  5 | Москва-Волгорад  |        3 |         5 |  1 | ПАЗ
  6 | Москва-Иваново   |          |           |  1 | ПАЗ
  1 | Москва-Болшево   |        1 |         1 |  2 | ЛИАЗ
  2 | Москва-Пушкино   |        1 |         2 |  2 | ЛИАЗ
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ
  5 | Москва-Волгорад  |        3 |         5 |  2 | ЛИАЗ
  6 | Москва-Иваново   |          |           |  2 | ЛИАЗ
  1 | Москва-Болшево   |        1 |         1 |  3 | MAN
  2 | Москва-Пушкино   |        1 |         2 |  3 | MAN
  3 | Москва-Ярославль |        2 |         3 |  3 | MAN
  4 | Москва-Кострома  |        2 |         4 |  3 | MAN
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN
  6 | Москва-Иваново   |          |           |  3 | MAN
  1 | Москва-Болшево   |        1 |         1 |  4 | МАЗ
  2 | Москва-Пушкино   |        1 |         2 |  4 | МАЗ
  3 | Москва-Ярославль |        2 |         3 |  4 | МАЗ
  4 | Москва-Кострома  |        2 |         4 |  4 | МАЗ
  5 | Москва-Волгорад  |        3 |         5 |  4 | МАЗ
  6 | Москва-Иваново   |          |           |  4 | МАЗ
  1 | Москва-Болшево   |        1 |         1 |  5 | НЕФАЗ
  2 | Москва-Пушкино   |        1 |         2 |  5 | НЕФАЗ
  3 | Москва-Ярославль |        2 |         3 |  5 | НЕФАЗ
  4 | Москва-Кострома  |        2 |         4 |  5 | НЕФАЗ
  5 | Москва-Волгорад  |        3 |         5 |  5 | НЕФАЗ
  6 | Москва-Иваново   |          |           |  5 | НЕФАЗ
(30 строк)

```

> Cross join обьединяет многое ко многим. Каждую запись A (6 записей) с каждой записью B (5 записей) = 30 записей.


### 6. Реализовать полное соединение двух или более таблиц

```sql
select *
from bus  
full join model_bus on bus.id_model=model_bus.id;
 id |      route       | id_model | id_driver | id | name  
----+------------------+----------+-----------+----+-------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN
  6 | Москва-Иваново   |          |           |    | 
    |                  |          |           |  4 | МАЗ
    |                  |          |           |  5 | НЕФАЗ
(8 строк)



select *
from bus
full join model_bus on (bus.id_model=model_bus.id) full join driver on (driver.id=bus.id_driver);
 id |      route       | id_model | id_driver | id | name  | id | first_name | second_name 
----+------------------+----------+-----------+----+-------+----+------------+-------------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ   |  1 | Иван       | Иванов
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ   |  2 | Петр       | Петров
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ  |  3 | Савелий    | Сидоров
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ  |  4 | Антон      | Шторкин
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN   |  5 | Олег       | Зажигаев
  6 | Москва-Иваново   |          |           |    |       |    |            | 
    |                  |          |           |  4 | МАЗ   |    |            | 
    |                  |          |           |  5 | НЕФАЗ |    |            | 
    |                  |          |           |    |       |  6 | Аркадий    | Паровозов
(9 строк)

```

> Full join это полное соединение всех таблиц даже если нет записей. Одновременно как right + left join

### 5. Реализовать запрос, в котором будут использованы разные типы соединений

```sql

select bus.route, model_bus.name, driver.first_name, driver.second_name 
from bus 
left join model_bus on (model_bus.id = bus.id_model) 
inner join driver on (driver.id = bus.id_driver);
      route       | name | first_name | second_name 
------------------+------+------------+-------------
 Москва-Болшево   | ПАЗ  | Иван       | Иванов
 Москва-Пушкино   | ПАЗ  | Петр       | Петров
 Москва-Ярославль | ЛИАЗ | Савелий    | Сидоров
 Москва-Кострома  | ЛИАЗ | Антон      | Шторкин
 Москва-Волгорад  | MAN  | Олег       | Зажигаев
(5 строк)


select *
from bus
full join model_bus on (bus.id_model=model_bus.id) inner join driver on (driver.id=bus.id_driver);
 id |      route       | id_model | id_driver | id | name | id | first_name | second_name 
----+------------------+----------+-----------+----+------+----+------------+-------------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ  |  1 | Иван       | Иванов
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ  |  2 | Петр       | Петров
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ |  3 | Савелий    | Сидоров
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ |  4 | Антон      | Шторкин
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN  |  5 | Олег       | Зажигаев
(5 строк)

```


### ****

```sql
-- Сравним размер индексов (для сравнения выводится и размер таблицы):
select indexname,
    pg_size_pretty(pg_total_relation_size(indexname::regclass))
from pg_indexes
where tablename = 'bus'
union all
select 'bus', pg_size_pretty(pg_table_size('bus'::regclass));

 indexname | pg_size_pretty 
-----------+----------------
 bus       | 16 kB
(1 строка)



-- Добавим индексы
CREATE INDEX "idx_bus_model" ON "bus" USING btree( "id_model" Asc NULLS Last );
analyze bus;
select indexname,                                                                  
    pg_size_pretty(pg_total_relation_size(indexname::regclass))
from pg_indexes
where tablename = 'bus'
union all
select 'bus', pg_size_pretty(pg_table_size('bus'::regclass));
   indexname   | pg_size_pretty 
---------------+----------------
 idx_bus_model | 16 kB
 bus           | 16 kB
(2 строки)


```

