
1. Развернул виртуальную машину с 4 vCPU и 8 GB RAM
2. Поставил на нее PostgreSQL 15
3. Приступил к редактированию конфиг. файла в /etc/postgresql/15/main/postgresql.conf

В разделе memory выставил параметры так:
```
shared_buffers = 2048MB
temp_buffers = 32MB
work_mem = 32MB
maintenance_work_mem = 320MB
effective_io_concurrency = 100
effective_cache_size = 6GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
```
В разделе Checkpoints выставил параметры так:
```
max_wal_size = 10240MB
min_wal_size = 5120MB
```

Также настроил параметры автовакуума:
```
autovaccuum
  - { option: "autovacuum",       value: "on" }
  - { option: "autovacuum_max_workers",       value: "2" }
  - { option: "autovacuum_vacuum_scale_factor",       value: "0.1" }
  - { option: "autovacuum_analyze_threshold",       value: "100" }
  - { option: "autovacuum_analyze_scale_factor",       value: "0.2" }
  - { option: "autovacuum_freeze_max_age",       value: "400000000" }
  - { option: "autovacuum_multixact_freeze_max_age",       value: "800000000" }
  - { option: "autovacuum_vacuum_cost_delay",       value: "10" }
  - { option: "autovacuum_vacuum_cost_limit",       value: "1000" }
  - { option: "autovacuum_naptime",       value: "120" }
  ```
И параметры логирования:
```
  - { option: 'log_line_prefix', value: '%t [%p]: db=%d,user=%u,app=%a,client=%h '}
  - { option: 'log_rotation_age', value: '1d'}
  - { option: 'log_truncate_on_rotation', value: 'on'}
  - { option: 'log_min_messages', value: 'info'}
  - { option: 'log_statement', value: 'ddl'}
  - { option: 'log_min_error_statement', value: 'error'}
  - { option: 'log_error_verbosity', value: 'default'}
  - { option: 'log_min_duration_statement', value: '3000'}
  - { option: 'log_checkpoints', value: 'on'}   
  - { option: 'log_connections', value: 'on'}   
  - { option: 'log_disconnections', value: 'on'}   
  - { option: 'log_lock_waits', value: '1'}
```
4.  Перезапустил postgresql 
5. Создал базу данных homework, а дальше начал нагружать с помощью утилиты pgbench:

Cначала проинициализировал
```
 pgbench -i -s 10 homework
 dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
1000000 of 1000000 tuples (100%) done (elapsed 0.98 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.89 s (drop tables 0.00 s, create tables 0.03 s, client-side generate 1.04 s, vacuum 0.21 s, primary keys 0.61 s).
```
Потом дал нагрузку:
```
pgbench -c 50 -j 4 -t 60 homework

scaling factor: 10
query mode: simple
number of clients: 50
number of threads: 4
number of transactions per client: 60
number of transactions actually processed: 3000/3000
latency average = 28.154 ms
initial connection time = 38.099 ms
tps = 1775.925553 (without initial connection time)
```
6. В итоге значение tps = 1775.925553
