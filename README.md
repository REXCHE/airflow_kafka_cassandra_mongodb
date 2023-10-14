# Overview

![image](https://github.com/dogukannulu/airflow_kafka_cassandra_mongodb/assets/91257958/b5ffd185-e046-43cc-ace6-cb7c4069d95f)

link to article First part
https://medium.com/@dogukannulu/data-engineering-end-to-end-project-part-1-airflow-kafka-cassandra-mongodb-docker-a87f2daec55e


Tech Stack
Apache Airflow
Apache Kafka
Cassandra
MongoDB
Docker
Apache Zookeeper
EmailOperator
SlackWebhookOperator
Overview
In this article, we are going to create a data pipeline. The whole pipeline will be orchestrated by Airflow. We are going to first create a Kafka topic if it does not exist. After creating it, we will produce messages that include e-mail and OTP (one-time password) as records. This part will illustrate streaming data coming to the Kafka topic.

While the data is being produced into the Kafka topic, we are going to consume it as well. We will obtain the data from the Kafka topic and will insert them both into the Cassandra table and MongoDB collection. We are going to check if the correct data exists in those. If so, we will send an e-mail to the incoming e-mail address and a Slack message including the e-mail address and OTP.

We can think of this project as a real-life e-mail validation. Let’s say there are streaming records that come to the Kafka topic including e-mail and OTP data. We will illustrate that part with the Kafka producer. Kafka consumer and data check parts will help us detect if the e-mail and OTP already exist or not.


# we have to run the docker file and docker-compose.yaml


The additional services include:

Kafka
Zookeeper
MongoDB
Cassandra
Kafka UI
Mongo Express
Please don’t forget to add the external network cassandra-kafka for all the services. If it doesn’t exist yet, you may create it with:

docker network create cassandra-kafka
Once we add the new services and parameters to the default Airflow docker-compose, we can run our containers.

docker compose up airflow-init
This command will initiate Airflow first. Then, we will run the below command to run all the services.

docker compose up -d --build
This command will build the container upon the Dockerfile and start all other services. If we follow the above instructions, we will have a directory called dags. We should locate all our scripts under the dags directory including the DAG script itself. All the scripts that I will explain in this article will be used as Airflow DAG tasks.


#Create Kafka Topic from /scripts/kafka_create_topic.py
The first thing we have to do is create a new Kafka topic. If the topic already exists, the script will also return the result accordingly.

after creating the topic'''

We have used the bootstrap servers according to the ones defined in the docker-compose file. We can define the client.id however we want. The script will return “Exists” if the topic already exists and “Created” if the topic has just been created. We will use this information while creating BranchPythonOperator in the second part. We can define the replication factor as 3 since we have 3 Kafka brokers as containers.

We are going to create two DummyOperators depending on the result of this task soon while creating the Airflow DAG. Our topic’s name will be email_topic. We can check if the topic exists or not via Kafka UI.



#Kafka Producer /scripts/kafka_producer.py
For this project, we want to illustrate streaming data coming to our Kafka topic. That’s why we have to create a Kafka producer as well.

The script will produce messages to the email_topic which include sample_email@my_email.com as key and 1234567 as value. The value will be the one-time password and the key will be the e-mail addresses coming to the Kafka topic. This process will go on for 20 seconds. We can modify the time period part according to our use case.

We can also manually check if the data is produced to the email_topic via Kafka UI.


Kafka Consumer for Cassandra /scripts/kafka_consumer_cassandra.py
Up until this point, we have created a Kafka topic and produced messages to the email_topic. From now on, we have to consume the messages coming to email_topic. This will be in two parts, the first part is for Cassandra and the second part is for MongoDB. In this section, I will explain the one for Cassandra.

After importing all the necessary libraries, we have to connect to Cassandra and execute the necessary commands.

This class will be used to connect to the Cassandra server first. Then, it will create a keyspace named email_namespace and a table named email_table. After obtaining the messages coming to the Kafka topic, it will insert it into the newly created table.


The above function will consume all the incoming messages for a predefined time period (30 seconds for this case) and populate the corresponding Cassandra table with them. If the data already exists in the table, it will skip those and will log them on the Logs section of Airflow.


We will use this function for our Airflow task. This basically combines all the methods we have created so far. It will create the keyspace and table after connecting to the Cassandra server. After consuming the messages coming to the email_topic, it will insert the non-existing ones into the Cassandra table.

We can manually check the data's existence by running the following commands in order.

docker exec -it cassandra /bin/bash/
cqlsh -u cassandra -p cassandra
select * from email_namespace.email_table;


#Kafka Consumer for MongoDB /scripts/kafka_consumer_mongodb.py
In this section, I will explain how to connect to MongoDB and insert the incoming messages into the corresponding collection. After importing all the necessary libraries, we have to connect to Cassandra and execute the necessary commands.

We don’t need to explicitly create a new database since it is created on the fly for MongoDB (this part is different than Cassandra).

The above class will consume all the incoming messages for a predefined time period (30 seconds for this case) and populate the corresponding MongoDB collection with them. If the data already exists in the table, it will skip those.

We will use this function for our Airflow task. This basically combines all the methods and classes we have created so far. It will create the collection after connecting to the MongoDB server. After consuming the messages coming to email_topic, it will insert the non-existing ones into the MongoDB collection.
We can check the data’s existence manually via Mongo Express.




# part 2
https://medium.com/@dogukannulu/data-engineering-end-to-end-project-part-2-airflow-kafka-cassandra-mongodb-docker-52a2ec7113de

#Check Cassandra Data /scripts/check_cassandra.py  class CassandraConnector:
In this section, we will check the specific e-mail address’ existence in the Cassandra table.

We will connect to Cassandra and select from the corresponding table. If the related data exists in the table, we will create a dictionary with that data. If not, we will return an empty dictionary. Creating the dictionary even if it is empty is necessary because if it is empty, the EmailOperator task will fail. That will help us detect if the correct data exists or not and we will use the result of this script for the EmailOperator.

We will use the function (def check_cassandra_main())as our Airflow task. We will define the specific e-mail address and check the data’s existence.


#Check MongoDB Data /scripts/check_mongodb.py def check_mongodb_main():
In this section, we will check the specific e-mail address’ existence in the MongoDB collection.


We will connect to MongoDB and select from the corresponding collection. If the related data exists in the collection, we will create a dictionary with that data. If not, we will return an empty dictionary. Creating the dictionary even if it is empty is necessary because if it is empty, the EmailOperator task will fail. That will help us detect if the correct data exists or not and we will use the result of this script for the EmailOperator.

We will use the above function as our Airflow task. We will define the specific e-mail address and check the data’s existence.

mongodb_uri is defined depending on the configuration in the Docker container.



# EmailOperator
In this part, I will explain how to get ready to create the task with the EmailOperator. I won’t separate it into two parts for Cassandra and MongoDB since they will be the same when it comes to the preparation.

We are going to use the Gmail SMTP server to send an e-mail. Remember that we have added some parameters to our docker-compose file during the first part of the article. You can see them below:

AIRFLOW__SMTP__SMTP_HOST: 'smtp.gmail.com'
AIRFLOW__SMTP__SMTP_MAIL_FROM: 'sample_email@my_email.com'
AIRFLOW__SMTP__SMTP_USER: 'sample_email@my_email.com'
AIRFLOW__SMTP__SMTP_PASSWORD: 'your_password'
AIRFLOW__SMTP__SMTP_PORT: '587'
The first thing we have to do is replace sample_email@my_email.com with the desired e-mail address that we want to send the e-mails from. For the password part, we have to create a new app password that will allow us to send e-mails. For that, we have to go to this link and create the password. Once we create it, we can replace the your_password part with the one we obtained.

Once we get all of them ready, we can then run our docker-compose file. This will allow us to send e-mails in the end. I will explain how to create the EmailOperator task in the Airflow DAG section below.
Once successful, we will get the e-mail below.


#SlackWebhookOperator
For this project, we want to send a Slack message that contains e-mail and OTP information in the message body. I will explain how to send that message in this part without separating it into two parts for Cassandra and MongoDB since they are pretty similar to each other.

First of all, we have to go to this link to create a Slack Webhook token.

We should first click on the “Create App” button
It will redirect us to another page. On that page, we should log in to the account that we want to send the message to.
Once we are logged in, we should create a dedicated workspace.
Inside the workspace, we should also create a dedicated channel so that it will help us see the incoming messages better.
While creating the webhook, we should create a bot to send the messages to the dedicated channel. I named my bot Airflow Slack Webhook.
Once we create the token, it should look like this: https://hooks.slack.com/services/T…
To be able to send the messages to Slack, we have to go to Airflow UI with username airflow and password airflow. From the Admin -> Connections section, we have to create a Slack connection. We should define the parameters as below:

Connection ID: We can name our connection however we want.
Connection Type: Slack Incoming Webhook
Slack Webhook Endpoint: https://hooks.slack.com/services/
Webhook Token: This will come from the webhook token we obtained recently. We should take the part that comes after the endpoint part. It generally starts with T.



After defining all, we should now create the connection. After creating the connection, we can now use SlackWebhookOperator successfully. It will get the token from the connections and use it accordingly to send messages to the dedicated workspace and channel.
Once successful, we will get the Slack message below.

#Airflow DAG

Here comes the last part of the project. We are going to combine all the parts we have created so far. First of all, we have to import the necessary operators. These include:

DummyOperator
PythonOperator
BranchPythonOperator
EmailOperator
SlackWebhookOperator


Once created, we can define email and OTP data coming from the data check scripts. These will be used for EmailOperator and SlackWebhookoperator.

We should also create a new function decide_branch. This will be used for PythonBranchOperator depending on the result it creates. We are going to connect this to two separate dummy operators.

This DAG will be running daily, but we can change schedule_interval depending on our use case.



#EmailOperator:

to: The e-mail address that is obtained from the NoSQL database. We are going to send our e-mail to this address. It will be sent from the e-mail address we defined in the docker-compose file
subject: The e-mail subject
html_content: The message body that we will include in the e-mail

#SlackWebhookOperator:

slack_webhook_conn_id: The connection ID that we defined during the creation of the new Slack connection.
message: The message we want to send to Slack. We can include emojis or define the message like a body which will include separate sections in the message itself.
channel: We should define the channel we want the message to be sent
username: The message will be sent with the name of the bot we created. If we don’t use a bot, we can use the username parameter as well.
Below is what our DAG will look like once we complete working on it. You can check it out via the Airflow UI.

