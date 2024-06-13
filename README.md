# flink-cdc-demo
# Pre-requisites

- Docker
- Flink
- Flink SQL Client

# Getting Started

Start the local flink cluster. You can start the cluster by running the following command in the $FLINK_HOME directory

```shell
./bin/start-cluster.sh
```
Then you can access the Flink Web UI at http://localhost:8081

Start the local flink sql client

```shell
./bin/sql-client.sh
```
Start services

```shell
docker-compose up -d
```

Run the following command to create the tables and insert some data into MySQL and PostgreSQL

```shell
cat mysql-init.sql | docker-compose -f docker-compose.yaml exec -T mysql mysql -uroot -p123456
```


```shell
cat postgres-init.sql | docker-compose -f docker-compose.yaml exec -T postgres psql -h localhost -U postgres
```

On the Flink SQL Client terminal, create table for source and sink

```sql
SET execution.checkpointing.interval = 3s;
```

Create the source table for the products table in MySQL

```sql
CREATE TABLE products (
id INT,
name STRING,
description STRING,
PRIMARY KEY (id) NOT ENFORCED
) WITH (
'connector' = 'mysql-cdc',
'hostname' = 'localhost',
'port' = '3306',
'username' = 'root',
'password' = '123456',
'database-name' = 'mydb',
'table-name' = 'products',
'server-time-zone' = 'UTC'
);
```

Create the source table for the orders table in MySQL

```sql
CREATE TABLE orders (
order_id INT,
order_date TIMESTAMP(0),
customer_name STRING,
price DECIMAL(10, 5),
product_id INT,
order_status BOOLEAN,
PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
'connector' = 'mysql-cdc',
'hostname' = 'localhost',
'port' = '3306',
'username' = 'root',
'password' = '123456',
'database-name' = 'mydb',
'table-name' = 'orders',
'server-time-zone' = 'UTC'
);
```

Create the source table for the shipments table in PostgreSQL

```sql
CREATE TABLE shipments (
shipment_id INT,
order_id INT,
origin STRING,
destination STRING,
is_arrived BOOLEAN,
PRIMARY KEY (shipment_id) NOT ENFORCED
) WITH (
'connector' = 'postgres-cdc',
'hostname' = 'localhost',
'port' = '5432',
'username' = 'postgres',
'password' = 'postgres',
'database-name' = 'postgres',
'schema-name' = 'public',
'table-name' = 'shipments',
'slot.name' = 'flink'
);
```

Create the sink table for the enriched_orders table in Elasticsearch

```sql
CREATE TABLE enriched_orders (
order_id INT,
order_date TIMESTAMP(0),
customer_name STRING,
price DECIMAL(10, 5),
product_id INT,
order_status BOOLEAN,
product_name STRING,
product_description STRING,
shipment_id INT,
origin STRING,
destination STRING,
is_arrived BOOLEAN,
PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
'connector' = 'elasticsearch-7',
'hosts' = 'http://localhost:9200',
'index' = 'enriched_orders'
);
```

Insert some data into the enriched_orders table in Elasticsearch which is the result of joining the orders, products, and shipments tables which submits a flink job to join the tables

```sql
INSERT INTO enriched_orders
SELECT o.*, p.name, p.description, s.shipment_id, s.origin, s.destination, s.is_arrived
FROM orders AS o
LEFT JOIN products AS p ON o.product_id = p.id
LEFT JOIN shipments AS s ON o.order_id = s.order_id;
```

In order to see the changes in the enriched_orders table in Elasticsearch, you can update the orders table in MySQL and shipments table in PostgreSQL

```sql
INSERT INTO orders VALUES (default, '2020-07-30 15:22:00', 'Jark', 29.71, 104, false);
UPDATE orders SET order_status = true WHERE order_id = 10004;
```

```sql
INSERT INTO shipments VALUES (default,10004,'Shanghai','Beijing',false);
UPDATE shipments SET is_arrived = true WHERE shipment_id = 1004;
```

Stop the services

```shell
docker-compose down
```

Stop the local flink cluster

```shell

./bin/stop-cluster.sh
```