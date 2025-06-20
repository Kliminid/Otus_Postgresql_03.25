
1. Изучив структуры базы данных, я выбрал для секционирования таблицу flights, так как она содержит большое количество записей (262,788 в демо-базе) и имеет явный временной критерий - дату вылета (scheduled_departure).
2. Я выбрал секционирование по диапазону на основе даты вылета (scheduled_departure), так как это естественный критерий для запросов к рейсам и позволяет эффективно отсекать большие объемы данных при запросах по периоду.
3.  Создал основную секционированную таблицу
```
CREATE TABLE flights_partitioned (
    flight_id serial NOT NULL,
    flight_no character(6) NOT NULL,
    scheduled_departure timestamptz NOT NULL,
    scheduled_arrival timestamptz NOT NULL,
    departure_airport character(3) NOT NULL,
    arrival_airport character(3) NOT NULL,
    status character varying(20) NOT NULL,
    aircraft_code character(3) NOT NULL,
    actual_departure timestamptz,
    actual_arrival timestamptz,
    CONSTRAINT flights_partitioned_pkey PRIMARY KEY (flight_id, scheduled_departure),
    CONSTRAINT flights_partitioned_check CHECK (scheduled_arrival > scheduled_departure),
    CONSTRAINT flights_partitioned_check1 CHECK (actual_arrival IS NULL OR actual_departure IS NOT NULL AND actual_arrival IS NOT NULL OR actual_arrival > actual_departure),
    CONSTRAINT flights_partitioned_aircraft_code_fkey FOREIGN KEY (aircraft_code) REFERENCES aircrafts_data(aircraft_code),
    CONSTRAINT flights_partitioned_arrival_airport_fkey FOREIGN KEY (arrival_airport) REFERENCES airports_data(airport_code),
    CONSTRAINT flights_partitioned_departure_airport_fkey FOREIGN KEY (departure_airport) REFERENCES airports_data(airport_code)
) PARTITION BY RANGE (scheduled_departure);
```
4. Создал секции по годам (2016-2017)
```
CREATE TABLE flights_y2016 PARTITION OF flights_partitioned
    FOR VALUES FROM ('2016-01-01') TO ('2017-01-01');

CREATE TABLE flights_y2017 PARTITION OF flights_partitioned
    FOR VALUES FROM ('2017-01-01') TO ('2018-01-01');
```
5. Cкопировал данные из исходной таблицы в секционированную и создал индексы для каждой секции
```
INSERT INTO flights_partitioned
SELECT * FROM flights;


CREATE INDEX ON flights_y2016 (departure_airport);
CREATE INDEX ON flights_y2016 (arrival_airport);
CREATE INDEX ON flights_y2016 (status);
CREATE INDEX ON flights_y2016 (scheduled_departure);

CREATE INDEX ON flights_y2017 (departure_airport);
CREATE INDEX ON flights_y2017 (arrival_airport);
CREATE INDEX ON flights_y2017 (status);
CREATE INDEX ON flights_y2017 (scheduled_departure);
```
6. Переименовывал таблицы и перенаправил внешние ключи
```
ALTER TABLE flights RENAME TO flights_old;
ALTER TABLE flights_partitioned RENAME TO flights;

ALTER TABLE ticket_flights
    DROP CONSTRAINT ticket_flights_flight_id_fkey,
    ADD CONSTRAINT ticket_flights_flight_id_fkey FOREIGN KEY (flight_id) REFERENCES flights(flight_id);
    
ALTER TABLE boarding_passes
    DROP CONSTRAINT boarding_passes_flight_id_fkey,
    ADD CONSTRAINT boarding_passes_flight_id_fkey FOREIGN KEY (flight_id) REFERENCES flights(flight_id);
```
7. Тестирование производительности до секционирования по выборке рейсов за определенный месяц:
```
EXPLAIN ANALYZE
SELECT * FROM flights_old 
WHERE scheduled_departure BETWEEN '2017-08-01' AND '2017-08-31';

Seq Scan on flights_old  (cost=0.00..5843.56 rows=12733 width=63) (actual time=0.013..25.763 rows=12733 loops=1)
  Filter: ((scheduled_departure >= '2017-08-01 00:00:00+00'::timestamp with time zone) AND (scheduled_departure <= '2017-08-31 00:00:00+00'::timestamp with time zone))
  Rows Removed by Filter: 250055
Planning Time: 0.097 ms
Execution Time: 26.243 ms
```
После:
```
EXPLAIN ANALYZE
SELECT * FROM flights 
WHERE scheduled_departure BETWEEN '2017-08-01' AND '2017-08-31';

Append  (cost=0.00..1271.76 rows=12733 width=63) (actual time=0.009..7.118 rows=12733 loops=1)
  ->  Seq Scan on flights_y2017 flights_1  (cost=0.00..1271.76 rows=12733 width=63) (actual time=0.008..5.444 rows=12733 loops=1)
        Filter: ((scheduled_departure >= '2017-08-01 00:00:00+00'::timestamp with time zone) AND (scheduled_departure <= '2017-08-31 00:00:00+00'::timestamp with time zone))
Planning Time: 0.089 ms
Execution Time: 7.677 ms
```
Можно заметить,что время выполнения уменьшилось с 26.243 ms до 7.677 ms.

8. Агрегация по статусам рейсов за год до секционирования:
```
EXPLAIN ANALYZE
SELECT status, COUNT(*) 
FROM flights_old 
WHERE scheduled_departure BETWEEN '2017-01-01' AND '2017-12-31'
GROUP BY status;

HashAggregate  (cost=6372.32..6372.37 rows=5 width=13) (actual time=42.640..42.641 rows=5 loops=1)
  Group Key: status
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on flights_old  (cost=0.00..5843.56 rows=105735 width=6) (actual time=0.011..23.937 rows=105735 loops=1)
        Filter: ((scheduled_departure >= '2017-01-01 00:00:00+00'::timestamp with time zone) AND (scheduled_departure <= '2017-12-31 00:00:00+00'::timestamp with time zone))
        Rows Removed by Filter: 157053
Planning Time: 0.078 ms
Execution Time: 42.669 ms
```
После:
```
EXPLAIN ANALYZE
SELECT status, COUNT(*) 
FROM flights 
WHERE scheduled_departure BETWEEN '2017-01-01' AND '2017-12-31'
GROUP BY status;
 
 HashAggregate  (cost=1271.76..1271.81 rows=5 width=13) (actual time=15.186..15.187 rows=5 loops=1)
  Group Key: status
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on flights_y2017 flights_1  (cost=0.00..1271.76 rows=105735 width=6) (actual time=0.006..7.057 rows=105735 loops=1)
        Filter: ((scheduled_departure >= '2017-01-01 00:00:00+00'::timestamp with time zone) AND (scheduled_departure <= '2017-12-31 00:00:00+00'::timestamp with time zone))
Planning Time: 0.077 ms
Execution Time: 15.210 ms
```
Можно заметить,что время выполнения уменьшилось с 42.669 ms до 15.210 ms

8. Вывод: Секционирование таблицы flights по дате вылета дало значительное улучшение производительности для запросов, фильтрующих данные по временным диапазонам. Основные преимущества:уменьшение времени выполнения запросов в 2.5-3.5 раза для типовых запросов по периодам,снижение нагрузки на ввод/вывод за счет работы с меньшими объемами данных, упрощение управления данными - можно легко добавлять/удалять секции за определенные периоды и возможность оптимизации индексов для каждой секции отдельно.












