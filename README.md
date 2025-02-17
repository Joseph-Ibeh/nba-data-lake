# Sports Analytics Data Lake

This project sets up a data lake to store and process NBA sports data using AWS services such as S3, AWS Glue, and Athena. It uses the SportsData.io API to fetch NBA player data and process it for analytics purposes. There are two main scripts in the project:

1. **setup_nba_data_lake.py**: This script sets up the data lake by creating the necessary AWS resources and fetching NBA data from the SportsData.io API.
2. **delete.py**: This script deletes the resources created during the setup, including S3 buckets, Glue database, and Athena query results.

## Overview of AWS Services Used

- **S3 Buckets**: Used to store data in different formats (JSON files).
- **AWS Glue**: A serverless integration service to catalog the data stored in S3 and load it into Athena.
- **Amazon Athena**: Allows querying of the data stored in S3 using SQL queries.

## Interpretation of the Project

This project is designed to automate the setup of a sports analytics data lake for NBA data. By using AWS services, it allows users to:

- Collect NBA player data from the SportsData.io API.
- Store the data in an S3 bucket.
- Catalog and structure the data using AWS Glue.
- Set up Athena to allow querying of the data stored in S3.

This approach provides a scalable, cost-efficient solution for analyzing large amounts of sports data and performing advanced analytics using SQL-based queries.

---

## Steps to Set Up

### Prerequisites

Before running the script, ensure you have the following:

1. Go to [Sportsdata.io](https://sportsdata.io) and create a free account.
2. At the top left, you should see "Developers"; hover over it to see "API Resources".
3. Click on **Introduction & Testing**, then select "SportsDataIO API Free Trial" and fill out the information. Ensure to select NBA for this tutorial.
4. You will receive an email with the subject "Launch Developer Portal". Follow the link provided.
5. By default, it takes you to the NFL section. On the left, click on **NBA**.
6. Scroll down until you see **"Standings"**.
7. Under **"Query String Parameters"**, the value in the drop-down box is your API key.
8. Copy this string as you will need to paste it into the script.

### IAM Role/Permissions:
Ensure the user or role running the script has the following permissions:
- **S3**: `s3:CreateBucket`, `s3:PutObject`, `s3:DeleteBucket`, `s3:ListBucket`
- **Glue**: `glue:CreateDatabase`, `glue:CreateTable`, `glue:DeleteDatabase`, `glue:DeleteTable`
- **Athena**: `athena:StartQueryExecution`, `athena:GetQueryResults`

---

### Step 1: Open CloudShell Console

1. Go to [aws.amazon.com](https://aws.amazon.com) and sign into your account.
2. In the top right, next to the search bar, click the square with a `>_` inside to open the CloudShell.

![cloud shell](https://github.com/Joseph-Ibeh/nba-data-lake/blob/main/Assets/log%20in.png)


### Step 2: Create the `setup_nba_data_lake.py` File

1. In the CLI (Command Line Interface), type:

```
nano setup_nba_data_lake.py
```

2. Paste the following script into the file:

```
import boto3
import json
import time
import requests
from dotenv import load_dotenv
import os

# Load environment variables from .env file
load_dotenv()

# AWS configurations
region = "us-east-1"  # Replace with your preferred AWS region
bucket_name = "Jose-sports-analytics-data-lake"  # Change to a unique S3 bucket name
glue_database_name = "glue_nba_data_lake"
athena_output_location = f"s3://{bucket_name}/athena-results/"

# Sportsdata.io configurations (loaded from .env)
api_key = "SPORTS_DATA_API_KEY"  # Replace with your API key
nba_endpoint = "NBA_ENDPOINT"  # Get NBA endpoint from .env

# Use the hardcoded values from the script
api_key = SPORTS_DATA_API_KEY
nba_endpoint = NBA_ENDPOINT

# Create AWS clients
s3_client = boto3.client("s3", region_name=region)
glue_client = boto3.client("glue", region_name=region)
athena_client = boto3.client("athena", region_name=region)

def create_s3_bucket():
    """Create an S3 bucket for storing sports data."""
    try:
        if region == "us-east-1":
            s3_client.create_bucket(Bucket=bucket_name)
        else:
            s3_client.create_bucket(
                Bucket=bucket_name,
                CreateBucketConfiguration={"LocationConstraint": region},
            )
        print(f"S3 bucket '{bucket_name}' created successfully.")
    except Exception as e:
        print(f"Error creating S3 bucket: {e}")

def create_glue_database():
    """Create a Glue database for the data lake."""
    try:
        glue_client.create_database(
            DatabaseInput={
                "Name": glue_database_name,
                "Description": "Glue database for NBA sports analytics.",
            }
        )
        print(f"Glue database '{glue_database_name}' created successfully.")
    except Exception as e:
        print(f"Error creating Glue database: {e}")

def fetch_nba_data():
    """Fetch NBA player data from sportsdata.io."""
    try:
        headers = {"Ocp-Apim-Subscription-Key": api_key}
        response = requests.get(nba_endpoint, headers=headers)
        response.raise_for_status()  # Raise an error for bad status codes
        print("Fetched NBA data successfully.")
        return response.json()  # Return JSON response
    except Exception as e:
        print(f"Error fetching NBA data: {e}")
        return []

def convert_to_line_delimited_json(data):
    """Convert data to line-delimited JSON format."""
    print("Converting data to line-delimited JSON format...")
    return "\n".join([json.dumps(record) for record in data])

def upload_data_to_s3(data):
    """Upload NBA data to the S3 bucket."""
    try:
        # Convert data to line-delimited JSON
        line_delimited_data = convert_to_line_delimited_json(data)

        # Define S3 object key
        file_key = "raw-data/nba_player_data.jsonl"

        # Upload JSON data to S3
        s3_client.put_object(
            Bucket=bucket_name,
            Key=file_key,
            Body=line_delimited_data
        )
        print(f"Uploaded data to S3: {file_key}")
    except Exception as e:
        print(f"Error uploading data to S3: {e}")

def create_glue_table():
    """Create a Glue table for the data."""
    try:
        glue_client.create_table(
            DatabaseName=glue_database_name,
            TableInput={
                "Name": "nba_players",
                "StorageDescriptor": {
                    "Columns": [
                        {"Name": "PlayerID", "Type": "int"},
                        {"Name": "FirstName", "Type": "string"},
                        {"Name": "LastName", "Type": "string"},
                        {"Name": "Team", "Type": "string"},
                        {"Name": "Position", "Type": "string"},
                        {"Name": "Points", "Type": "int"}
                    ],
                    "Location": f"s3://{bucket_name}/raw-data/",
                    "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
                    "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
                    "SerdeInfo": {
                        "SerializationLibrary": "org.openx.data.jsonserde.JsonSerDe"
                    },
                },
                "TableType": "EXTERNAL_TABLE",
            },
        )
        print(f"Glue table 'nba_players' created successfully.")
    except Exception as e:
        print(f"Error creating Glue table: {e}")

def configure_athena():
    """Set up Athena output location."""
    try:
        athena_client.start_query_execution(
            QueryString="CREATE DATABASE IF NOT EXISTS nba_analytics",
            QueryExecutionContext={"Database": glue_database_name},
            ResultConfiguration={"OutputLocation": athena_output_location},
        )
        print("Athena output location configured successfully.")
    except Exception as e:
        print(f"Error configuring Athena: {e}")

# Main workflow
def main():
    print("Setting up data lake for NBA sports analytics...")
    create_s3_bucket()
    time.sleep(5)  # Ensure bucket creation propagates
    create_glue_database()
    nba_data = fetch_nba_data()
    if nba_data:  # Only proceed if data was fetched successfully
        upload_data_to_s3(nba_data)
    create_glue_table()
    configure_athena()
    print("Data lake setup complete.")

if __name__ == "__main__":
    main()
```

![python script](https://github.com/Joseph-Ibeh/nba-data-lake/blob/main/Assets/py-script.png)


### Step 3: Create the delete.py Script

```
import boto3
from botocore.exceptions import ClientError

# Define the names of resources to delete
BUCKET_NAME = "Jose-sports-analytics-data-lake"
GLUE_DATABASE_NAME = "glue_nba_data_lake"

def delete_athena_query_results(bucket_name):
    """Delete Athena query results stored in the specified S3 bucket."""
    s3 = boto3.client("s3")
    try:
        print(f"Deleting Athena query results in bucket: {bucket_name}")
        objects = s3.list_objects_v2(Bucket=bucket_name, Prefix="athena-results/")
        if "Contents" in objects:
            for obj in objects["Contents"]:
                s3.delete_object(Bucket=bucket_name, Key=obj["Key"])
                print(f"Deleted Athena query result: {obj['Key']}")
    except ClientError as e:
        print(f"Error deleting Athena query results in bucket {bucket_name}: {e}")

def delete_s3_bucket(bucket_name):
    """Delete a specific S3 bucket and its contents."""
    s3 = boto3.client("s3")
    try:
        print(f"Deleting bucket: {bucket_name}")
        # Delete all objects in the bucket
        objects = s3.list_objects_v2(Bucket=bucket_name)
        if "Contents" in objects:
            for obj in objects["Contents"]:
                s3.delete_object(Bucket=bucket_name, Key=obj["Key"])
                print(f"Deleted object: {obj['Key']}")
        # Delete the bucket
        s3.delete_bucket(Bucket=bucket_name)
        print(f"Deleted bucket: {bucket_name}")
    except ClientError as e:
        print(f"Error deleting bucket {bucket_name}: {e}")

def delete_glue_resources(database_name):
    """Delete Glue database and associated tables."""
    glue = boto3.client("glue")
    try:
        print(f"Deleting Glue database: {database_name}")
        # Get tables in the database
        tables = glue.get_tables(DatabaseName=database_name)["TableList"]
        for table in tables:
            table_name = table["Name"]
            print(f"Deleting Glue table: {table_name} in database {database_name}")
            glue.delete_table(DatabaseName=database_name, Name=table_name)
        # Delete the database
        glue.delete_database(Name=database_name)
        print(f"Deleted Glue database: {database_name}")
    except ClientError as e:
        print(f"Error deleting Glue resources for database {database_name}: {e}")

def main():
    print("Deleting resources created during data lake setup...")
    # Delete the S3 bucket
    delete_s3_bucket(BUCKET_NAME)
    # Delete Glue resources
    delete_glue_resources(GLUE_DATABASE_NAME)
    # Delete Athena query results
    delete_athena_query_results(BUCKET_NAME)
    print("All specified resources deleted successfully.")

if __name__ == "__main__":
    main()
```
![delete script](https://github.com/Joseph-Ibeh/nba-data-lake/blob/main/Assets/delete_py.png)


**ls to see see the files and directories that are present in the current directory or the directory**

![delete script](https://github.com/Joseph-Ibeh/nba-data-lake/blob/main/Assets/ls.png)

# Running the Scripts 

```
python3 setup_nba_data_lake.py
```

![run s](https://github.com/Joseph-Ibeh/nba-data-lake/blob/main/Assets/ran%20setup%20script.png)

*N.b ignore the errors, it was because I ran the script twice*
 

# Go to s3 bucket and open the Json file to get the data #

![s3](https://github.com/Joseph-Ibeh/nba-data-lake/blob/main/Assets/resources%20created.png)


![json](https://github.com/Joseph-Ibeh/nba-data-lake/blob/main/Assets/json%20file.png)


To delete the resources, run:

```
python3 delete.py
```

 **Challenges** 

Hardcoding API Key and Endpoint:
Issue: Initially, I faced challenges with loading the API key and endpoint properly.

**Solution:** I hardcoded these values temporarily into the script to ensure everything was working.

Takeaway: It's important to test environment variable loading and configurations before running the script.

**Conclusion**
This project demonstrates how to set up a data lake using AWS services and integrate data from an external API for sports analytics.
