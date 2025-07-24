
1. Cоздал вм с помощью виртуал бокс
2. Создал БД, схему и в ней таблицу
```
create database hw_backup;
CREATE DATABASE
postgres=# \c hw_backup;
You are now connected to database "hw_backup" as user "postgres".
hw_backup=# CREATE SCHEMA hw_schema;
CREATE SCHEMA
hw_backup=# CREATE TABLE hw_schema.products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    quantity INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE
```
3. Заполнил таблицы автосгенерированными 100 записями
```
INSERT INTO hw_schema.products (name, price, quantity)
SELECT 
    'Product ' || i,
    (RANDOM() * 1000)::DECIMAL(10,2),
    (RANDOM() * 100)::INT
FROM generate_series(1, 100) AS i;
```
4. Под линукс пользователем Postgres создал каталог для бэкапов
```
mkdir /var/lib/postgresql/backups
 chown postgres:postgres /var/lib/postgresql/backups
 chmod 0700 /var/lib/postgresql/backups
 ```
 5. Сделал логический бэкап используя утилиту COPY,зайдя в мою бд
 ```
\copy hw_schema.products TO '/var/lib/postgresql/backups/products_backup.csv' WITH CSV HEADER;
```
6. Восстановил во вторую таблицу данные из бэкапа
```
CREATE TABLE hw_schema.products_backup AS TABLE hw_schema.products WITH NO DATA;
\copy hw_schema.products_backup FROM '/var/lib/postgresql/backups/products_backup.csv' WITH CSV HEADER;
```
7. Используя утилиту pg_dump создал бэкап в кастомном сжатом формате двух таблиц
```
pg_dump -U postgres -d hw_backup -n hw_schema -t hw_schema.products -t hw_schema.products_backup -F c -f /var/lib/postgresql/backups/custom_backup.dump
```
8. Используя утилиту pg_restore восстановил в новую БД только вторую таблицу!
```
CREATE DATABASE hw_restored;
psql -U postgres -d hw_restored -c "CREATE SCHEMA IF NOT EXISTS hw_schema;"
pg_restore -U postgres -d hw_restored -t 'hw_schema.products_backup' /var/lib/postgresql/backups/custom_backup.dump
```



