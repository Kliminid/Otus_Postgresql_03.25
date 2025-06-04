
1. Создал  ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
2. Установил на вм PostgreSQL 15 с дефолтными настройками
3. Создал БД для тестов: выполнить pgbench -i postgres
4. Запустил pgbench -c8 -P 6 -T 60 -U postgres postgres. Выдал такой резульат:
```
progress: 6.0 s, 413.3 tps, lat 19.244 ms stddev 15.015
progress: 12.0 s, 532.2 tps, lat 15.011 ms stddev 12.007
progress: 18.0 s, 592.0 tps, lat 13.482 ms stddev 10.414
progress: 24.0 s, 589.9 tps, lat 13.543 ms stddev 10.038
progress: 30.0 s, 597.0 tps, lat 13.379 ms stddev 9.974
progress: 36.0 s, 590.1 tps, lat 13.523 ms stddev 10.307
progress: 42.0 s, 599.6 tps, lat 13.326 ms stddev 9.966
progress: 48.0 s, 595.3 tps, lat 13.417 ms stddev 10.143
progress: 54.0 s, 593.8 tps, lat 13.448 ms stddev 9.664
progress: 60.0 s, 599.2 tps, lat 13.329 ms stddev 9.283
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 34222
latency average = 14.001 ms
latency stddev = 10.735 ms
initial connection time = 19.741 ms
tps = 570.369338 (without initial connection time)
```
5. Применил параметры настройки PostgreSQL из прикрепленного к материалам занятия файла.
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
6. Запустил pgbench еще раз и tps стал меньше. Это связано с тем,что параметры,которые я применил в конфиге постгреса предназначены для систем с большим объемом памяти и количеством ядер.Когда я увеличил shared_buffers, effective_cache_size, work_mem, то выделил больше памяти для PostgreSQL но снизил производительность OC.
```
progress: 6.0 s, 377.1 tps, lat 21.053 ms stddev 15.562
progress: 12.0 s, 419.7 tps, lat 19.076 ms stddev 16.069
progress: 18.0 s, 566.8 tps, lat 14.095 ms stddev 10.425
progress: 24.0 s, 568.5 tps, lat 14.053 ms stddev 10.280
progress: 30.0 s, 559.5 tps, lat 14.284 ms stddev 9.919
progress: 36.0 s, 590.8 tps, lat 13.532 ms stddev 9.615
progress: 42.0 s, 591.0 tps, lat 13.523 ms stddev 9.960
progress: 48.0 s, 587.3 tps, lat 13.595 ms stddev 9.998
progress: 54.0 s, 589.0 tps, lat 13.573 ms stddev 9.903
progress: 60.0 s, 590.6 tps, lat 13.509 ms stddev 9.822
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 32651
latency average = 14.683 ms
latency stddev = 11.274 ms
initial connection time = 20.416 ms
tps = 544.149950 (without initial connection time)
```
7. Создал таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк и посмотрел размер файла.Для этого узнал oid базы и таблицы.Размер таблицы равен 66MB
```
SELECT oid FROM pg_database WHERE datname = 'postgres';

SELECT pg_class.oid, pg_size_pretty(pg_relation_size(pg_class.oid))
FROM pg_class
JOIN pg_namespace ON pg_class.relnamespace = pg_namespace.oid
WHERE pg_class.relname = 'test_table' AND pg_namespace.nspname = 'public';

du -h /var/lib/postgresql/15/main/base/13799/16421
```
8.  5 раз обновил все строчки и добавил к каждой строчке любой символ
9. Посмотрел количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```
SELECT relname, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_all_tables
WHERE relname = 'test_table';
  relname   | n_dead_tup |        last_autovacuum        |       last_autoanalyze
------------+------------+-------------------------------+-------------------------------
 test_table |          0 | 2025-06-04 12:05:15.785203+03 | 2025-06-04 12:05:16.786438+03
(1 row)

```
10. Подождал 10 минут, проверяя, пришел ли автовакуум, а потом 5 раз обновил все строчки и добавил к каждой строчке любой символ
```
UPDATE test_table SET data = data || 'a';
```
11.  Размер таблицы стал  439MB. Размер увеличился так как при обновлении строки PostgreSQL не удаляет старую версию, а создает новую.
12. Отключил автовакуум на конкретной таблице
```
ALTER TABLE test_table SET (autovacuum_enabled = false);
```
13. 10 раз обновил все строчки и добавил к каждой строчке любой символ и посмотрел размер таблицы. Размер стал 879MB. Размер увеличился так как я отключил автовакуум и старые версии строк накапливаются в таблице.



