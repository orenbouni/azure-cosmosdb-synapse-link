# azure-cosmosdb-synapse-link

Demonstration of **Azure CosmosDB** with **Azure Synapse Analytics**
integration via **Synapse Link**

**Chris Joakim, Microsoft, Global Black Belt NoSQL/CosmosDB**, chjoakim@microsoft.com

### Table of Contents

- [Part 1: Architecture of Synapse Link, and this Demonstration App](#part1)
- [Part 2: Setup this Demonstration App in Your Azure Subscription](#part2)
- [Part 3: Demonstration](#part3)
  - 3.1 Understand the International Air Travel Data
  - 3.2 Populate CosmosDB with the DotNet Console App
  - 3.3 Count the CosmosDB Documents with the DotNet Console App
  - 3.4 Query the CosmosDB Documents with the DotNet Console App
  - 3.5 Query the Synapse Link Data with a PySpark Notebook in Synapse

<p align="center"><img src="presentation/img/horizonal-line-1.jpeg" width="95%"></p>

<a name="part1"></a>

## Part 1: Architecture of Synapse Link, and this Demonstration App

- A **net5.0 client program** reads a data file, and Bulk Loads JSON documents to CosmosDB
- The CosmosDB documents flow into **Synapse Link** in near realtime
- Synapse Link performs **both copy AND data transformation (to columnar format)** operations
- No other ETL solution is needed (i.e. - Databricks)
- Query the Synapse Link data with **PySpark Notebooks in Azure Synapse Analytics**


<p align="center"><img src="presentation/img/csl-demo.png" width="100%"></p>

<p align="center"><img src="presentation/img/horizonal-line-1.jpeg" width="95%"></p>

## Synapse Link data movement and transformation

- Synapse Link performs **both copy AND data transformation (to columnar format)** operations
- A **columnar datastore** is more suitable for analytical query processing
- The **inserts, updates, and deletes** to your operational data are automatically synced to analytical store
- Auto-sync latency is usually within 2 minutes, but up to 5 minutes
- Supported for the **Azure Cosmos DB SQL (Core)** API and **Azure Cosmos DB API for MongoDB** APIs

<p align="center"><img src="presentation/img/transactional-analytical-data-stores.png" width="100%"></p>

<p align="center"><img src="presentation/img/horizonal-line-1.jpeg" width="95%"></p>

## Synapse Link Details

- **No impact to CosmosDB performance or RU costs**
- Is Scalable and Elastic
- The Synapse Link data can be queried in Azure Synapse Analytics by:
  - **Azure Synapse Spark pools**
    - Spark Streaming not yet supported
  - **Azure Synapse Serverless SQL pools** (not provisioned pools)
- Pricing consists of **storage and IO operations**
- Schema constraints:
  - Only the first 1000 document properties
  - Only the first 127 document nested levels
  - No explicit versioning, the schema is inferred
  - CosmosDB stores JSON
  - Attribute names are mormalized: {"id": 1, "Name": "fred", "name": "john"}
  - Addtibute names with odd characters: colons, semicolons, parens, =, etc
- Two Schema Types:
  - Well-defined 
    - Default option for SQL (CORE) API accounts
    - The schema, with datatypes, grows are documents are added
      - Non-conforming attributes are ignored
        - doc1: {"id": "1", "a":123}      <-- "a" is an integer, added to schema
        - doc2: {"id": "2", "a": "str"}   <-- "a" isn't an integer, ignored
  - Full Fidelity
    - Default option for Azure Cosmos DB API for MongoDB accounts
    - None of the above dataname normalization or datatype enforcement
    - Can be optionally be used by the SQL API
      - az cosmosdb create ... --analytical-storage-schema-type "FullFidelity" 

- See https://docs.microsoft.com/en-us/azure/cosmos-db/analytical-store-introduction

---

## Links / References

- [What is Azure Synapse Link for Azure Cosmos DB?](https://docs.microsoft.com/en-us/azure/cosmos-db/synapse-link)
- [Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction)
- [Azure Synapse Analytics](https://azure.microsoft.com/en-us/services/synapse-analytics/)
- [Analytical Store Pricing](https://docs.microsoft.com/en-us/azure/cosmos-db/analytical-store-introduction#analytical-store-pricing)


Go to [Part 3: Demonstration](#part3)

<p align="center"><img src="presentation/img/horizonal-line-1.jpeg" width="95%"></p>

<a name="part2"></a>

## Part 2: Setup this Demonstration App in Your Azure Subscription

### Laptop/Workstation/VM Requirements

- Either the Windows, Linux, or macOS operating system
- [git](https://git-scm.com/)
- [dotnet 5](https://dotnet.microsoft.com/download/dotnet/5.0)
- [az CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)  

### Clone this GitHub Repository

```
$ cd <some-parent-directory>

$ git clone https://github.com/cjoakim/azure-cosmosdb-synapse-link.git

$ cd azure-cosmosdb-synapse-link
```

#### Directory Structure of this Repository

```
├── DotnetConsoleApp      <-- net5.0 console application
│   ├── data              <-- json and csv files, zipped
│   └── sql               <-- CosmosDB query sql file(s)
├── az                    <-- provisioning scripts using the az CLI
├── presentation
│   └── presentation.md   <-- primary presentation file
└── synapse
    └── pyspark           <-- pyspark notebooks for Azure Synapse
```

### Compile Code, Unzip Data Files

```
$ cd DotnetConsoleApp
$ dotnet restore               <-- install the dotnet packages from NuGet (i.e. - CosmosDB SDK)
$ dotnet build                 <-- compile the C# code

$ mkdir out

$ cd data
... unzip the two zip files    <-- the zip files contain csv and json files too large for GitHub
$ cd ..

$ dotnet run                   <-- displays the list of commands supported by Program.cs
```

### Provision Azure Resources

It is recommended that you provision these Azure Resources with either the 
**Azure Portal** or the **az CLI**.  This repo contains working az CLI scripts.

- **Azure CosmosDB Account, SQL API**
  - database named **demo**
  - container named **travel** with partition key **/pk**
  - both the account and the container should have the **Analytical Store Enabled**

- **Azure Synapse**
  - with a **spark pool of three small nodes**

### Provision Resources with the az CLI

**If you provisioned resources in Azure Portal, you can skip this section.**

#### Setup the az CLI

```
$ cd az

$ az login 

$ az account set --subscription <your-subscription-id>

$ az account show

$ az extension add -n storage-preview
$ az extension add --name synapse
```

#### Initial Environment Varibles

Set these on your system as both the az CLI provisioning process,
and the DotNet program, will use them.

```
export AZURE_SUBSCRIPTION_ID=<your-specified-username>
export AZURE_SYNAPSE_USER=<your-specified-username>
export AZURE_SYNAPSE_PASS=<your-specified-password>
export AZURE_CSL_COSMOSDB_BULK_BATCH_SIZE=500
```

#### Provisioning on Linux or macOS with the az CLI in bash scripts

First, edit file **config.sh** - this file specifies your Azure Region,
Resource Group, and Azure Resource configuration details.

**Please do a change-all on this script to change "cjoakim" to YOUR ID!**

```
$ ./create_all.sh
```

#### Provisioning on Windows with the az CLI in PowerShell scripts

**Note: The PowerShell scripts for Windows in this repo are currently under construction.  The az commands, however, are portable across OS platforms.**

### Additional Environment Varibles

After provisioning, see your **CosmosDB account Azure Portal** to get these values.

```
export AZURE_CSL_COSMOSDB_SQLDB_URI
export AZURE_CSL_COSMOSDB_SQLDB_KEY
export AZURE_CSL_COSMOSDB_SQLDB_CONN_STRING
export AZURE_CSL_COSMOSDB_SQLDB_PREF_REGIONS=eastus   <-- example value
```

### Your CosmosDB Settings in Azure Portal

Your account should look similar to the following:

<p align="center"><img src="presentation/img/travel-container-settings.png" width="95%"></p>

Note: I set the Time To Live (TTL) on my container to 86,400 seconds.
This represents 24-hours, or 1-day (60 * 60 * 24).

### Configure Azure Synapse

- Create a Linked Service to the CosmosDB Synapse Link Data
- Right-mouse the CosmosDB Synapse Link Data ""travel" icon
- Create a Notebook PySpark Notebook that reads that data as a Dataframe
- Edit the cells of the Notebook to look like the following

```

TODO

```

<p align="center"><img src="presentation/img/horizonal-line-1.jpeg" width="95%"></p>

<a name="part3"></a>

## Part 3: Demonstration

### 3.1 Understand the International Air Travel Data

Each line in file data/air_travel_departures.json contains a document that looks
logically similar to the following:

```
{
  "id": "a7a868a4-ff6f-11eb-96e6-acde48001122",
  "pk": "GUM:MAJ",
  "date": "2006/05/01",
  "year": "2006",
  "month": "5",
  "from_iata": "GUM",
  "to_iata": "MAJ",
  "airlineid": "20177",
  "carrier": "PFQ",
  "count": "10",
  "route": "GUM:MAJ",
  "from_airport_name": "Guam Intl",
  "from_airport_tz": "Pacific/Guam",
  "from_location": {
    "type": "Point",
    "coordinates": [
      144.795983,
      13.48345
    ]
  },
  "to_airport_name": "Marshall Islands Intl",
  "to_airport_country": "Marshall Islands",
  "to_airport_tz": "Pacific/Majuro",
  "to_location": {
    "type": "Point",
    "coordinates": [
      171.272022,
      7.064758
    ]
  },
  "doc_epoch": 1629214058.4217112
}
```

### 3.2 Populate CosmosDB with the DotNet Console App

#### See the available commands for Program.cs

```
$ cd DotnetConsoleApp

$ dotnet run
...

Command-Line Examples:
dotnet run list_databases
dotnet run create_database <dbname> <shared-ru | 0>
dotnet run delete_database <dbname>
dotnet run update_database_throughput <dbname> <shared-ru>
---
dotnet run list_containers <dbname>
dotnet run create_container <dbname> <cname> <pk> <ru>
dotnet run update_container_throughput <dbname> <cname> <ru>
dotnet run update_container_indexing <dbname> <cname> <json-doc-infile>
dotnet run truncate_container <dbname> <cname>
dotnet run delete_container <dbname> <cname>
---
dotnet run bulk_load_container <dbname> <cname> <pk-attr> <json-rows-infile> <batch-count>
dotnet run bulk_load_container demo travel route data/air_travel_departures.json 1
---
dotnet run count_documents <dbname> <cname>
---
dotnet run execute_queries <dbname> <cname> <queries-file>
dotnet run delete_route <dbname> <cname> <route>
dotnet run delete_route demo travel CLT:MBJ
```

#### Populate the Database, using the DotNet SDK Bulk Loading functionality 

```
$ dotnet run bulk_load_container demo travel route data/air_travel_departures.json 100
...

{"id":"fffd3f5f-6aa6-468d-811d-24e0802f3054","pk":"JFK:PUJ","date":"2002/01/01","year":"2002","month":"1","from_iata":"JFK","to_iata":"PUJ","airlineid":"20402","carrier":"MMQ","count":"1","route":"JFK:PUJ","from_airport_name":"John F Kennedy Intl","from_airport_tz":"America/New_York","from_location":{"type":"Point","coordinates":[-73.778925,40.639751]},"to_airport_name":"Punta Cana Intl","to_airport_country":"Dominican Republic","to_airport_tz":"America/Santo_Domingo","to_location":{"type":"Point","coordinates":[-68.363431,18.567367]},"doc_epoch":1630355396413,"doc_time":"2021/08/30-20:29:56"}

writing batch 98 (500) at 1630355396414

{"id":"25e58a8d-b053-4849-ae9c-4324493fcddb","pk":"MIA:MAO","date":"2004/09/01","year":"2004","month":"9","from_iata":"MIA","to_iata":"MAO","airlineid":"20232","carrier":"A2","count":"1","route":"MIA:MAO","from_airport_name":"Miami Intl","from_airport_tz":"America/New_York","from_location":{"type":"Point","coordinates":[-80.290556,25.79325]},"to_airport_name":"Eduardo Gomes Intl","to_airport_country":"Brazil","to_airport_tz":"America/Boa_Vista","to_location":{"type":"Point","coordinates":[-60.049721,-3.038611]},"doc_epoch":1630355398116,"doc_time":"2021/08/30-20:29:58"}

writing batch 99 (500) at 1630355398117

{"id":"53bb2083-532d-47eb-8511-63fd5682f533","pk":"GUM:HND","date":"2005/08/01","year":"2005","month":"8","from_iata":"GUM","to_iata":"HND","airlineid":"20185","carrier":"JO","count":"58","route":"GUM:HND","from_airport_name":"Guam Intl","from_airport_tz":"Pacific/Guam","from_location":{"type":"Point","coordinates":[144.795983,13.48345]},"to_airport_name":"Tokyo Intl","to_airport_country":"Japan","to_airport_tz":"Asia/Tokyo","to_location":{"type":"Point","coordinates":[139.779694,35.552258]},"doc_epoch":1630355399824,"doc_time":"2021/08/30-20:29:59"}

writing batch 100 (500) at 1630355399824

EOJ Totals:
  Database:             demo
  Container:            travel
  Input Filename:       data/air_travel_departures.json
  Max Batch Count:      100
  BulkLoad startEpoch:  1630355232531
  BulkLoad finishEpoch: 1630355401536
  BulkLoad elapsedMs:   169005
  BulkLoad elapsedSec:  169.005
  BulkLoad elapsedMin:  2.81675
  Batch Size:           500
  Batch Count:          100
  Exceptions:           0
  Document/Task count:  50000
  Document per Second:  295.84923522972696
```

The above loads 100 batches (50,000 documents) into the database named demo, 
the container named travel, using the given json data file and the value of the
route attribute as the partition key.

The last document in each batch (of 500) is logged to the output, and then
end-of-job totals are displayed.

This load process can be run several times as necessary, and unique documents 
will be created from the same input data.  This is enabled by this C# code that 
sets the **id attribute** of each new document to a Guid:

```
    jsonDoc.id = Guid.NewGuid().ToString();  <-- See Program.cs, method BulkLoadContainer
```

Look at your CosmosDB account in Azure Portal to confirm that the documents were added.

<p align="center"><img src="presentation/img/documents-in-azure-portal.png" width="95%"></p>

### 3.3 Count the CosmosDB Documents with the DotNet Console App

```
$ dotnet run count_documents demo travel

CountDocuments demo travel -> 50000
```

### 3.4 Query the CosmosDB Documents with the DotNet Console App

```
$ rm out/q*.json   <-- remove the previous query response output files

$ dotnet run execute_queries demo travel sql/queries.txt

================================================================================
executing qname: q0, db: demo, cname: travel, sql: SELECT COUNT(1) FROM c
QueryResponse: q0 db: demo container: travel status: OK ru: 2.89 items: 1 excp: False
file written: out/q0_demo_travel.json

================================================================================
executing qname: q1, db: demo, cname: travel, sql: SELECT * FROM c WHERE c.pk = 'ATL:MBJ'
QueryResponse: q1 db: demo container: travel status: OK ru: 3.3 items: 13 excp: False
file written: out/q1_demo_travel.json

================================================================================
executing qname: q2, db: demo, cname: travel, sql: SELECT * FROM c WHERE c.pk = 'ATL:MBJ'
QueryResponse: q2 db: demo container: travel status: OK ru: 3.3 items: 13 excp: False
file written: out/q2_demo_travel.json

================================================================================
executing qname: q3, db: demo, cname: travel, sql: SELECT * FROM c WHERE c.pk = 'ATL:MBJ' offset 0 limit 5
QueryResponse: q3 db: demo container: travel status: OK ru: 2.99 items: 5 excp: False
file written: out/q3_demo_travel.json

================================================================================
executing qname: q4, db: demo, cname: travel, sql: SELECT * FROM c WHERE c.to_airport_country = 'Jamaica'
QueryResponse: q4 db: demo container: travel status: OK ru: 32.4 items: 773 excp: False
file written: out/q4_demo_travel.json
```

The console output shows the query, the RU charge, and the number of items (documents)
returned.  See the output/xxx.json file for the actual query results.

Edit file sql/queries.txt as necessary, to add your own queries.

### 3.5 Query the Synapse Link Data with a PySpark Notebook in Synapse

