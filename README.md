# Monitor 101 Lab Demo.

## Lab's objectives:

1. In this lab you will troubleshoot errors that prevent the workloads to start.

2. You will also get familiar with Monitoring Integration.

3. Execute a Postgres load test and see how the metrics are reported in Monitor.


## 1. Checkout branch "with-errors"

## 2. Create the following deployments in your K8s cluster

```
kubectl create ns vote
kubectl apply -f db-deployment.yaml
kubectl apply -f redis-deployment
kubectl apply -f db-service.yaml
kubectl apply -f redis-service.yaml
kubectl apply -f result-deployment.yaml
kubectl apply -f result-service.yaml
kubectl apply -f vote-deployment.yaml
kubectl apply -f vote-service.yaml
kubectl apply -f worker-deployment.yaml
```

## 3. Navigate to Sysdig Monitor and check the Advisor data.

Using Troubleshooting advisor, find the errors that are preventing the containers to start.

## 4. Apply the appropiate solution to the redis and postgres deployments.

Change the appropiate configuration solutions in the workloads.

## 5. Configure the Monitor Integration for redis-deployment and db-deployment

Navigate to Monitor Integration and find the applications detected by Sysdig. 

### 5.1. Redis integration

#### 5.1.1. Login to the pod and create redis user and password 

Note: you can use the code from the integration page's instruction as reference.

Ej:
```
% kubectl -n vote exec -ti redis-6986c5d458-jw6h2 -- /bin/sh
/data # redis-cli
127.0.0.1:6379> ACL SETUSER redis +client +ping +info +config|get +cluster|info +slowlog +latency +memory +select +get +scan +xinfo +type +pfcount +strlen +llen +scard +zcard +hlen +xlen +eval allkeys on >redis
OK
127.0.0.1:6379> exit
/data # exit
```

#### 5.1.2. Create K8s redis secret
```
% kubectl create secret -n vote generic redis-exporter-auth \
  --from-literal=user=redis \
  --from-literal=password=redis
```

#### 5.1.3. Install the redis exporter

```
helm install -n vote -f redis-exporter-values.yaml --repo https://sysdiglabs.github.io/integrations-charts vote-redis redis-exporter
```

#### 5.1.4. Check redis exporter logs:

Ej:
```
% kubectl -n vote logs redis-exporter-vote-redis-deploy-7988c8857c-4xr25
time="2023-02-27T21:07:40Z" level=info msg="Redis Metrics Exporter [no-tag]    build date:     sha1: [no-sha]    Go: go1.17.8    GOOS: linux    GOARCH: amd64"
time="2023-02-27T21:07:40Z" level=info msg="Providing metrics at :9121/metrics"
```

#### 5.2. Postgres Integration

#### 5.2.1. Configure the postgres user.

Note: you can use the code from the integration page's instruction as reference.

Ej:

```
% kubectl -n vote exec -ti db-85d5d76699-mgwfz -- /bin/bash
root@db-85d5d76699-mgwfz:/# psql -h localhost -U postgres 
psql (9.4.26)
Type "help" for help.

postgres=# CREATE OR REPLACE FUNCTION __tmp_create_user() returns void as $$
postgres$# BEGIN
postgres$#   IF NOT EXISTS (
postgres$#           SELECT                       -- SELECT list can stay empty for this
postgres$#           FROM   pg_catalog.pg_user
postgres$#           WHERE  usename = 'postgres_exporter') THEN
postgres$#     CREATE USER postgres_exporter;
postgres$#   END IF;
postgres$# END;
postgres$# $$ language plpgsql;
CREATE FUNCTION
postgres=# 
postgres=# SELECT __tmp_create_user();
 __tmp_create_user 
-------------------
 
(1 row)

postgres=# DROP FUNCTION __tmp_create_user();
DROP FUNCTION
postgres=# 
postgres=# ALTER USER postgres_exporter WITH PASSWORD 'password';
ALTER ROLE
postgres=# ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;
ALTER ROLE
postgres=# 
postgres=# -- If deploying as non-superuser (for example in AWS RDS), uncomment the GRANT
postgres=# -- line below and replace <MASTER_USER> with your root user.
postgres=# -- GRANT postgres_exporter TO <MASTER_USER>;
postgres=# CREATE SCHEMA IF NOT EXISTS postgres_exporter;
CREATE SCHEMA
postgres=# GRANT USAGE ON SCHEMA postgres_exporter TO postgres_exporter;
GRANT
postgres=# GRANT CONNECT ON DATABASE postgres TO postgres_exporter;
GRANT
postgres=# 
postgres=# CREATE OR REPLACE FUNCTION get_pg_stat_activity() RETURNS SETOF pg_stat_activity AS
postgres-# $$ SELECT * FROM pg_catalog.pg_stat_activity; $$
postgres-# LANGUAGE sql
postgres-# VOLATILE
postgres-# SECURITY DEFINER;
CREATE FUNCTION
postgres=# 
postgres=# CREATE OR REPLACE VIEW postgres_exporter.pg_stat_activity
postgres-# AS
postgres-#   SELECT * from get_pg_stat_activity();
CREATE VIEW
postgres=# 
postgres=# GRANT SELECT ON postgres_exporter.pg_stat_activity TO postgres_exporter;
GRANT
postgres=# 
postgres=# CREATE OR REPLACE FUNCTION get_pg_stat_replication() RETURNS SETOF pg_stat_replication AS
postgres-# $$ SELECT * FROM pg_catalog.pg_stat_replication; $$
postgres-# LANGUAGE sql
postgres-# VOLATILE
postgres-# SECURITY DEFINER;
CREATE FUNCTION
postgres=# 
postgres=# CREATE OR REPLACE VIEW postgres_exporter.pg_stat_replication
postgres-# AS
postgres-#   SELECT * FROM get_pg_stat_replication();
CREATE VIEW
postgres=# 
postgres=# GRANT SELECT ON postgres_exporter.pg_stat_replication TO postgres_exporter;
GRANT
postgres=# ALTER USER postgres_exporter WITH PASSWORD 'password';
ALTER ROLE
```

#### 5.2.2. Create K8s secret:

```
kubectl create -n vote secret generic postgresql-exporter \
  --from-literal=username=postgres_exporter \
  --from-literal=password=password

```

#### 5.2.3. Create Postgres exporter

```
helm install -n vote -f postgres-exporter-values.yaml --repo https://sysdiglabs.github.io/integrations-charts vote-db postgresql-exporter
```

#### 5.2.4. Check prometheus exporter pod logs:

Ej:
```
% kubectl -n vote logs redis-exporter-vote-redis-deploy-7988c8857c-4xr25
time="2023-02-27T21:07:40Z" level=info msg="Redis Metrics Exporter [no-tag]    build date:     sha1: [no-sha]    Go: go1.17.8    GOOS: linux    GOARCH: amd64"
time="2023-02-27T21:07:40Z" level=info msg="Providing metrics at :9121/metrics"
```

## 6. Check the integration dashboards in Monitor.

After the integration is configured, Sysdig will give more dashboards templates for monitoring internal application's metrics. 

## 7. Login to DB's pod and execute a load test with pgbench:

In order to simulate real application usage, login to the Postgres's pod and execute a load test with pgbench.

```
% pgbench -h localhost -p 5432 -U postgres -i -s 1000 -c 1000 -j 100 -T 60 postgres
```

## 8. Navigate to the following Dashboards:

- PostgreSQL Database Details
- PostgreSQL Instance Health
- Node Status & Performance
- Workload Status & Performance

Notice the changes in the applications performance after running pgbench.

## 9. Create a notification channel.

Enable at least 1 notification channel that will be used for the alert creation.

## 10. Create an alert.

From some of the dashboards that are showing more activity create an alert and send it to a notification channel.



