
1. Cоздал новый кластер PostgresSQL 14

2. Зашел в созданный кластер под пользователем postgres и создал бд testdb
```
CREATE DATABASE testdb;
```
3. Зашел в созданную базу данных под пользователем postgres
```
\c testdb
```
4. Cоздал новую схему testnm
```
CREATE SCHEMA testnm;
```
5. Создал новую таблицу t1 с одной колонкой c1 типа integer и вставил строку со значением c1=1
```
CREATE TABLE t1 (c1 integer);
INSERT INTO t1 VALUES (1)
```
6. Cоздал новую роль readonly и дал новой роли право на подключение к базе данных testdb,право на использование схемы testnm,право на select для всех таблиц схемы testnm
```
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
```
7. Cоздал пользователя testread с паролем test123
```
CREATE USER testread WITH PASSWORD 'test123';
```
8. Дал роль readonly пользователю testread
```
GRANT readonly TO testread;
```
9. Зашел под пользователем testread в базу данных testdb и сделал select * from t1; Вышла ошибка, так как при создании таблицы я не указал схему и t1 по умолчанию создалась в схеме public, а я давал пользователю readonly  права SELECT только на таблицы в схеме testnm.
```
 \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```
10.  Вернулся в базу данных testdb под пользователем postgres
```
psql -d testdb
```
11. Удалил таблицу t1 и создал ее заново но уже с явным указанием имени схемы testnm и вставил строку со значением c1=1
```
DROP TABLE t1;
CREATE TABLE testnm.t1 (c1 integer);
INSERT INTO testnm.t1 VALUES (1);
```
12. Зашел под пользователем testread в базу данных testdb и сделал select * from testnm.t1, вышла ошибка так как grant SELECT on all TABLEs in SCHEMA testnm TO readonly дал доступ только для существующих на тот момент времени таблиц, а t1 пересоздавалась. ALTER default будет действовать для новых таблиц, а grant SELECT on all TABLEs in SCHEMA testnm TO readonly отработал только для существующих на тот момент времени. Надо сделать снова или grant SELECT или пересоздать таблицу

13.  Попробовал выполнить команду create table t2(c1 integer); insert into t2 values (2); Вышла ошибка, потому что пользователю testread была предоставлена роль readonly, которой не предоставлено право на создание таблиц. По умолчанию, пользователи имеют право на создание объектов только в схеме public (если не настроено иначе). Роли readonly это право не давалось.
Следует сделать так:
```
REVOKE CREATE ON SCHEMA public FROM readonly;
```
14. При попытке выполнить команду create table t3(c1 integer); insert into t2 values (2) возникла ошибка, так как пользователь testread (с ролью readonly) больше не имеет права на создание таблиц в схеме public, потому что мы отменили это право для роли readonly.
