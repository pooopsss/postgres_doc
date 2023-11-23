### Бэкапы #9

```sql
-- Создаем таблицу и данные
create database base_for_backup;
\c base_for_backup

create schema bk_schema

CREATE TABLE bk_schema.b_table(
  id serial,
  fio char(100)
);

INSERT INTO bk_schema.b_table(fio) SELECT vl from generate_series(1,100), gen_random_uuid() as vl;

```


```sql
-- Создаем таблицу и данные
create database base_for_backup;
\c base_for_backup

create schema bk_schema

CREATE TABLE bk_schema.b_table(
  id serial,
  fio char(100)
);

INSERT INTO bk_schema.b_table(fio) SELECT vl from generate_series(1,100), gen_random_uuid() as vl;

```

```bash
postgres@pooopsssvd1:~$ mkdir /var/lib/postgresql/backup_folder
```
#### Сделаем логический бэкап используя утилиту COPY
```sql
\copy bk_schema.b_table to '/var/lib/postgresql/backup_folder/b_table.sql';
```

#### Восстановим в 2 таблицу данные из бэкапа
```sql
CREATE TABLE bk_schema.b_table2(
  id serial,
  fio char(100)
);

\copy bk_schema.b_table2 from /var/lib/postgresql/backup_folder/b_table.sql
```

#### PG_DUMP сохранем две таблицы в дамп
```bash
pg_dump -d base_for_backup -t bk_schema.b_table2 -t bk_schema.b_table -Fc -f dump_backup.sql

```

#### PG_RESOTRE восстанавливаем в другую бд и только вторую таблицу
```sql
create database base2;
\c base2
create schema bk_schema;
```
```bash
pg_restore --dbname base2 --table=b_table2 dump_backup.sql
```
```sql
\dt bk_schema.*
             Список отношений
   Схема   |   Имя    |   Тип   | Владелец 
-----------+----------+---------+----------
 bk_schema | b_table2 | таблица | postgres
(1 строка)

```


### P.S. 
> Все ожидаю когда обьяснят бекап Diff, Inc и restore к ним. ОЧЕНЬ ОЧЕНЬ ОЧЕНЬ

