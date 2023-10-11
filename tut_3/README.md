## Установка и настройка PostgteSQL в контейнере Docker #2


### Postgres **докере**

```yaml
# postgres/docker-compose.yml:
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
      - ./.docker/postgres/postgresql.conf:/var/lib/postgresql/data/postgresql.conf
```

```bash
# postgres/.docker/posgres/Dockerfile:
FROM postgres:16
COPY ./pg_hba.conf /etc/postgresql/pg_hba.conf
```

```bash
### Запуск контейнера
docker-compose up --build
```

```bash
### Вход в образ
docker exec -ti postgres_otus_db bash
```

####Смотрим подключился ли наш файл pg_hba.conf

```sql
plsql -U otus
show hba_file;
 hba_file               
--------------------------------------
/etc/postgresql/pg_hba.conf
(1 row)
```

```bash
cat /etc/postgresql/pg_hba.conf
....
....
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust

host all all 192.168.130.25/32 scram-sha-256
host all all all scram-sha-256
```
> Видим что файл применился

> С версии Postgres 13 появилось шифрование scram-sha-256. Если база запустилась с md5 и там созданы пользователи, то **нельзя** менять шифрования сразу в настройках базы. Вначале нужно поменять тип шиврование пользователей и лишь потом переводить сервер в новый режим. 

> Проблема с которой столкнулся и непонятно почему. Это то что пришлось *postgresql.conf* через volumes устанавливать, а не *COPY* в *Dockerfile*, как файл *pg_hba.conf*. Почему-то файл перезаписывался. Догадка - файл пересоздается после запуска, а  не до. 



### Postgres Client
#### Модифицируем docker-compose
```yaml
# ostgres/docker-compose.yml:
...
  db_client:
    build:
      context: .docker/postgres_client
    container_name: db_client
    depends_on:
      - db
    links:
      - db
    environment:
      PGDATABASE: ${DB_DATABASE}
      PGHOST: ${PG_HOST}
      PGPORT: 5432
      PGUSER: ${DB_USERNAME}
      PGPASSWORD: ${DB_PASSWORD}
```

```bash
# .docker/postgres_client/Dockerfile:
FROM alpine:latest
RUN apk --no-cache add postgresql15-client
ENTRYPOINT [ "psql" ]
```

```bash
### Запуск контейнера
docker-compose up --build
```

```sql
### Создаем таблицу и пару записей
create table t (i integer, s text);
insert into t values(1, 'One');
insert into t values(1, 'Two');
select * from t;
 i |  s  
---+-----
 1 | One
 1 | Two
(2 rows)

```


```bash
### Выполняем запрос с клиента
docker-compose run --rm db_client -c 'select * from t;'
Creating postgres_db_client_run ... done
 i |  s  
---+-----
 1 | One
 1 | Two
(2 rows)

```


```bash
### Останавливаем сервер
docker-compose down
Removing db_client        ... done
Removing postgres_otus_db ... done
Removing network postgres_default
```


```bash
### Запускаем сервер
docker-compose up --build --force-recreate
```

```bash
### Выполняем запрос с клиента
docker-compose run --rm db_client -c 'select * from t;'
Creating postgres_db_client_run ... done
 i |  s  
---+-----
 1 | One
 1 | Two
(2 rows)

```

> Хотел в разных docker-compose но так нельзя если только на базе не делать network-mode: host или сеть external