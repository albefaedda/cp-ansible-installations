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
	user_id varchar(100) NOT NULL,
	name varchar(100) NULL,
	address varchar(100) NULL,
	date_of_birth date NULL,
	email varchar(100) NOT NULL,
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

## Ansible

Our Ansible installation script contains different sections: 

- General section for connection to Confluent Cloud: 

```yml
    ansible_connection: "ssh"
    ansible_user: "ubuntu"
    ansible_become: "true"
    ansible_ssh_private_key_file: "{{ SSH_PRIVATE_KEY }}"
    jmxexporter_enabled: "true"
    jolokia_enabled: "true"

    ccloud_kafka_enabled: "true"
    ccloud_kafka_bootstrap_servers: "{{ BROKER_URL }}"
    ccloud_kafka_key: "{{ API_KEY }}"
    ccloud_kafka_secret: "{{ API_SECRET }}"
    ccloud_schema_registry_enabled: "true"
    ccloud_schema_registry_url: "{{ SR_URL }}"
    ccloud_schema_registry_key: "{{ SR_API_KEY }}"
    ccloud_schema_registry_secret: "{{ SR_API_SECRET }}"
```

- Kafka Connect plugins and hosts: 

```yml
kafka_connect:
  vars:
    kafka_connect_plugins_path:
      - /usr/share/java

    kafka_connect_confluent_hub_plugins:
      - mongodb/kafka-connect-mongodb:1.9.1
      - debezium/debezium-connector-sqlserver:2.0.1

  hosts:
    <connect-host-1>:
    <connect-host-2>:
```

- Control Center host: 

```yml
control_center:
  hosts:
    <control-center-host>:
```

- Kafka Connect Workers Override properties:

```yml
    kafka_connect_custom_properties:
      group.id: connect-cluster-on-prem
      
      replication.factor: 3
      topic.creation.enable: true

      # Confluent Cloud Connection to Kafka Cluster

      sasl.mechanism: PLAIN
      security.protocol: SASL_SSL
      bootstrap.servers: "{{ BROKER_URL }}"
      sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="{{ API_KEY }}" password="{{ API_SECRET }}";

      # Configuration for embedded producer
      
      producer.security.protocol: SASL_SSL
      producer.sasl.mechanism: PLAIN
      producer.sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="{{ API_KEY }}" password="{{ API_SECRET }}";
      
      # Configuration for embedded consumer
      
      consumer.sasl.mechanism: PLAIN
      consumer.security.protocol: SASL_SSL
      consumer.sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="{{ API_KEY }}" password="{{ API_SECRET }}";
      
      # Confluent Schema Registry for Kafka Connect
      
      key.converter: io.confluent.connect.avro.AvroConverter
      key.converter.basic.auth.credentials.source: USER_INFO
      key.converter.schema.registry.basic.auth.user.info: "{{ SR_API_KEY }}:{{ SR_API_SECRET }}"
      key.converter.schema.registry.url: "{{ SR_URL }}"
      
      value.converter: io.confluent.connect.avro.AvroConverter
      value.converter.basic.auth.credentials.source: USER_INFO
      value.converter.schema.registry.basic.auth.user.info: "{{ SR_API_KEY }}:{{ SR_API_SECRET }}"
      value.converter.schema.registry.url: "{{ SR_URL }}"
```

- Configurations for the Connect plugins:

    This line opens the connect plugins list: 

    ```yml
    kafka_connect_connectors:
    ```

    - MongoDB Source Connector

    ```yml
      - name: mongodb-orders-source-connector
        config:
          connector.class: "com.mongodb.kafka.connect.MongoSourceConnector"
          tasks.max: "1"
          connection.uri: "mongodb://{{ MONGODB_USER }}:{{ MONGODB_PASSWORD }}@{{ MONGODB_CONN_STRING }}"
          database: "{{ MONGODB_NAME }}"
          topic.prefix: "mongo.src"
          collection: "orders"
          startup.mode: "copy_existing"
          output.format.value: "schema"
          topic.creation.default.enable: "true"
          topic.creation.default.replication.factor: "3"
          topic.creation.default.partitions: "1"
          errors.log.enable: "true"
          errors.log.include.messages: "true"
    ```

    - SQL Server Source Connector

    ```yml
      - name: sql-server-users-source-connector
        config: 
          connector.class: "io.debezium.connector.sqlserver.SqlServerConnector"
          tasks.max: "1"
          database.hostname: "{{ SQL_SERVER_ENDPOINT }}"
          database.server.name: "{{ SQL_SERVER_ENDPOINT }}"
          database.port: "{{ SQL_SERVER_CONN_PORT }}"
          database.user: "{{ SQL_SERVER_USER }}"
          database.password: "{{ SQL_SERVER_PASSWORD }}"
          database.dbname: "{{ SQL_SERVER_DB_NAME }}"
          database.encrypt: "false"
          database.history.kafka.bootstrap.servers: "{{ BROKER_URL }}"
          database.history.consumer.sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="{{ API_KEY }}" password="{{ API_SECRET }}";
          database.history.consumer.sasl.mechanism: "PLAIN"
          database.history.consumer.security.protocol: "SASL_SSL"
          database.history.consumer.ssl.endpoint.identification.algorithm: "https"
          database.history.producer.sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="{{ API_KEY }}" password="{{ API_SECRET }}";
          database.history.producer.sasl.mechanism: "PLAIN"
          database.history.producer.security.protocol: "SASL_SSL"
          database.history.skip.unparseable.ddl: "true"
          decimal.handling.mode: "precise"
          errors.log.enable: "true"
          errors.log.include.messages: "true"
          errors.tolerance: "all"
          event.processing.failure.handling.mode": "fail"
          table.include.list: "{{ SQL_SERVER_SCHEMA_NAME }}.users"
          database.history.kafka.topic: "sql.src.history"
          max.batch.size: "1000"
          poll.interval.ms: "1000"
          provide.transaction.metadata: "false"
          snapshot.isolation.mode: "repeatable_read"
          snapshot.mode: "initial"
          time.precision.mode: "adaptive"
          tombstones.on.delete: "true"
          topic.prefix: "sql.src"
          topic.creation.default.enable: "true"
          topic.creation.default.replication.factor: "3"
          topic.creation.default.partitions: "1"
          transforms: "CastMoney"
          transforms.CastMoney.type: "org.apache.kafka.connect.transforms.Cast$Value"
          transforms.CastMoney.spec: "credits:double"
    ```

I have extracted few variables using placeholders and have an external file where these can be filled with the right value and the file can be encrypted with `ansible-vault`, for example.

Here's a template for the file: 

```properties
SSH_PRIVATE_KEY: ~/path/to-my/aws-key.pem

BROKER_URL: <kafka-bootstrap>.aws.confluent.cloud:9092
API_KEY: <kafka-api-key>
API_SECRET: <kafka-api-secret>

SR_URL: https://<sr-name>.aws.confluent.cloud
SR_API_KEY: <sr-api-key>
SR_API_SECRET: <sr-api-secret>

MONGODB_CONN_STRING: <your-ec2-hostname>.compute.amazonaws.com:27017
MONGODB_NAME: my-mongo-db
MONGODB_USER: mongo-user
MONGODB_PASSWORD: 4n07h3r_p4$$w0rD

SQL_SERVER_ENDPOINT: <rds-db-instance>.<region>.rds.amazonaws.com
SQL_SERVER_CONN_PORT: 1433
SQL_SERVER_DB_NAME: my-db-name
SQL_SERVER_SCHEMA_NAME: my-schema-name
SQL_SERVER_USER: my-db-user
SQL_SERVER_PASSWORD: p4$$w0rD
```

You can use the following command to encrypt the file with ansible-vault (it'll ask for a password):

```sh
ansible-vault encrypt secrets_file.enc
```

Once you have all the variables set, then you can run a check to make sure ansible can connect to your VMs: 

```sh
ansible -i hosts.yml all -m ping -e "SSH_PRIVATE_KEY=~/path/to-my/aws-key.pem"
```

If these are successful, then download the ansible collection for Confluent Platform: 

```sh
ansible-galaxy collection install confluent.platform:7.3.3
```

And then we're ready to run the deployment of the Connect Cluster and connector plugins: 

```sh
ansible-playbook -e @secrets_file.enc --vault-password-file password_file -i hosts.yml confluent.platform.all
```

To check the Connect and connectors logs, login on the VMs where the workers have been deployed, and run the following commands:

```sh
sudo su

cd /var/log/kafka

tail -1000f connect.log
```

You can also check the installed connectors and their status from the browser, using this url: 

```
http://<your-connect-worker-host>:8083/connectors

http://<your-connect-worker-host>:8083/connectors/<connector-name>/status
```

You can finally go to your Confluent Cloud UI and make sure you see the data produced in the topics - or use the confluent CLI to consume messages from the topic. 