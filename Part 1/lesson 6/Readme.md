Single-инстанс (primary)
```text
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
number of transactions per client: 1000
number of transactions actually processed: 10000/10000
number of failed transactions: 0 (0.000%)
latency average = 1.488 ms
initial connection time = 7.943 ms
tps = 6721.441937 (without initial connection time)
```

асинхронная реплика

```text
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
number of transactions per client: 1000
number of transactions actually processed: 10000/10000
number of failed transactions: 0 (0.000%)
latency average = 1.612 ms
TPS = 5812.441937 (without initial connection time)
```

