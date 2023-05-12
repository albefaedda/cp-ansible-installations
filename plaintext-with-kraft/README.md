# Installation of Confluent Platform 7.4.0 with KRaft

I have extracted few variables using placeholders and have an external file where these can be filled with the right value and the file can be encrypted with `ansible-vault`, for example.

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

We're now ready to run the deployment of the Confluent Platform: 

```sh
ansible-playbook -e @secrets_file.enc --vault-password-file password_file -i hosts.yml confluent.platform.all
```

## Access Confluent Control Center

Use your C3 host followed by port 9021 to access it from the browser `http://<control-center-url>:9021`


## Verify that the platform is up and running from Confluent Control Center

From the control center UI move between the different tabs and verify that the different components are in the correct state. 
You can also view the topics, Producers/Consumers throughput and more.



# Test

## Create topic

```sh
kafka-topics --create --topic my-first-topic --bootstrap-server <kafka-broker-dns-name>:9092
```

## Produce messages

```sh
kafka-console-producer --topic my-first-topic --bootstrap-server <kafka-broker-dns-name>:9092
```

## Consume messages

```sh
kafka-console-consumer --topic my-first-topic --bootstrap-server <kafka-broker-dns-name>:9092 --from-beginning
```

## Create a Topic for Avro messages

```sh
kafka-topics --create --topic test-avro-topic --bootstrap-server <kafka-broker-dns-name>:9092
```

## Produce with Avro

```sh
kafka-avro-console-producer --bootstrap-server <kafka-broker-dns-name>:9092 \
  --topic test-avro-topic \
  --property schema.registry.url=http://<sr-dns-name>:8081 \
  --property value.schema="$(< schema/order-schema.json)" 
 <write your messages here>
```
 ## Consume with Avro

```sh
 kafka-avro-console-consumer --bootstrap-server <kafka-broker-dns-name>:9092 \
  --topic test-avro-topic --from-beginning \
  --property schema.registry.url=http://<sr-dns-name>:8081 \
  --property value.schema="$(< schema/order-schema.json)"
```
