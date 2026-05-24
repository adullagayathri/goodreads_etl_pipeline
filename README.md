# GoodReads Data Pipeline

<img src="https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/goodreads.png" align="center">

## Architecture 
![Pipeline Architecture](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/architecture.png)

The pipeline consists of various modules:

 - [GoodReads Python Wrapper](https://github.com/adullagayathri/goodreads)
 - ETL Jobs
 - Redshift Warehouse Module
 - Analytics Module 

#### Overview
Data is captured in real time from the GoodReads API using a Python wrapper (View usage - [Fetch Data Module](https://github.com/adullagayathri/goodreads/blob/master/example/fetchdata.py)). The data collected from the API is stored on local disk and moved to the Landing Bucket on AWS S3. ETL jobs are written in Spark and scheduled in Airflow to run every 10 minutes.

### ETL Flow

 - Data collected from the API is moved to landing zone S3 buckets.
 - The ETL job includes an S3 module which copies data from the landing zone to the working zone.
 - Once the data is moved to the working zone, a Spark job is triggered which reads the data and applies transformations. The dataset is repartitioned and moved to the Processed Zone.
 - The warehouse module of the ETL jobs picks up data from the processed zone and stages it into Redshift staging tables.
 - Using the Redshift staging tables, an UPSERT operation is performed on the Data Warehouse tables to update the dataset.
 - ETL job execution is completed once the Data Warehouse is updated. 
 - An Airflow DAG runs data quality checks on all Warehouse tables once the ETL job execution is completed.
 - The Airflow DAG has analytics queries configured in a Custom Designed Operator. These queries are run and a Data Quality Check is performed on selected Analytics Tables.
 - DAG execution completes after these Data Quality checks.

## Environment Setup

### Hardware Configuration
EMR: The pipeline utilizes a 3 node cluster with the following instance types:

    m5.xlarge
    4 vCore, 16 GiB memory, EBS only storage
    EBS Storage: 64 GiB

Redshift: The setup uses a 2 node cluster with instance types "dc2.large".

### Setting Up Airflow

Detailed instructions on how to setup Airflow using an AWS CloudFormation script are provided here: [Airflow using AWS CloudFormation](https://github.com/adullagayathri/Data_Engineering_Projects/blob/master/Airflow_Livy_Setup_CloudFormation.md)

**NOTE: This setup uses EC2 instances and a Postgres RDS instance. Ensure you review the associated costs before running the CloudFormation Stack.** 

The project uses "sshtunnel" to submit Spark jobs using an SSH connection from the EC2 instance. This setup does not automatically install "sshtunnel" for Apache Airflow. You can install it by running the following command: 

    pip install apache-airflow[sshtunnel]

Finally, copy the dag and plugin folders to the EC2 instance inside the Airflow home directory. Also, refer to [Airflow Connection](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/Airflow_Connections.md) for setting up connections to EMR and Redshift from Airflow.

### Setting up EMR
Spinning up an EMR cluster is straightforward. You can follow the AWS Guide available [here](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-gs.html).

ETL jobs in this project use [psycopg2](https://pypi.org/project/psycopg2/) to connect to the Redshift cluster for staging and warehouse queries. 
To install psycopg2 on EMR:

    sudo pip-3.6 install psycopg2

psycopg2 requires "postgresql-devel" and "postgresql-libs". If the installation fails, run:

    sudo yum install postgresql-libs
    sudo yum install postgresql-devel

ETL jobs also use [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) to move files between S3 buckets. To install boto3:

    pip-3.6 install boto3 --user

Finally, PySpark uses Python 2 as the default setup on EMR. To switch to Python 3, set the following environment variables:

    export PYSPARK_DRIVER_PYTHON=python3
    export PYSPARK_PYTHON=python3

Copy the ETL scripts to EMR to prepare the environment for job execution.

### Setting up Redshift
You can follow the AWS Guide to run a Redshift cluster or use the [Redshift_Cluster_IaC.py](https://github.com/adullagayathri/Data_Engineering_Projects/blob/master/Redshift_Cluster_IaC.py) script to create the cluster automatically. 

## How to Run 
Ensure the Airflow webserver and scheduler are running. 
Open the Airflow UI at http://< ec2-instance-ip >:< configured-port >

GoodReads Pipeline DAG
![Pipeline DAG](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/goodreads_dag.PNG)

DAG View:
![DAG View](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/DAG.PNG)

DAG Tree View:
![DAG Tree](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/DAG_tree_view.PNG)

DAG Gantt View: 
![DAG Gantt View](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/DAG_Gantt.PNG)

## Testing the Limits
The "goodreadsfaker" module in this project generates synthetic data used to test the ETL pipeline under heavy load. 

To test the pipeline, "goodreadsfaker" was used to generate 11.4 GB of data to be processed every 10 minutes (including ETL jobs, warehouse population, and analytical queries). This equates to approximately 68 GB/hour and 1.6 TB/day.

Source DataSet Count:
![Source Dataset Count](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/DatasetCount.PNG)

DAG Run Results:
![GoodReads DAG Run](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/DAG_tree_view.PNG)

Data Loaded to Warehouse:
![GoodReads Warehouse Count](https://github.com/adullagayathri/goodreads_etl_pipeline/blob/master/docs/images/WarehouseCount.PNG)

## Scenarios

- Data increase by 100x (Read vs Write):
    - Redshift: As an analytical database optimized for aggregation, it maintains high performance for read-heavy workloads.
    - Scalability: Increase the EMR cluster size to handle larger volumes of data processing.

- Pipelines running on a daily schedule (e.g., 7 AM):
    - The DAG is currently scheduled to run every 10 minutes but can be reconfigured for any specific interval.
    - Data quality operators ensure integrity. Email triggers can be configured to alert the team in case of failures.
    
- Availability for 100+ users:
    - Concurrency limits can be managed within the Amazon Redshift cluster. While the default is 50 parallel queries, additional clusters can be launched to meet business demand.

## Maintainer
Gayathri Adulla is an AI Engineer and MLOps specialist focused on building scalable data infrastructure and production AI systems. 

- GitHub: https://github.com/adullagayathri
- LinkedIn: https://www.linkedin.com/in/gayathri-adulla-924056216/
- Email: gayathri.adulla@outlook.com