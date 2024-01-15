
# readme

Code to reproduce issue https://github.com/ClickHouse/ClickHouse/issues/56646


# steps

## run clickhouse in container

```bash
docker compose up -d
```

## run admin console and prepare DB

Run admin console under `default` user

```bash
docker compose exec -it s11 clickhouse-client -n
```

Initialize DB, user and roles

```sql
CREATE DATABASE test ON CLUSTER test4;
CREATE TABLE test.tt1 ON CLUSTER test4 (id UInt64, value String) ENGINE = MergeTree() ORDER BY id;
CREATE TABLE test.tt2 ON CLUSTER test4 (id UInt64, value String) ENGINE = MergeTree() ORDER BY id;
CREATE USER sa_user ON CLUSTER test4;
CREATE ROLE sa_role ON CLUSTER test4;
GRANT SHOW, SELECT ON test.tt1 TO sa_role ON CLUSTER test4;
GRANT sa_role TO sa_user ON CLUSTER test4;
```

## run user console and check permissions

```bash
docker compose exec -it s11 clickhouse-client -u sa_user -n
```

Check permissions

```
SHOW GRANTS;
┌─GRANTS───────────────────┐
│ GRANT sa_role TO sa_user │
└──────────────────────────┘
SHOW GRANTS FOR sa_role;
┌─GRANTS FOR sa_role────────────────────────────────────────────────────────────────┐
│ GRANT SHOW TABLES, SHOW COLUMNS, SHOW DICTIONARIES, SELECT ON test.tt1 TO sa_role │
└───────────────────────────────────────────────────────────────────────────────────┘

SELECT * FROM test.tt1
Ok.
```

Закрываем сессию юзера.


## Make changes in role permissions

Run admin console under `default` user

```bash
docker compose exec -it s11 clickhouse-client -n
```

Add new permissions to role

```sql
GRANT SHOW, SELECT ON test.tt2 TO sa_role ON CLUSTER test4;
```


## run user console and check permissions

```bash
docker compose exec -it s11 clickhouse-client -u sa_user -n
```

Check permissions

```sql
SHOW GRANTS;
┌─GRANTS───────────────────┐
│ GRANT sa_role TO sa_user │
└──────────────────────────┘
SHOW GRANTS FOR sa_role;
┌─GRANTS FOR sa_role────────────────────────────────────────────────────────────────┐
│ GRANT SHOW TABLES, SHOW COLUMNS, SHOW DICTIONARIES, SELECT ON test.tt1 TO sa_role │
│ GRANT SHOW TABLES, SHOW COLUMNS, SHOW DICTIONARIES, SELECT ON test.tt2 TO sa_role │
└───────────────────────────────────────────────────────────────────────────────────┘

SELECT * FROM test.tt1
Ok.
SELECT * FROM test.tt2
Ok.
```

Not reproduced. And new grants transparantly applies to current session (without reconnection).

