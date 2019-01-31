The Greenplum Stream Server (GPSS) is an ETL (extract, transform, load) tool. An instance of the GPSS server ingests streaming data from one or more clients, using Greenplum Database readable external tables to transform and insert the data into a target Greenplum table. The data source and the format of the data are specific to the client.

The Greenplum Stream Server includes the gpss command-line utility. When you run gpss, you start an instance of GPSS; this instance waits indefinitely for client data.

The Greenplum Stream Server also includes the gpsscli command-line utility, a client tool for submitting data load jobs to a GPSS instance and managing those jobs.

Limitations
The Greenplum Stream Server does not support loading data from multiple Kafka topics to the same Greenplum Database table. All jobs will hang if GPSS encounters this situation.




# Preparations - 
## Master Side
	1. Prepare a database + table which will receive the kafka data
		```a. testdb=# CREATE TABLE json_from_kafka( customer_id int8, month int4, amount_paid decimal(9,2) );```
	2. Registering the GPSS Extension
	You must explicitly register the Greenplum Stream Server extension in each database in which you will use GPSS to write data to Greenplum tables.
		a. $ ssh gpadmin@gpmaster
		b. gpmaster$ . /usr/local/greenplum-db/greenplum_path.sh
		c. mdw$ psql -d testdb
		d. testdb=# CREATE EXTENSION gpss;
		e. Perform steps 3 and 4 for each database in which the Greenplum Stream Server will write client data.
	3. Configuring the Greenplum Stream Server over our Master
	Our stream server will combine within the same process both gpss listener + gpfdist
		a. Create "gpss_config.json" file that responsible to configure our GPSS service
		Default host address is - localhost
		Example:
		{
		    "ListenAddress": {
		        "Host": "",
		        "Port": 50007,
		        "SSL": false
		    },
		    "Gpfdist": {
		        "Host": "",
		        "Port": 8319
		    }
		}
		
		gpmaster$ gpss gpss_config.json --log-dir . &
		
		b. Now we have both 'gpfdist' listener to dispatch data to our segments nodes.
		+ 'gpss' listener to receive data from Kafka client node.
	4. Create 'jsonload_cfg.yaml' file
	In this file we configure our data source and where to put it, on which database and table.
	https://gpdb.docs.pivotal.io/5110/greenplum-kafka/load-json-example.html
		a. Example:
		DATABASE: from_kafka_db
		USER: gpadmin
		HOST: localhost
		PORT: 5432
		KAFKA:
		   INPUT:
		     SOURCE:
		        BROKERS: kafka-client:9092
		        TOPIC: topic_json_gpkafka
		     COLUMNS:
		        - NAME: jdata
		          TYPE: json
		     FORMAT: json
		     ERROR_LIMIT: 10
		   OUTPUT:
		     TABLE: json_from_kafka
		     MAPPING:
		        - NAME: customer_id
		          EXPRESSION: (jdata->>'cust_id')::int
		        - NAME: month
		          EXPRESSION: (jdata->>'month')::int
		        - NAME: amount_paid
		          EXPRESSION: (jdata->>'expenses')::decimal
		   COMMIT:
		     MAX_ROW: 1000
		
## Client Side
	1. Install Kafka
	In this tutorial we will use kafka as our message broker or ETL server
	That receive topics or data and send them over to our gpss listener at the master node.
	https://www.digitalocean.com/community/tutorials/how-to-install-apache-kafka-on-centos-7
	2. Make sure you have a routing between kafka server to gpmaster server on port 9092
	3. Create a Kafka topic
	Greenplum has a limitation of 1:1 topic to table. It cannot ingress several topics to 1 table.
	https://gpdb.docs.pivotal.io/5110/greenplum-kafka/load-json-example.html
	Example:
	kafkahost$ $KAFKA_INSTALL_DIR/bin/kafka-topics.sh --create \
	    --zookeeper localhost:2181 --replication-factor 1 --partitions 1 \
	    --topic topic_json_gpkafka
	4. Insert data to the new kafka topic
		a. kafkahost$ vi sample_data.json
		b. Copy and paste:
		{ "cust_id": 1313131, "month": 12, "expenses": 1313.13 }
		{ "cust_id": 3535353, "month": 11, "expenses": 761.35 }
		{ "cust_id": 7979797, "month": 10, "expenses": 4489.00 }
		{ "cust_id": 7979797, "month": 11, "expenses": 18.72 }
		{ "cust_id": 3535353, "month": 10, "expenses": 6001.94 }
		{ "cust_id": 7979797, "month": 12, "expenses": 173.18 }
		{ "cust_id": 1313131, "month": 10, "expenses": 492.83 }
		{ "cust_id": 3535353, "month": 12, "expenses": 81.12 }
		{ "cust_id": 1313131, "month": 11, "expenses": 368.27 }
	5. Stream the data into kafka topic we created
	kafkahost$ $KAFKA_INSTALL_DIR/bin/kafka-console-producer.sh \
	    --broker-list localhost:9092 \
	    --topic topic_json_gpkafka < sample_data.json
	6. Verify the data got inserted
	kafkahost$ $KAFKA_INSTALL_DIR/bin/kafka-console-consumer.sh \
	    --bootstrap-server localhost:9092 --topic topic_json_gpkafka \
	    --from-beginning

# Submit a job
## Over the master server - 
	gpmaster$ gpsscli submit --name kafkajson2gp --gpss-port 50007 ./jsonload_cfg.yaml
	List all the jobs
	gpmaster$ gpsscli list --all --gpss-port 50007
	Start the job
	gpmaster$ gpsscli start kafkajson2gp --gpss-port 50007
	To stop receiving and insert the new rows - stop the job
	gpmaster$ gpsscli stop kafkajson2gp --gpss-port 50007
	2. Examine the 'gpss' command output. It should look like that - 
	... -[INFO]:- ... Inserted 9 rows
	... -[INFO]:- ... Rejected 0 rows
	3. View the new content at the table we have created
	gpmaster$ psql -d testdb
	
	testdb=# SELECT * FROM json_from_kafka WHERE customer_id='1313131' 
	           ORDER BY amount_paid;
	 customer_id | month | amount_paid 
	-------------+-------+-------------
	     1313131 |    11 |      368.27
	     1313131 |    10 |      492.83
	     1313131 |    12 |     1313.13
	
		
		
