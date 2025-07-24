
1. Cоздал 3 вм и постави на них postgreSQL
2. на 1 вм отредактировал конфиг и хба файл,сделал ребут и релоад
```
конфиг:
listen_addresses = '*'
wal_level = logical
хба:
host    replication     repl_user       192.168.1.72/32         md5
host    replication     repl_user       192.168.1.78/32         md5
host    postgres     repl_user          192.168.1.72/32         md5
host    postgres     repl_user          192.168.1.78/32         md5
```
3. cоздал таблицы,пользователя для репликации,дал права,создал публикацию
```
CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE test2 (
    id SERIAL PRIMARY KEY,
    data VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE USER repl_user WITH REPLICATION ENCRYPTED PASSWORD 'repl_password';

GRANT SELECT ON TABLE test, test2 TO repl_user;
CREATE PUBLICATION pub_test FOR TABLE test;
```
4. редактура конфига и хба файла на 2 вм, по аналогии с первой,только в хба файле ip 1 вм и 3 вм.
5. Создал таблицу,пользователя для репликации,публикацию для таблицы test2,создал подписку на таблицу test1 с 1 вм
```
CREATE SUBSCRIPTION sub_test 
CONNECTION 'host=192.168.1.71 port=5432 dbname=postgres user=repl_user password=12345' 
PUBLICATION pub_test;
```
6. на 1 вм создал подписку на таблицу test2 c 2 вм
```
CREATE SUBSCRIPTION sub_test2 
CONNECTION 'host=192.168.1.72 port=5432 dbname=postgres user=repl_user password=12345' 
PUBLICATION pub_test2;
```
7. на 3 вм редактура хба файла и конфиг
```
по аналогии с прошлыми вм, но в конфиге еще hot_standby = on
```
8. cоздал таблицы, пользователя,подписки на обе публикации
```
CREATE SUBSCRIPTION sub_test_3vm
CONNECTION 'host=192.168.1.71 port=5432 dbname=postgres user=repl_user password=123456' 
PUBLICATION pub_test;

CREATE SUBSCRIPTION sub_test2_3vm
CONNECTION 'host=192.168.1.72 port=5432 dbname=postgres user=repl_user password=12345' 
PUBLICATION pub_test2;
```
9. Проверяем:
на 1 вм
```
INSERT INTO test (data) VALUES ('Test data from VM1');
```
на 2 вм
```
INSERT INTO test2 (data) VALUES ('Test data from VM2');
```
на 3 вм проверяем 
```
postgres=# SELECT * FROM test;
 id |        data        |         created_at
----+--------------------+----------------------------
  1 | Test data from VM1 | 2025-07-24 16:21:56.366137
(1 row)

postgres=# SELECT * FROM test2;
 id |        data        |         created_at
----+--------------------+----------------------------
  1 | Test data from VM2 | 2025-07-24 16:22:07.974894
(1 row)
```
10. Настраиваем бэкап на 3вм: создали каталог,накинули прав, в cron добавляем задание
```
 mkdir /backup
 chown postgres:postgres /backup
 chmod 0700 /backup
и в кроне:
0 2 * * * pg_dump -Fc postgres > /backup/db_backup_$(date +\%Y-\%m-\%d).dump
```
