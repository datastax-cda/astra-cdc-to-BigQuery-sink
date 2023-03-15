# astra-cdc-to-BigQuery-sink
This is a step by step guide to enable change data capture (CDC) on a table in Astra DB and then create a BigQuery sink in Astra Streaming to feed data from the table into Google BigQuery. This provides the ability to send data changes made in Astra DB to BigQuery for downstream analytical use cases.

## Prerequisites

- Create a free [Astra account](https://astra.datastax.com/)  
- Create an Astra database following these [instructions](https://awesome-astra.github.io/docs/pages/astra/cdc-for-astra/) 
  - Choose a region that is supports CDC for Astra DB by visiting the Streaming documentation's ["Regions" page](https://docs.datastax.com/en/streaming/astra-streaming/operations/astream-regions.html)
  - For our example we will be using the following database details:
    - Database Name: `my_company`
    - Region: `us-east1` in `GCP`
    - Keyspace Name: `our_product`
    - Provider and Region: `Google Cloud > North America > us-central1`
    
   
