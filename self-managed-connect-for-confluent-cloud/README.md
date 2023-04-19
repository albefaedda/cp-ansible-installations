# Self-managed connectors with cp-ansible playbook

In this module I'm going to deploy Confluent Platform Connect cluster with Ansible and create a data pipeline to ingest data to a Kafka topic from a MongoDB collection and then use a SQL Server Sink Connector to read the data from the Kafka topic and write it to SQL Server. 

## MongoDB - installation on AWS EC2

Follow [this guide](https://medium.com/@calvin.hsieh/steps-to-install-mongodb-on-aws-ec2-instance-62db66981218) to install a MongoDB instance on EC2:

## MongoDB - create DB

Login to Mongo from command line: 

```sh
mongo -u admin -p password admin
```

To list the databases available, run the command: 

```mongo
show dbs
```

To use a database, or create a new one if it doesn't exist, then run the command: 

```mongo
use ecommerce
```
This is going to create a new database called `ecommerce`

## MongoDB - create Collection and insert data

To create a new collection, or insert new data into a collection already existing, we need to run the following command: 

```mongo
db.orders.insert({"number":1,"shipping_address":"ABC Sesame Street,Wichita, KS. 12345","subtotal":110.00,"tax":10.00,"grand_total":120.00,"shipping_cost":0.00,"user_id": "u_1"})
```

`orders` is the name of the collection. 

Let's insert more data into the `orders` collection: 

```mongo
db.orders.insert({"number":2,"shipping_address":"123 Cross Street,Irving, CA. 12345","subtotal":5.00,"tax":0.53,"grand_total":6.53,"shipping_cost":1.00,"user_id": "u_2"})

db.orders.insert({"number":3,"shipping_address":"5014  Pinnickinick Street, Portland, WA. 97205","subtotal":93.45,"tax":9.34,"grand_total":102.79,"shipping_cost":0.00,"user_id": "u_3"})

db.orders.insert({"number":4,"shipping_address":"4082 Elmwood Avenue, Tempe, AX. 85281","subtotal":50.00,"tax":1.00,"grand_total":51.00,"shipping_cost":0.00,"user_id": "u_4"})

db.orders.insert({"number":5,"shipping_address":"123 Cross Street,Irving, CA. 12345","subtotal":33.00,"tax":3.33,"grand_total":38.33,"shipping_cost":2.00,"user_id": "u_2"})
```


## Sql Server - create Database Table and insert data

Before proceeding with creating a table, if you don't want to use the default database, create a new one from the UI.

Then open a SQL console and enable the database for CDC as we are using Debezium CDC Source Connector to ingest data into Kafka. 
You need a SQL Server Standard Edition (at least) to be able to enable CDC. 
 
```sql
EXEC sys.sp_cdc_enable_db;
```

Then create a table for our users:

```sql
CREATE TABLE ecommerce.dbo.users (
	user_id varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
	name varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
	address varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
	date_of_birth date NULL,
	email varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
	credits money NOT NULL,
	CONSTRAINT users_PK PRIMARY KEY (user_id)
);
```
And finally insert data to the table (the select statement at the end to ensure the data is inserted correctly):

```sql
INSERT INTO ecommerce.dbo.users
(user_id, name, address, date_of_birth, email, credits)
VALUES('u_1', 'Pippo', 'ABC Sesame Street,Wichita, KS. 12345', '1975-08-21', 'pippo@gmail.com', 12.50);

INSERT INTO ecommerce.dbo.users
(user_id, name, address, date_of_birth, email, credits)
VALUES('u_2', 'Pluto', '123 Cross Street,Irving, CA. 12345', '1975-03-15', 'pluto@msn.com', 0.00);

INSERT INTO ecommerce.dbo.users
(user_id, name, address, date_of_birth, email, credits)
VALUES('u_3', 'Paperino', '5014  Pinnickinick Street, Portland, WA. 97205', '1991-07-06', 'paperino@outlook.com', 1.00);

INSERT INTO ecommerce.dbo.users
(user_id, name, address, date_of_birth, email, credits)
VALUES('u_4', 'Foo', '4082 Elmwood Avenue, Tempe, AX. 85281', '1991-01-02', 'foo@email.com', 20.00);

INSERT INTO ecommerce.dbo.users
(user_id, name, address, date_of_birth, email, credits)
VALUES('u_5', 'Will', '79 Cilmaengwyn Road, Pontardawe, SA8 4QL', '1995-12-29', 'will@google.com', 76.00);

INSERT INTO ecommerce.dbo.users
(user_id, name, address, date_of_birth, email, credits)
VALUES('u_6', 'Tom', 'Yale Haven, Station Road, Church Village, CF38 1AF', '1985-09-22', 'tom@gmail.com', 15.50);

INSERT INTO ecommerce.dbo.users
(user_id, name, address, date_of_birth, email, credits)
VALUES('u_7', 'Jerry', 'The Sycamores, Maesbury Marsh, SY10 8JB', '1987-10-12', 'jerry@outlook.com', 30.75);

SELECT user_id, name, address, date_of_birth, email, credits
FROM ecommerce.dbo.users;

```
