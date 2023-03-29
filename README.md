# astra-cdc-to-BigQuery-sink
This is a step by step guide to enable change data capture (CDC) on a table in Astra DB and then create a BigQuery sink in Astra Streaming to feed data from the table into Google BigQuery. This provides the ability to send data changes made in Astra DB to BigQuery for downstream analytical use cases.

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
    
## Create streaming tenant

1. Login to your [Astra account](https://astra.datastax.com/)

    ![image](https://user-images.githubusercontent.com/41307386/225459590-dc605fbb-3b87-4309-a95b-6c674fec664f.png)

2. From the Astra home page, choose "Create a Streaming Tenant" from the Quick Access section and create a new tenant with the following details:

    Tenant Name: `cdctest`   
    Provider and Region: `Google Cloud > uscentral1`

    ![image](https://user-images.githubusercontent.com/41307386/225459466-a4a310f3-9fd0-4bff-b455-265068f52c59.png)
    
    ![image](https://user-images.githubusercontent.com/41307386/225460342-0c6abcf9-f511-404b-a717-a6d488d45052.png)

    The new tenant will be ready very quickly and your view will automatically refresh to its “Quickstart” tab. CDC will automatically create a namespace and topic within the tenant.
    
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


## Create BigQuery streaming sink

As of this posting, the BigQuery streaming sink is in an "experimental" state, meaning it hasn't been fully tested or certified. Currently, creating the sink using the UI does not result in a functioning sink - so the sink needs to be created with the pulsar-admin CLI.

Reference for utilizing the pulsar-admin CLI with Astra is found in this Astra Streaming Demo [repo](https://github.com/chrisjohnson16/astra-streaming-demo).  

Reference for the Pulsar BigQuery sink connector, which is used by Astra, is available in this [repo](https://github.com/datastax/pulsar-3rdparty-connector/tree/master/pulsar-connectors/bigquery). 
