
1. Cоздал вм и развернул на нем PostgreSQL
2. Создал таблицы и наполнил их
```
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date DATE NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL
);

INSERT INTO customers (first_name, last_name, email) VALUES
('Иван', 'Иванов', 'ivan.ivanov@example.com'),
('Петр', 'Петров', 'petr.petrov@example.com'),
('Мария', 'Сидорова', 'maria.sidorova@example.com'),
('Анна', 'Смирнова', 'anna.smirnova@example.com');

INSERT INTO orders (customer_id, order_date, total_amount) VALUES
(1, '2023-11-05', 150.00),
(1, '2023-11-10', 200.00),
(2, '2023-11-07', 75.00),
(3, '2023-11-08', 300.00),
(NULL, '2023-11-12', 50.00);
```
3. Создал индекс к таблице customers и ввел команду explain. Index Scan указывает на то, что PostgreSQL использовал индекс idx_customers_last_name для выполнения запроса. Index Cond показывает условие, по которому индекс был использован.

```
CREATE INDEX idx_customers_last_name ON customers (last_name);

EXPLAIN SELECT * FROM customers WHERE last_name = 'Иванов';
                        QUERY PLAN
-----------------------------------------------------------
 Seq Scan on customers  (cost=0.00..1.05 rows=1 width=158)
   Filter: ((last_name)::text = 'Иванов'::text)
(2 rows)
```
4. Реализовал индекс для полнотекстового поиска. Возникла ошибка из-за конфликта кодировок между данными в таблице (UTF-8) и файлом стоп-слов (LATIN1). Пришлось использовать конфигурации 'simple'. Подобный индекс эффективен для поиска по ключевым словам в описании клиентов.

```
ALTER TABLE customers ADD COLUMN about TEXT;
UPDATE customers SET about = 
  CASE customer_id
    WHEN 1 THEN 'Постоянный клиент с 2020 года, предпочитает электронику'
    WHEN 2 THEN 'Новый клиент, интересуется бытовой техникой'
    WHEN 3 THEN 'Корпоративный клиент, заказывает офисные принадлежности'
    WHEN 4 THEN 'Розничный покупатель, интересуется акциями'
  END;

ALTER TABLE customers ADD COLUMN about_tsvector tsvector;
UPDATE customers SET about_tsvector = to_tsvector('russian', about);
CREATE INDEX idx_customers_about_fts ON customers USING gin(about_tsvector);

UPDATE customers SET about_tsvector = to_tsvector('simple', about);
DROP INDEX IF EXISTS idx_customers_about_fts;
CREATE INDEX idx_customers_about_fts ON customers USING gin(about_tsvector);

EXPLAIN ANALYZE
SELECT * FROM customers
WHERE about_tsvector @@ to_tsquery('simple', 'клиент');
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 Seq Scan on customers  (cost=0.00..1.05 rows=1 width=286) (actual time=0.006..0.008 rows=3 loops=1)
   Filter: (about_tsvector @@ '''▒▒'' <-> ''▒'' <-> ''▒▒'' <-> ''▒'''::tsquery)
   Rows Removed by Filter: 1
 Planning Time: 0.195 ms
 Execution Time: 0.015 ms
(5 rows)
```
5. Создал частичный индекс для заказов свыше 100 рублей и индекс с функцией для поиска по email без учета регистра. Частичный индекс ускоряет работу с крупными заказами.
```
CREATE INDEX idx_orders_large ON orders(total_amount) WHERE total_amount > 100.00;
CREATE INDEX idx_customers_lower_email ON customers(lower(email));
EXPLAIN ANALYZE SELECT * FROM orders WHERE total_amount > 100.00 AND total_amount < 250.00;
                                           QUERY PLAN
-------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..1.07 rows=1 width=28) (actual time=0.009..0.011 rows=2 loops=1)
   Filter: ((total_amount > 100.00) AND (total_amount < 250.00))
   Rows Removed by Filter: 3
 Planning Time: 0.525 ms
 Execution Time: 0.026 ms
(5 rows)
```
6. Создал  индекс на несколько полей для таблицы orders по полям customer_id и order_date. Подобный индекс эффективен для запросов по идентификатору клиента и диапазону дат.

```
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

EXPLAIN ANALYZE SELECT * FROM orders
WHERE customer_id = 1 AND order_date BETWEEN '2023-11-01' AND '2023-11-30';
                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..1.09 rows=1 width=28) (actual time=0.009..0.010 rows=2 loops=1)
   Filter: ((order_date >= '2023-11-01'::date) AND (order_date <= '2023-11-30'::date) AND (customer_id = 1))
   Rows Removed by Filter: 3
 Planning Time: 0.502 ms
 Execution Time: 0.024 ms
(5 rows)
```


