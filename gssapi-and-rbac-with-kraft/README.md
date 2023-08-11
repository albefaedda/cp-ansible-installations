# Installation of Confluent Platform 7.4.0 with KRaft, LDAP for Authentication and RBAC for Authorization

## Configuration

Follows a list of configuration that need to be taken into account, including ansible playbook placeholders, security certificates and keytabs. 

### Placeholders

I have extracted few variables using placeholders and have an external file where these can be filled with the right value and the file can be encrypted with `ansible-vault`.

Here's a template for the file: 

```properties
SSH_PRIVATE_KEY: ~/path/to-my/aws-key.pem
```

You can use the following command to encrypt the file with ansible-vault (it'll ask for a password):

```sh
ansible-vault encrypt secrets_file.enc
```

Once you have all the variables set, then you can run a check to make sure ansible can connect to your VMs: 

```sh
ansible -i hosts.yml all -m ping -e "SSH_PRIVATE_KEY=~/path/to-my/aws-key.pem"
```

### SSL Certificates

This installation expects setting up SSL certificates to the servers where the Confluent Platform components will be installed. For this to work, you need to create these certificates before running the deployment with Ansible. 
Ansible expects to find the keystore certificates in the path `/home/ubuntu/ssl/` with a name of `{{ inventory_hostname }}-keystore.jks` where `inventory_hostname` identifies the name of the host as specified in the cp-ansible hosts.yaml configuration file. 

The cp-ansible hosts.yaml also specifies mTLS for one of the Kafka listeners; for this reason you also need to create a truststore for your client authentication and this is expected at location `/home/ubuntu/ssl/kafka-truststore.jks`.

### Kerberos

We need to create Keytabs for our users; to do this we need to install Kerberos: 

```sh
sudo apt install krb5-user -y
```

This creates a configuration file in `/etc/krb5.conf` which includes: 

```conf
[libdefaults]

       default_realm = BOOTCAMP-REGION.CONFLUENT.IO

[realms]

       BOOTCAMP-AMERICAS.CONFLUENT.IO = {
                kdc = samba.bootcamp-americas.confluent.io:88
                admin_server = samba.bootcamp-americas.confluent.io:749
       }
```

We need to add the following properties under `libdefault`:

```conf
canonicalize = false  
allow_weak_crypto = true
```

We then need to generate the keytabs for the Confluent Platform hostnames and make sure the file names match with them; the template for file name (e.g. schemaregistry) looks like `/home/ubuntu/kerberos/schemaregistry-{{inventory_hostname_short}}.keytab` and the principal `schemaregistry/{{ inventory_hostname }}@{{ kerberos.realm }}`. 


## Deployment

We're now ready to run the deployment of the Confluent Platform: 

```sh
ansible-playbook -e @secrets_file.enc --vault-password-file password_file -i hosts.yml confluent.platform.all
```

### Access Confluent Control Center

Use your C3 host URL followed by port 9021 to access it from the browser `http://<control-center-url>:9021`; it'll ask you to login - you need to use an LDAP user.


### Verify that the platform is up and running from Confluent Control Center

From the control center UI move between the different tabs and verify that the different components are in the correct state. 
You can also view the topics, Producers/Consumers throughput and more.

## Test

You need to create a properties file (e.g. `client.properties`) containing the security certificates for the client to connect to the cluster as well as the authentication credentials for the user to perform its operations. 
An example of this file could be as follows: 

```properties
security.protocol=SSL
ssl.keystore.location=/path/to/kafka/kafka-keychain.jks
ssl.key.password=changeme
ssl.truststore.location=/path/to/kafka-truststore.jks

sasl.mechanism=PLAIN
security.protocol=SASL_SSL
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="alice" password="alice-secret";
```

### Create topic

```sh
kafka-topics --create --topic my-first-topic --bootstrap-server <kafka-broker-dns-name>:9092 \ 
    --command.config client.properties 
```

### Produce messages

```sh
kafka-console-producer --topic my-first-topic --bootstrap-server <kafka-broker-dns-name>:9092 \ 
    --producer.config client.properties
```

### Consume messages

```sh
kafka-console-consumer --topic my-first-topic --bootstrap-server <kafka-broker-dns-name>:9092 \ 
    --consumer.config client.properties --from-beginning
```

### Create a Topic for Avro messages

```sh
kafka-topics --create --topic test-avro-topic --bootstrap-server <kafka-broker-dns-name>:9092 \  
    --command.config client.properties 
```

To produce messages in Avro format, we need to use the kafka-avro-console-producer/consumer and need to provide Schema Registry certificate and authentication details as part of the command. 

### Produce with Avro

```sh
kafka-avro-console-producer --bootstrap-server <kafka-broker-dns-name>:9092 \
  --topic test-avro-topic \
  --property schema.registry.url=http://<sr-dns-name>:8081 \
  --property value.schema="$(< schema/order-schema.json)" \ 
  --property schema.registry.ssl.truststore.location=path/to/schema-registry/keystore.jks \
  --property basic.auth.credentials.source="USER_INFO" \ 
  --property basic.auth.user.info='sr-user:sr-pwd'
 <write your messages here>
```

 ### Consume with Avro

```sh
 kafka-avro-console-consumer --bootstrap-server <kafka-broker-dns-name>:9092 \
  --topic test-avro-topic --from-beginning \
  --property schema.registry.url=http://<sr-dns-name>:8081 \
  --property value.schema="$(< schema/order-schema.json)" \ 
  --property schema.registry.ssl.truststore.location=path/to/schema-registry/keystore.jks \
  --property basic.auth.credentials.source="USER_INFO" \ 
  --property basic.auth.user.info='sr-user:sr-pwd'
```

### Check registered subjects in SR: 

```sh
curl --user <schema-registry-api-key>:<schema-registry-api-secret> \
<schema-registry-url>/subjects
```

### Register a new schema in SR:

```sh
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\": \"string\", \"name\": \"string\", \"address\": \"string\", \"age\": \"int\"}"}' \
  http://<schema-registry-url>/subjects/my-topic-value/versions
  ```