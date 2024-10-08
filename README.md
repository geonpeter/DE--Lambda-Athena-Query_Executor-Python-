﻿# DE--Lambda-Athena-Query_Executor-Python-
This AWS Lambda function connects to Amazon Athena, executes a SQL query, and retrieves the results from S3.
 ```sql
import boto3
import time
import json

output_loc = "s3://geons-bucket/output/output_athena/"
database = "my-db"
query = "SELECT * FROM fam_table;"

def lambda_handler(event, context):
    client = boto3.client('athena')

    # Start the Athena query execution
    response = client.start_query_execution(
        QueryString=query,
        QueryExecutionContext={'Database': database},
        ResultConfiguration={'OutputLocation': output_loc}
    )
    
    # Get the query execution ID
    query_exec_id = response['QueryExecutionId']
    
    # Initialize query status
    query_status = None
    
    # Poll until query is finished
    while True:
        query_status = client.get_query_execution(QueryExecutionId=query_exec_id)['QueryExecution']['Status']['State']
        
        if query_status in ['SUCCEEDED', 'FAILED', 'CANCELLED']:
            break  # Exit the loop when query finishes
        
        time.sleep(2)  # Wait for 2 seconds before checking again
    
    # If query succeeded, get results
    if query_status == "SUCCEEDED":
        result = client.get_query_results(QueryExecutionId=query_exec_id)
        print(result)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'QueryResults': result['ResultSet']
            })
        }
    else:
        # If query failed, return an error message
        return {
            'statusCode': 400,
            'body': json.dumps(f"Query Failed with status {query_status}")
        }

```

## Features
- Connects to Amazon Athena using the boto3 SDK.
- Executes a SQL query on a specified Athena database.
- Polls Athena for query completion using an Execution ID.
- Retrieves and returns query results upon success.
- Returns error status in case of query failure.

## Pre-requesties
- AWS Lambda configured with the necessary IAM permissions:
- AmazonAthenaFullAccess
- AmazonS3ReadOnlyAccess (or write access to the specified S3 bucket).
- boto3 Python library.

## Configuration
Update the following variables in the code:

- output_loc: S3 location where Athena will store query results.
- database: The name of the Athena database.
- query: The SQL query to execute.

## Usage
- Clone the repository.
- Update the output_loc, database, and query variables with your specific details.
- Deploy the Lambda function to AWS.
- Invoke the function with the following format:
```sql
{
  "event": {}
}

```
## How the Athena Query Execution Works

1 Starting an Athena Query Execution
The Athena query is started using the start_query_execution function provided by boto3's Athena client. This function accepts:
- QueryString: The SQL query to execute.
- QueryExecutionContext: Specifies the Athena database.
- ResultConfiguration: The S3 bucket location where query results will be stored.

```sql
response = client.start_query_execution(
    QueryString=query,
    QueryExecutionContext={'Database': database},
    ResultConfiguration={'OutputLocation': output_loc}
)

```
2 Query Execution ID
When an Athena query is started, a unique Query Execution ID is generated. This ID is essential because Athena runs queries asynchronously, so the ID helps track the status of the query (whether it's running, succeeded, or failed). The ID is returned as part of the response to start_query_execution.
```sql
query_exec_id = response['QueryExecutionId']

```

3 Polling Query Status Using while Loop
Since Athena queries run asynchronously, we need to check the status of the query periodically until it either completes (successfully or with failure) or gets canceled. This is why the while loop is used to poll Athena until the query status becomes SUCCEEDED, FAILED, or CANCELLED.
- The while loop fetches the status using get_query_execution and checks the State of the query.
- It sleeps for 2 seconds between checks to avoid excessive API calls.

```sql
while True:
    query_status = client.get_query_execution(QueryExecutionId=query_exec_id)['QueryExecution']['Status']['State']
    
    if query_status in ['SUCCEEDED', 'FAILED', 'CANCELLED']:
        break  # Exit the loop when query finishes
    
    time.sleep(2)  # Wait for 2 seconds before checking again

```

4 Fetching Query Results
Once the query status changes to "SUCCEEDED", we can retrieve the query results using get_query_results. The results are stored in the S3 location specified earlier, and the function returns them as part of the Lambda response.

```sql
if query_status == "SUCCEEDED":
    result = client.get_query_results(QueryExecutionId=query_exec_id)
    return {
        'statusCode': 200,
        'body': json.dumps({
            'QueryResults': result['ResultSet']
        })
    }

```
If the query fails or is canceled, an error message is returned.


## Error Handling
The function will return a 400 status code with an error message if the query fails. Otherwise, it will return the query results in the response body with a 200 status code.
