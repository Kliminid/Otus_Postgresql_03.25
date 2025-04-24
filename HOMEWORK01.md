1. Создал ВМ с помощью Virtual box

2. В настройках Virtual box в разделе "сеть" заменил тип подключения с NAT на Сетевой мост,чтобы можно было подключаться по ssh

3. Зашел от рута и установил утилиту openssh-server

```sql
su - root
apt install openssh-server
```
4. Затем по ssh подключился к ВМ с помощью MobaXterm(Для удобства)

5. Обновляю список пакетов и устанавливаю postgresql 
```sql
apt update
apt install postgresql 
```
6. Проверяем статус postgresql 
```sql 
systemctl status postgresql 
```
7. Запускаю везде psql от пользователя postgres
```sql 
su - postgres
psql
```
8. Выключаю auto commit в двух сессиях.
```sql 
\set AUTOCOMMIT off
```
9. В первой сессии создаю новую таблицу и наполняю ее данными
```sql
 CREATE TABLE persons(id serial, first_name text, second_name text);
 INSERT INTO persons(first_name, second_name) VALUES('ivan', 'ivanov');
 INSERT INTO persons(first_name, second_name) VALUES('petr', 'petrov');
 COMMIT;
```
10. Проверяю уровень изоляции в обеих сессиях
```sql 
SHOW transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
11. Начинаю новую транзакцию в обеих сессиях с дефолтным (не меняя) уровнем изоляции
```sql 
BEGIN;
```
12. В первой сессии добавляю новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
```sql 
INSERT INTO persons(first_name, second_name) VALUES('sergey', 'sergeev');
```
13. Во второй сессии делаю select from persons. Новой записи нет,потому что уровень изоляции транзакции read committed. Транзакция в первой сессии еще не зафиксирована.
```sql 
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
14.  Завершаю транзакцию в первой сессии
```sql 
COMMIT;
```
15. Во второй сессии делаю  select from persons,  новая запись появилась, так как транзакция была зафиксирована в первой сесси и теперь изменения видны другим транзакциям с уровнем изоляции read committed.Завершаю вторую сессию.
```sql
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
COMMIT;
```
16. Начинаю новые но уже repeatable read транзации.
```sql 
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```
17. В первой сессии делаю INSERT INTO persons(first_name, second_name) VALUES('sveta', 'svetova');
```sql 
INSERT INTO persons(first_name, second_name) VALUES('sveta', 'svetova');
```
18. Во второй сессии делаю SELECT * FROM persons; Новую запись я не вижу, так как выставлен уровень repeatable read.Транзакция с таким уровнем видит видит “снимок” базы данных на момент начала транзакции. Все изменения, сделанные другими транзакциями после начала этой транзакции, будут невидимы.
```sql 
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
19. Завершаю первую транзакцию, делаю SELECT * FROM persons во второй сессии.Новую запись я не увидел, так как транзакция во 2-ой сессии видит все тот же "снимок" бд на момент начала этой транзакции.
```sql 
COMMIT;
во 2 сессии:
SELECT * FROM persons;
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

```
20. Завершаю вторую транзакцию и делаю select * from persons второй сессии. Новые данные вижу, так как происходит чтение нового “снимка” бд, включающего все зафиксированные изменения, которые произошли до этого момента.
```sql 
COMMIT;
postgres=# SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
