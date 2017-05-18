# DynamoDB-Cross-Region-Replication

## Using Lambda Function for Cross-region Replication
1. Create a table in Region A and Region B
```sh
aws dynamodb create-table --table-name tbA --attribute-definitions AttributeName=ID,AttributeType=S --key-schema AttributeName=ID,KeyType=HASH --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 --region us-east-1

aws dynamodb create-table --table-name tbB --attribute-definitions AttributeName=ID,AttributeType=S --key-schema AttributeName=ID,KeyType=HASH --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 --region ap-southeast-2
```

2. Create the lambda function
Use the following python code
```python
from __future__ import print_function

print('Loading function')

import json
import boto3
import time

DDB_TABLE = 'tbB'
REGION = 'ap-southeast-2'

ddb = boto3.client('dynamodb', region_name=REGION)


def lambda_handler(event, context):
    #print("Received event: " + json.dumps(event, indent=2))
    now = int(time.time() * 1000)

    for record in event['Records']:
        #print(record['eventID'])
        #print(record['eventName'])
        #print("DynamoDB Record: " + json.dumps(record['dynamodb'], indent=2))
        
        if record['eventName'] == 'INSERT' or record['eventName'] == 'MODIFY':
            ddb.put_item(TableName=DDB_TABLE, Item=record['dynamodb']['NewImage'])
        if record['eventName'] == 'REMOVE':
            ddb.delete_item(TableName=DDB_TABLE, Key=record['dynamodb']['Keys'])
    print('{{"BatchSize": {} }}'.format(len(event['Records'])))
        
    return 'Successfully processed {} records.'.format(len(event['Records']))
```

3. Configure the trigger
- Select Dynamodb
- Batch size set to 100 initially
- Starting position set to Trim horizon

4. PROBLEM: after a few successful batches, it started to get "Function call failed" on certain request.
    > Cause: Default timeout was 3 second, which means when a function running for over 3 seconds, the process is freezed. Check out [Understanding Container Reuse in AWS Lambda](https://aws.amazon.com/blogs/compute/container-reuse-in-lambda/).
    > Solution: increase timeout to 1 min

5.
