1. Получение deadlock

```postgresql
ERROR:  deadlock detected
DETAIL:  Process 65 waits for ShareLock on transaction 911; blocked by process 48.
Process 48 waits for ShareLock on transaction 912; blocked by process 65.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
```

Логи через сам докер 
```text
2025-03-24 17:19:13.644 UTC [65] ERROR:  deadlock detected
2025-03-24 17:19:13.644 UTC [65] DETAIL:  Process 65 waits for ShareLock on transaction 911; blocked by process 48.
	Process 48 waits for ShareLock on transaction 912; blocked by process 65.
	Process 65: update accounts set amount = amount + 200  where id = 1;
	Process 48: update accounts set amount = amount + 100 where id=2;
2025-03-24 17:19:13.644 UTC [65] HINT:  See server log for query details.
2025-03-24 17:19:13.644 UTC [65] CONTEXT:  while updating tuple (0,1) in relation "accounts"
2025-03-24 17:19:13.644 UTC [65] STATEMENT:  update accounts set amount = amount + 200  where id = 1;
```