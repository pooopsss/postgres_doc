### Триггеры, поддержка заполнения витрин 14

### Установка postgres (Debian 12)

``` bash
sudo apt udpate
sudo apt upgrade

sudo apt install postgresql

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt install postgresql-16
```

> Скачиваем и заливаем hw_triggers.sql

```sql
-- Заливаем дамп
psql < hw_triggers.sql

--Проверяем
\dt  pract_functions.*
                   Список отношений
      Схема      |      Имя      |   Тип   | Владелец
-----------------+---------------+---------+----------
 pract_functions | good_sum_mart | таблица | postgres
 pract_functions | goods         | таблица | postgres
 pract_functions | sales         | таблица | postgres
(3 строки)
```

```sql
-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM  pract_functions.goods G
INNER JOIN  pract_functions.sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

   good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 строки)



 select * from pract_functions.sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-01-30 21:56:02.690568+03 |        10
        2 |       1 | 2024-01-30 21:56:02.690568+03 |         1
        3 |       1 | 2024-01-30 21:56:02.690568+03 |       120
        4 |       2 | 2024-01-30 21:56:02.690568+03 |         1
(4 строки)
```

####Создадим тригерную функцию для установки значений  good_sum_mart

```sql
CREATE OR REPLACE FUNCTION  reset_mart_table() RETURNS TRIGGER
  LANGUAGE plpgsql AS
$reset_mart_table$
DECLARE
    currentCursor NO SCROLL CURSOR FOR SELECT G.good_name, sum(G.good_price * S.sales_qty) as good_calc
                            FROM  pract_functions.goods G
                            INNER JOIN  pract_functions.sales S ON S.good_id = G.goods_id
                            GROUP BY G.good_name;

    currentRec record;
    -- _a integer default 0;
BEGIN
--     IF _currentGood IS NULL THEN
--         RAISE EXCEPTION  'Товар с id:% не найден', good_id;
--     END IF;
    TRUNCATE TABLE good_sum_mart;
    for currentRec in currentCursor loop
        -- if _a = 1 then
        --     RAISE EXCEPTION  'Fake error - Проверка на transaction';
        --end if;
         INSERT INTO good_sum_mart (good_name, sum_sale) VALUES (currentRec.good_name, currentRec.good_calc);

        --  _a := _a+1;
    end loop;


    RETURN NULL;
    -- COMMIT;
-- EXCEPTION
--     WHEN OTHERS THEN
--         ROLLBACK;
--         RAISE EXCEPTION 
END
$reset_mart_table$;

```


### Добавление триггеров *Delete*, *Update* на таблицу *goods* (*insert* не ставлю потому что считаем по sales)
```sql
    
CREATE OR REPLACE TRIGGER update_goods_trigger
    AFTER UPDATE OR DELETE ON pract_functions.goods
    FOR EACH STATEMENT
    EXECUTE FUNCTION reset_mart_table();


select * from pract_functions.goods;
 goods_id |        good_name         |  good_price
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
(2 строки)
-- Изменям цену на один товар
update pract_functions.goods set good_price=0.70 where goods_id=1;


select * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        91.70
(2 строки)

-- Все верно 131x0.70 =  91.70
```

### Добавление триггеров *Delete*, *Update*, "Insert" на таблицу *sales* 
```sql
    
CREATE OR REPLACE TRIGGER update_goods_trigger
    AFTER UPDATE OR DELETE OR INSERT ON pract_functions.sales
    FOR EACH STATEMENT
    EXECUTE FUNCTION reset_mart_table();

select * from pract_functions.sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-01-30 21:56:02.690568+03 |        10
        2 |       1 | 2024-01-30 21:56:02.690568+03 |         1
        3 |       1 | 2024-01-30 21:56:02.690568+03 |       120
        4 |       2 | 2024-01-30 21:56:02.690568+03 |         1
(4 строки)
-- Добавим еще спичек
insert into pract_functions.sales (good_id, sales_qty) VALUES (1, 10);



select * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        98.70
(2 строки)

-- Все верно 141x0.70 =  98.70


-- Удалю эту строку
 delete from  pract_functions.sales where sales_id=5;
 select * from good_sum_mart;
  good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        91.70
(2 строки)

--Изменяем количество
update pract_functions.sales set sales_qty=2 where sales_id=2;
select * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        92.40
(2 строки)
-- Все верно 132x0.70 =  92.40
```


> P.S. 
- На продакшене если товаров много то создается функция обработки на каждый товар. Так можно приостановить и продолжить.
- Вместо good_sum_mart.good_name -> goods_id (Foreign key) и тогда на delete можно убрать триггер будет
 