# astra-cdc-to-BigQuery-sink
This is a step by step guide to enable change data capture (CDC) on a table in Astra DB and then create a BigQuery sink in Astra Streaming to feed data from the table into Google BigQuery. This provides the ability to send data changes made in Astra DB to BigQuery, in near real-time, for downstream analytical use cases.

## Prerequisites

- Create a free [Astra account](https://astra.datastax.com/)  
- Create an Astra database following these [instructions](https://awesome-astra.github.io/docs/pages/astra/cdc-for-astra/) 
  - Choose a region that is supports CDC for Astra DB by visiting the Streaming documentation's ["Regions" page](https://docs.datastax.com/en/streaming/astra-streaming/operations/astream-regions.html)
  - For our example we will be using the following database details:
    - Database Name: `sandbox`
    - Region: `us-east1` in `GCP`
    - Keyspace Name: `sample`
    - Provider and Region: `Google Cloud > North America > us-east1`
- Access to a GCP project with BQ admin and project viewer permissions
	- And a GCP [Service Account](https://cloud.google.com/iam/docs/service-accounts-create) with BQ admin permissions - needed to authenticate using API keys
	- Create a JSON [key](https://cloud.google.com/iam/docs/keys-create-delete) for the service account.

<br>

## Create streaming tenant

1. Login to your [Astra account](https://astra.datastax.com/)

    ![image](https://user-images.githubusercontent.com/41307386/225459590-dc605fbb-3b87-4309-a95b-6c674fec664f.png)

2. From the Astra home page, choose "Create a Streaming Tenant" from the Quick Access section and create a new tenant with the following details:

    Tenant Name: `cdctest`   
    Provider and Region: `Google Cloud > useast1`

    ![image](https://user-images.githubusercontent.com/41307386/225459466-a4a310f3-9fd0-4bff-b455-265068f52c59.png)
    
    ![image](https://user-images.githubusercontent.com/41307386/225460342-0c6abcf9-f511-404b-a717-a6d488d45052.png)

    The new tenant will be ready very quickly and your view will automatically refresh to its “Quickstart” tab. CDC will automatically create a namespace and topic within the tenant.

<br>

## Create table and enable CDC

1. Navigate to the sandbox database and click on the CQL Console tab. 
    
    ![image](https://user-images.githubusercontent.com/41307386/225462035-ab3e95c4-7ed4-43a8-be82-98b9d65311ad.png)
    
2. Create a table that will hold the test account information

    ```
    create table sample.all_accounts (id uuid primary key, full_name text, email text);
    ```

    ![image](https://user-images.githubusercontent.com/41307386/225461916-e466a35d-2686-4884-a809-0ca3011c091e.png)
    
3. Navigate to the database’s "CDC" tab and choose "Enable CDC"

    ![image](https://user-images.githubusercontent.com/41307386/225462418-c19884cb-99d3-4768-a9e0-094d34989489.png)

4. Enable CDC with the following details:
  
    Tenant: `pulsar-gcp-useast1 / cdctest`
    
    Keyspace: `sample`
    
    Table name: `all_accounts`  

    ![image](https://user-images.githubusercontent.com/41307386/225462213-adf24397-a789-4155-977f-36413205d017.png)
    
5. Wait for the new CDC process to have a status of "Running"
  
    ![image](https://user-images.githubusercontent.com/41307386/225462888-4b3a5144-d686-4b52-915b-cb37a7535e73.png)

6. Add a new record to the all_accounts table, to initialize the streaming objects:
	```
	insert into sample.all_accounts (id, full_name, email) values (85540e16-aca8-11ec-b909-0242ac120002, 'Joe Smith', 'joesmith@domain.com');
	```

<br>

## Create BigQuery streaming sink

As of this posting, the BigQuery streaming sink is in an "experimental" state, meaning it hasn't been fully tested or certified. Currently, creating the sink using the UI does not result in a functioning sink - so the sink needs to be created with the pulsar-admin CLI.

Reference for utilizing the pulsar-admin CLI with Astra is found in this Astra Streaming Demo [repo](https://github.com/chrisjohnson16/astra-streaming-demo).  

Reference for the Pulsar BigQuery sink connector, which is used by Astra, is available in this [repo](https://github.com/datastax/pulsar-3rdparty-connector/tree/master/pulsar-connectors/bigquery). 

1. Create a [dataset](https://cloud.google.com/bigquery/docs/quickstarts/load-data-console#create_a_dataset) in BigQuery with a name of your choosing (for this example the dataset is named `astracdc_demo`) <br>
	![image](https://user-images.githubusercontent.com/41307386/229561017-27a8a689-ec90-45be-bd9b-8d6ba07c458f.png)

2. Create a Topic in the `astracdc` namespace of the streaming tenant for the offset storage: 
	- From the Streaming tenant navigate to: "Namespace and Topics"
	- Under the astracdc namespace click "Add Topic". 
	  ![image](https://user-images.githubusercontent.com/41307386/229567041-76642e1d-a656-4610-bbea-0896c4469345.png)
	- Give the topic a name and click "Add Topic". For this example, naming the topic bq-demo-offset-01
	  ![image](https://user-images.githubusercontent.com/41307386/229566495-54bc859b-e2a0-4fd5-8053-483ee713af49.png)
3. Prepare the BigQuery sink config file 
	- Refer to sample [bqdemoconfig.yaml](https://github.com/kaseycparker/astra-cdc-to-BigQuery-sink/blob/main/bqdemoconfig.yaml) 
		- Note: the keyfile in the example is redacted but is the JSON key downloaded for the GCP service account. All quotes `"` and backslashes `\` in the json key file need to be escaped when adding to the config yaml using the `\` escape character, i.e. `\"` and  `\\`
	```
	$ cat /tmp/bqdemoconfig.yaml 
	configs:
	  lingerTimeMs: "10"
	  topic: "cdctest/astracdc/data-342690fc-0ec9-4025-8c4e-d0966f71ecd1-sample.all_accounts"
	  sanitizeTopicName: "true"
	  kafkaConnectorConfigProperties:
			autoCreateTables: "true"
			bufferSize: "10"
			connector.class: "com.wepay.kafka.connect.bigquery.BigQuerySinkConnector"
			defaultDataset: "astracdc_demo"
			kafkaDataFieldName: "topicMetaData"
			keySource: "JSON"
			keyfile: "{\"type\": \"service_account\",\"project_id\": \"bq-test-382615\",\"private_key_id\": \"**redacted**\",\"private_key\": \"-----BEGIN PRIVATE KEY-----\\n**redacted**\\n-----END PRIVATE KEY-----\\n\",\"client_email\": \"**redacted\",\"client_id\": \"**redacted**\",\"auth_uri\": \"https://accounts.google.com/o/oauth2/auth\",\"token_uri\": \"https://oauth2.googleapis.com/token\",\"auth_provider_x509_cert_url\": \"https://www.googleapis.com/oauth2/v1/certs\",\"client_x509_cert_url\": \"https://www.googleapis.com/robot/v1/metadata/x509/redacted.iam.gserviceaccount.com\"}"
			name: "bq-demo"
			project: "bq-test-382615"
			queueSize: "100"
			sanitizeFieldNames: "true"
			sanitizeTopics: "false"
			tasks.max: "1"
			topics: "cdctest/astracdc/data-342690fc-0ec9-4025-8c4e-d0966f71ecd1-sample.all_accounts"
			kafkaKeyFieldName: "key"
			topic2TableMap: "persistent___cdctest_astracdc_data_342690fc_0ec9_4025_8c4e_d0966f71ecd1_sample_all_accounts_partition_0:all_accounts_partition_0,persistent___cdctest_astracdc_data_342690fc_0ec9_4025_8c4e_d0966f71ecd1_sample_all_accounts_partition_1:all_accounts_partition_1,persistent___cdctest_astracdc_data_342690fc_0ec9_4025_8c4e_d0966f71ecd1_sample_all_accounts_partition_2:all_accounts_partition_2"
			allowNewBigQueryFields: "true"
	  batchSize: "10"
	  offsetStorageTopic: "cdctest/astracdc/bq-demo-offset-01"
	  kafkaConnectorSinkClass: "com.wepay.kafka.connect.bigquery.BigQuerySinkConnector"
	```

	- Config Properties of note:
		- `sanitizeTopicName: "true"` - required value
		- `defaultDataset:`  - specifies the dataset to use in BQ - must already exist
		- `kafkaDataFieldName: "topicMetaData"` - required value
		- `name:` - name of the sink
		- `project:` - your GCP BigQuery project
		- `sanitizeFieldNames: "true"` - required value
		- `sanitizeTopics: "false"` - required value
		- `topics:` - CDC data topic from Astra Streaming for appropriate table(s)
		- `kafkaKeyFieldName:` - use to sink the CDC table key field into BiqQuery
		- `topic2TableMap:` - use to rename BQ tables instead of using topic name - one table per partition is created
		
3. Create the BigQuery sink using the pulsar-admin

	```
	pulsar-admin sinks create -t bigquery --processing-guarantees EFFECTIVELY_ONCE --inputs cdctest/astracdc/data-342690fc-0ec9-4025-8c4e-d0966f71ecd1-sample.all_accounts --sink-config-file /tmp/bqconfig.yaml --tenant cdctest --namespace astracdc --name bq-test2
	```

4. Verify the sink exists and is running in the Astra Console. 
	- Navigate to the streaming tenant in the console and click on sinks
	- Sink should initially show an "Initializing" status and then turn to a "Running" status
	![image](https://user-images.githubusercontent.com/41307386/229632403-f7ae1857-66cd-4047-96bb-e1ee899abba3.png)
	![image](https://user-images.githubusercontent.com/41307386/229632881-59d9a15c-5cbb-4e45-ab7a-7487c3bda78d.png)

	- Troubleshooting tip: If the sink Errors instead of moving to a Running state - click into the sink from the console and view/download the log and look for errors to triage the issue. Correct the the config issue, delete the sink and recreate as needed.
	![image](https://user-images.githubusercontent.com/41307386/229544861-b1fe9779-6591-478b-96e7-33663f7caca3.png)

### Test the CDC to BigQuery sink
1. Insert a new record into the all_accounts table and verify it flows to BigQuery:
    ```
    insert into sample.all_accounts (id, full_name, email) values (uuid(), 'Champ Ship', 'champ.ship@acoolplace.com');
    ```
	
2. Confirm in BigQuery that the table was created (if not already existing) and that the record was inserted.
	![image](https://user-images.githubusercontent.com/41307386/229635134-4fd5a24a-3df5-4f64-b9f0-734ca5e9fff7.png)
	- Note: 
		- One BigQuery per partition is created. Per functionality as of this writing, this cannot be changed. 
		- The table name can be controlled by mapping the full partition name to a table name of your choosing, using the `topic2TableMap:` config property. See the sample bqdemoconfig.yaml for an example. 
