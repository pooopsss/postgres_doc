## Работа с базами данных, пользователями и правами #4
 
```yaml
### Установка postgres
version: '3'
services:
  
  db:
    build:
      context: .docker/postgres
    container_name: postgres_otus_db
    expose:
      - ${DB_PORT}
    ports:
      - ${DB_PORT}:5432
    environment:
      POSTGRES_DB: ${DB_DATABASE}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    restart: unless-stopped
    volumes:
      - ./.docker/volumes/postgres/data:/var/lib/postgresql/data
```



```bash
### Вход в образ
docker exec -ti postgres_otus_db bash
psql -U otus
```
> otus как postgres пользователь - изначально проставлен в настройках вместо postgres. Поэтому его буду итспользовать вместо основного

```sql
create database testdb;
\c testdb otus
```

```sql
create schema testnm;
create table testnm.t1(c1 integer);
insert into testnm.t1(c1) values(1);
create role readonly;
#readonly право на подключение к базе данных testdb
grant connect on database testdb to readonly;
#право на использование схемы testnm
grant select on all tables in schema testnm to readonly;

```


```sql
#Создаем пользователя с паролем
create user testread with password 'test123';
# Даем роль readonly новому пользователю
grant readonly to testread;
```


```bash

#Входим
\c testdb testread
or
docker exec -ti postgres_otus_db psql -U testread -d testdb
```

```sql
testdb=>
select * from t1;
#ERROR:  permission denied for schema testnm
#LINE 1: select * from testnm.t1;

# Даем все привелении для схемы testnm на группу readonly
grant all privileges on schema testnm to group readonly;
# Даем все привелегии на таблицы для схемы testnm для пользователя readonly
grant all privileges on all tables in schema testnm to group readonly;
testdb=>
select * from t1;
 c1 
----
  1
(1 row)
```

> Доступ получилил.

```sql - testread
create table t2(c1 integer); 
insert into t2 values (2);
```

> Записи вставились. По умолчанию public доступен всем. А так как права на вставку есть то и в public sheme встала таблиа. ЧТо бы этого не произошло нужно снять права на public для данной роли. **Тут подсмотрел шпаргалку**

```sql
#-- otus
ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly; 

REVOKE CREATE on SCHEMA public FROM public; 
REVOKE ALL on DATABASE testdb FROM public; 

```

```sql
create table t3(c1 integer); 
```


> Теперь права на default отозваны и создавать на public если явно прав не стоит - не сможет

#### Предложения по данному уроку
- Вначале пройтись по всей теории и сделать все по пунктам - за ручку правильно
- Сделать Задание 1 для всех - где будет конретная задача
- Сделать Задачу 2 со звездочкой

> Для меня осталось многое что в темноте - больше каша в голове. Тема большая. Лучше бы не про установку три урока было, а по правам хотя бы два. Автор обьяснял не плохо, но тема большая и тут нужно как дебилам всем обьяснять на кубиках и картинками