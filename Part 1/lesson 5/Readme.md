1. Режим session

```text
pgbench -h 127.0.0.1 -p 6432 -n -U postgres -d wb -f ~/workload.sql -c 100 -j 4 -T 10
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /root/workload.sql
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 956730
number of failed transactions: 0 (0.000%)
latency average = 1.044 ms
initial connection time = 15.335 ms
tps = 95739.883883 (without initial connection time)
```
2. Режим Transaction

```text
pgbench -h 127.0.0.1 -p 6432 -n -U postgres -d wb -f ~/workload.sql -c 100 -j 4 -T 10
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /root/workload.sql
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 1080596
number of failed transactions: 0 (0.000%)
latency average = 0.924 ms
initial connection time = 15.192 ms
tps = 108212.742673 (without initial connection time)
```

3. Режим Statement

```text
pgbench -h 127.0.0.1 -p 6432 -n -U postgres -d wb -f ~/workload.sql -c 100 -j 4 -T 10
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /root/workload.sql
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 1405058
number of failed transactions: 0 (0.000%)
latency average = 0.711 ms
initial connection time = 10.101 ms
tps = 140631.003783 (without initial connection time)
```

Результат
- Session — для приложений, требующих сохранения состояния (например, временные таблицы).
- Transaction — лучший выбор для большинства веб-приложений.
- Statement — для максимальной производительности (если не нужны транзакции).