# Weather Data Analysis (OpenWeather API)

## Index

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Components](#components)
    - [1. Airflow DAGs](#1-airflow-dags)
        - [DAG 1: `openweather_api_dag`](#dag-1-openweather_api_dag)
        - [DAG 2: `transform_redshift_dag`](#dag-2-transform_redshift_dag)
    - [2. AWS Glue Script](#2-aws-glue-script)
    - [3. Buildspec File](#3-buildspec-file)
4. [Setup Instructions](#setup-instructions)

---

This project extracts weather data from the OpenWeather API, processes it using Apache Airflow, stores the data in Amazon S3, transforms the data using AWS Glue, and finally loads it into Amazon Redshift for analysis. The project is orchestrated using Airflow DAGs and involves several AWS services.

## Architecture

The architecture of this project is shown in the diagram below:

![Weather Data Analysis Architecture](https://github.com/jignesh-kachhad/weather-data-analysis/blob/main/Architecture%20.png)

## Components

### 1. Airflow DAGs

#### DAG 1: `openweather_api_dag`

This DAG is responsible for extracting weather data from the OpenWeather API, converting the JSON data to CSV, and uploading the CSV to an S3 bucket. It also triggers the second DAG for further processing.

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable
from datetime import datetime, timedelta
import json
from airflow.providers.amazon.aws.operators.s3 import S3CreateObjectOperator
import pandas as pd
import requests
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "start_date": datetime(2023, 8, 12),
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

dag = DAG(
    "openweather_api_dag",
    default_args=default_args,
    schedule_interval="@once",
    catchup=False, 
)

api_endpoint = "https://api.openweathermap.org/data/2.5/forecast"
api_params = {"q": "Toronto,Canada", "appid": Variable.get("api_key")}

def extract_openweather_data(**kwargs):
    print("Extracting started")
    ti = kwargs["ti"]
    response = requests.get(api_endpoint, params=api_params)
    data = response.json()
    print(data)
    df = pd.json_normalize(data["list"])
    print(df)
    ti.xcom_push(key="final_data", value=df.to_csv(index=False))

extract_api_data = PythonOperator(
    task_id="extract_api_data",
    python_callable=extract_openweather_data,
    provide_context=True,
    dag=dag,
)

upload_to_s3 = S3CreateObjectOperator(
    task_id="upload_to_S3",
    aws_conn_id="aws_default",
    s3_bucket="weather-data-gds1",
    s3_key="date={{ ds }}/weather_api_data.csv",
    data="{{ ti.xcom_pull(key='final_data') }}",
    dag=dag,
)

trigger_transform_redshift_dag = TriggerDagRunOperator(
    task_id="trigger_transform_redshift_dag",
    trigger_dag_id="transform_redshift_dag",
    dag=dag,
)

extract_api_data >> upload_to_s3 >> trigger_transform_redshift_dag
```

#### DAG 2: `transform_redshift_dag`

This DAG runs an AWS Glue job that transforms the CSV data and loads it into Amazon Redshift.

```python
from airflow import DAG
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.sensors.external_task import ExternalTaskSensor
from datetime import datetime, timedelta

default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "start_date": datetime(2023, 8, 12),
    "retries": 1,
    "retry_delay": timedelta(minutes=5),
}

dag = DAG(
    "transform_redshift_dag",
    default_args=default_args,
    schedule_interval="@once",
    catchup=False,
)

transform_task = GlueJobOperator(
    task_id="transform_task",
    job_name="glue_transform_task",
    script_location="s3://aws-glue-assets-126362963275-ap-south-1/scripts/weather_data_ingestion.py",
    s3_bucket="s3://aws-glue-assets-126362963275-ap-south-1",
    aws_conn_id="aws_default",
    region_name="ap-south-1",
    iam_role_name="glue-role",
    create_job_kwargs={
        "GlueVersion": "4.0",
        "NumberOfWorkers": 2,
        "WorkerType": "G.1X",
        "Connections": {"Connections": ["Redshift connection - 1"]},
    },
    dag=dag,
)
```

### 2. AWS Glue Script

This script is used by the Glue job to transform and load data into Amazon Redshift.

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
import datetime

args = getResolvedOptions(sys.argv, ["JOB_NAME"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

current_date = datetime.datetime.now().strftime("%Y-%m-%d")

weather_dyf = glueContext.create_dynamic_frame.from_options(
    format_options={"quoteChar": '"', "withHeader": True, "separator": ","},
    connection_type="s3",
    format="csv",
    connection_options={
        "paths": [f"s3://weather-data-gds1/date={current_date}/weather_api_data.csv"],
        "recurse": True,
    },
    transformation_ctx="weather_dyf",
)

changeschema_weather_dyf = ApplyMapping.apply(
    frame=weather_dyf,
    mappings=[
        ("dt", "string", "dt", "string"),
        ("weather", "string", "weather", "string"),
        ("visibility", "string", "visibility", "string"),
        ("`main.temp`", "string", "temp", "string"),
        ("`main.feels_like`", "string", "feels_like", "string"),
        ("`main.temp_min`", "string", "min_temp", "string"),
        ("`main.temp_max`", "string", "max_temp", "string"),
        ("`main.pressure`", "string", "pressure", "string"),
        ("`main.sea_level`", "string", "sea_level", "string"),
        ("`main.grnd_level`", "string", "ground_level", "string"),
        ("`main.humidity`", "string", "humidity", "string"),
        ("`wind.speed`", "string", "wind", "string"),
    ],
    transformation_ctx="changeschema_weather_dyf",
)

redshift_output = glueContext.write_dynamic_frame.from_options(
    frame=changeschema_weather_dyf,
    connection_type="redshift",
    connection_options={
        "redshiftTmpDir": "s3://aws-glue-assets-126362963275-ap-south-1/temporary/",
        "useConnectionProperties": "true",
        "aws_iam_role": "arn:aws:iam::126362963275:role/redshift-role",
        "dbtable": "public.weather_data",
        "connectionName": "Redshift connection - 1",
        "preactions": "DROP TABLE IF EXISTS public.weather_data; CREATE TABLE IF NOT EXISTS public.weather_data (dt VARCHAR, weather VARCHAR, visibility VARCHAR, temp VARCHAR, feels_like VARCHAR, min_temp VARCHAR, max_temp VARCHAR, pressure VARCHAR, sea_level VARCHAR, ground_level VARCHAR, humidity VARCHAR, wind VARCHAR);",
    },
    transformation_ctx="redshift_output",
)

job.commit()
```

### 3. Buildspec File

The buildspec file is used by AWS CodeBuild to automate the build and deployment process.

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo "Starting build process..."
  build:
    commands:
      - echo "Copying DAG files to S3..."
      - aws s3 cp --recursive ./dags s3://airflow-dag-bkt/dags/
      - echo "Copying requirements.txt files to S3..."
      - aws s3 cp ./requirements.txt s3://airflow-dag-bkt/
      - echo "Copying Glue scripts to S3..."
      - aws s3 cp --recursive ./scripts s3://aws-glue-assets-126362963275-ap-south-1/scripts/
  post_build:
    commands:
      - echo "Build and deployment process complete!!!"
```

## Setup Instructions

1. **Clone the repository:**
   ```sh
   git clone https://github.com/yourusername/weather-data-analysis.git
   cd weather-data-analysis
   ```

2. **Set up Airflow:**
   - Configure your Airflow connections for AWS.
   - Set the `api_key` variable in Airflow.

3. **Deploy the DAGs:**
   - Copy the DAG files to your Airflow DAGs directory or deploy them to your managed Airflow service.

4. **Configure AWS Glue:**
   - Ensure that the Glue job script is located in the specified S3 bucket.
   - Set up the required IAM roles and policies for Glue and Redshift access.

5. **Run the Build:**
   - Use AWS CodeBuild or a similar service to run the buildspec file, which will copy the necessary files to S3.

6. **Trigger the DAG:**
   - Trigger the `openweather_api_dag` from the Airflow UI to start the data pipeline.
