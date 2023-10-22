## Установка и настройка PostgreSQL #3


### Postgres **VM Box** на UbuntuServer 2022

```bash
### Установка postgres
sudo apt udpate
sudo apt upgrade

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt install postgresql-16
```

```bash
### Проверка
sudo pg_lsclusters
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```


```bash
### Подключение
sudo -i -u postgres
psql
```

```sql
postgres=#
create table test(c1 text);
#CREATE TABLE
insert into test values('1');
#INSERT 0 1
```

```bash
### Выход
\q
```


```bash
### Останавливаем postgres
\q
## exit postgres user
exit

### Остановка кластера
sudo pg_ctlcluster 16 main stop

sudo pg_lsclusters
#16  main    5432 down   postgres
```
> Создан и добавлен виртуальный диск через VirtualBox на 10GB. И перезапущена машина


```bash
### Установки дисковой утилиты
sudo apt install parted

sudo parted -l | grep Error
Error: /dev/sdb: unrecognised disk label

pooopsss@pooopsssvd1:~$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0 128,9M  1 loop /snap/docker/2893
loop1                       7:1    0 111,9M  1 loop /snap/lxd/24322
loop2                       7:2    0  73,9M  1 loop /snap/core22/864
loop3                       7:3    0  63,4M  1 loop /snap/core20/1974
loop4                       7:4    0  53,3M  1 loop /snap/snapd/19457
sda                         8:0    0    25G  0 disk 
├─sda1                      8:1    0     1M  0 part 
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0    23G  0 part 
  └─ubuntu--vg-ubuntu--lv 253:0    0  11,5G  0 lvm  /
sdb                         8:16   0    10G  0 disk 
sr0                        11:0    1  1024M  0 rom 
```

> Видим что система увидела диск /dev/sdb 10GB


```bash
### Установки файловой системы и метки тома
sudo mkfs.ext4 -L NEW_DISK /dev/sdb

# Смотрим что получилось
sudo lsblk --fs
sdb  ext4   1.0   NEW_DISK
                        547525ac-1dea-4766-b23c-71d932d48044   

# Монтируем
sudo mkdir -p /mnt/new_disk
sudo mount -o defaults /dev/sdb /mnt/new_disk/

# Добавляем в fstab для автоматического монтирование при перезагрузки
sudo nano /etc/fstab
LABEL=NEW_DISK /mnt/new_disk ext4 defaults 0 2

# Запускаем монтирование
sudo mount -a
# List device mounted
df -h -x tmpfs
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  6,4G  4,3G  60% /
/dev/sda2                          2,0G  251M  1,6G  14% /boot
/dev/sdb                           9,8G   24K  9,3G   1% /mnt/new_disk
```

> Отлично. Создаем файл тестовый и перезагружаем виртуальную машину

```bash
# Проверяем есть ли файл
ls /mnt/new_disk/
test.txt
```

> Отлично. FSTAB сработал. Том смонтирован. Даем права пользователю postgres, что бы кластер мог там что либо создавать

```bash
# Проверяем есть ли файл
ls /mnt/new_disk/
test.txt

# Права
sudo chown -R postgres:postgres /mnt/new_disk/

ls -lah /mnt/
total 12K
drwxr-xr-x  3 root     root     4,0K окт 22 12:28 .
drwxr-xr-x 20 root     root     4,0K окт 19 12:09 ..
drwxr-xr-x  3 postgres postgres 4,0K окт 22 12:33 new_disk
```


```bash
# Останавливаем класетр
sudo pg_ctlcluster 16 main stop

# Переносим данные кластера на новый диск
sudo mv /var/lib/postgresql/16 /mnt/new_disk/

# Пытаемся запустить кластер
sudo pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
```

> Конечно же не получилось, т.к В конфигурационном файле прописан старый путь /var/lib/postgresql/16. А там файлов нет уже. По умолчанию папка для настроек - /etc/postgresql/16/main/


```bash
# Открываем файл настроек
sudo nano /etc/postgresql/16/main/postgresql.conf
# Находим переменную data_directory и выставлеме новое значение
data_directory = '/mnt/new_disk/16/main'

#Сохраняем и выходим и перезапускаем кластер
sudo -u postgres pg_ctlcluster 16 main start

#Сервер запустился. Теперь заходим в ранее созданную базу и смотрим есть ли таблицы ранее созданные
sudo -u postgres psql
postgres=# \dt
          Список отношений
 Схема  | Имя  |   Тип   | Владелец 
--------+------+---------+----------
 public | test | таблица | postgres
(1 строка)

postgres=# select * from test;
 c1 
----
 1
(1 строка)
```

> Данные все на месте. Перенос успешный.

### Задание со звездочкой

> Делаю дубликат машины. Убираю с новой машины все fstab записи и привязки диска. Все сети в VirtualBox делаю как **"Сетевой мост"**. Перезагружаюсь. Получаю две машины со разными ip адресами: 
- 192.168.88.204 - старая
- 192.168.88.203 - новая


```bash
# Запрашиваем новый сетевой адрес на новой машине
sudo dhclient -r enp03s
sudo dhclient -v enp03s


# Смотрим диски 192.168.88.203
df -h -x tmpfs
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  6,4G  4,3G  61% /
/dev/sda2                          2,0G  251M  1,6G  14% /boot
```


```bash
# Заходим в postgresql смотрим что есть 192.168.88.203
df -h -x tmpfs
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  6,4G  4,3G  61% /
/dev/sda2                          2,0G  251M  1,6G  14% /boot
```


```sql
-- Создаем таблицу 192.168.88.203
postgres=#
create table test_new(c1 text);
#CREATE TABLE
insert into test_new values('new_value');
#INSERT 0 1

postgres=# \dt
            Список отношений
 Схема  |   Имя    |   Тип   | Владелец 
--------+----------+---------+----------
 public | test_new | таблица | postgres
(1 строка)

```

> Для подключение катологов удаленно по сети используется технология nfs

```bash
# Установка NFC-server на старую машину 192.168.88.204
sudo apt install nfs-kernel-server

# Установка NFC-common на новую машину 192.168.88.203
sudo apt install nfs-common

# Настраиваем сервер exports 192.168.88.204
sudo nano /etc/exports
/mnt/new_disk   0.0.0.0/0(rw,sync,no_root_squash,no_subtree_check)

# Перезапускаем 192.168.88.204
sudo exportfs -r
sudo systemctl restart nfs-kernel-server

# Перезапускаем 192.168.88.203
sudo mount.ext4 192.168.88.204:/mnt/new_disk /mnt/new_disk 
```
> У меня с версией ubuntu server произошел косяк. Не может смонтировать ни ext4 ни nfs типы. Так что на этом все ибо второй час перебираю все возможные варианты. Расскажу в теории.

> Создав nfs каталог его можно монтировать как nfs в любую систему. Так же в корне если говорим про базу нужно создать каталог с правами postgres и туда перенести как в первой части все. Таким образом можно использовать удаленный каталог как основной. Но думаю это фигня. Скорость работы низкая. Проще оставить основные данные на основных дисках, а через Tablespace раскидать уже можно хоть на nfs

