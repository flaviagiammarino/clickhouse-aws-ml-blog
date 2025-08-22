# Zero-Shot Time Series Forecasting with Chronos using Amazon Bedrock and ClickHouse

<image src="https://clickhouse-aws-ml-blog.s3.eu-west-2.amazonaws.com/chronos_bedrock/chronos_bedrock_architecture_diagram.png" style="width:90%">
</image>

## Overview

The emergence of large language models (LLMs) with zero-shot generalization capabilities in sequence modelling 
tasks has led to the development of time series foundation models (TSFMs) based on LLM architectures. 
By converting time series into strings of digits, TSFMs can leverage LLMs' capability to extrapolate future 
patterns from the context data.
TSFMs eliminate the traditional need for domain-specific model development, allowing organizations to deploy 
accurate time series solutions faster.

In this post, we will focus on Chronos, a family of TSFMs for time series forecasting developed by Amazon. 
In contrast to other TSFMs, that rely on LLMs pre-trained on text, Chronos models are trained from scratch 
on a large collection of time series datasets.
Moreover, unlike other TSFMs, which require fine-tuning on in-domain data, Chronos models generate accurate 
zero-shot forecasts, without any task-specific adjustments.

Recently, the Chronos family of TSFMs has been extended with Chronos-Bolt, a faster, more accurate, and more 
memory-efficient Chronos model that can also be used on CPU. Chronos-Bolt is available in AutoGluon-TimeSeries,
Amazon SageMaker JumpStart and Amazon Bedrock.

In the rest of this post, we will walk through a practical example of using Chronos-Bolt with time series data 
stored in ClickHouse. We will create a Bedrock endpoint, then build a Lambda function that invokes the Bedrock 
endpoint with context data queried from ClickHouse and returns the Chronos-Bolt forecasts. 

## Solution
In this particular example, we will work with the 15-minute time series of the Italian electricity system's 
total demand, which we downloaded from [Terna's data portal](https://dati.terna.it/en/download-center#/load/total-load) 
and stored in a table in ClickHouse which we called `total_load_data`. However, given that Chronos-Bolt 
doesn't require any domain adaptation, the same solution can be applied to any other time series. 

**`total_load_data`**

<image src="https://clickhouse-aws-ml-blog.s3.eu-west-2.amazonaws.com/chronos_bedrock/total_load_data.png" style="width:50%">
</image>

**Note:** To be able to run the code below, you will need to have [Boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) 
and the [AWS-CLI](https://docs.aws.amazon.com/cli/latest/) installed on your machine. 
You will also need to update several variables in the code to reflect your AWS 
configuration - such as your AWS account number, region, service roles, etc. - as will be outlined below.

### 1. Create the Bedrock endpoint
We start by deploying Chronos-Bolt to a Bedrock endpoint hosted on a CPU EC2 instance. 
This can be done in Python using the code below, or directly from the Bedrock console. 

**Note:** If using the code below, make sure to replace the following variables: 

- `"<bedrock-marketplace-arn>"`: The Bedrock marketplace ARN of Chronos-Bolt (Base) model. 
- `"<bedrock-execution-role>"`: The Bedrock execution role ARN.

```python
import boto3

# Create the Bedrock client
bedrock_client = boto3.client("bedrock")

# Create the Bedrock endpoint
response = bedrock_client.create_marketplace_model_endpoint(
    modelSourceIdentifier="<bedrock-marketplace-arn>",  
    endpointConfig={
        "sageMaker": {
            "initialInstanceCount": 1,  
            "instanceType": "ml.m5.4xlarge",  
            "executionRole": "<bedrock-execution-role>" 
        }
    },
    endpointName="chronos-bolt-base-endpoint",
    acceptEula=True,
)

# Get the Bedrock endpoint ARN
bedrock_endpoint_arn = response["marketplaceModelEndpoint"]["endpointArn"]
```

### 2. Create the Lambda function for invoking the Bedrock endpoint with ClickHouse data
After that, we build a Lambda function for invoking the Bedrock endpoint with time series data stored in ClickHouse.
In order to create the Lambda function's Docker image in Elastic Container Registry (ECR), we need the following files: 
- `app.py`: The Python code of the Lambda function.
- `requirements.txt`: The list of dependencies that need to be installed in the Docker container.
- `Dockerfile`: The file containing the instructions to build the Docker image.

#### 2.1 Create the Docker image
##### 2.1.1 `app.py`

The Lambda function takes as input the following parameters: 
- `initialization_timestamp`: The first timestamp for which the forecasts should be generated.
- `frequency`: The frequency of the time series, in number of minutes.
- `context_length`: The number past time series values (prior to `initialization_timestamp`) to use as context.
- `prediction_length`: The number of future time series values (on and after `initialization_timestamp`) to forecast.

The Lambda function connects to ClickHouse using [ClickHouse Connect](https://clickhouse.com/docs/integrations/python) 
and loads the context data using the `query_df` method, which returns the query output in a Pandas Dataframe. 
After that, the Lambda function invokes the Bedrock endpoint with the context data. 
The Bedrock endpoint response includes the predicted mean and the predicted 10th, 50th (median) and 90th percentiles 
of the time series at each future time step, which the Lambda function returns to the user in JSON format 
together with the corresponding timestamps.

The Python code of the Lambda function is reported below. 

**Note:** Before deploying the Lambda function, make sure to replace the following variables: 

- `"<clickhouse-host>"`: The ClickHouse host. 
- `"<clickhouse-user>"`: The ClickHouse username. 
- `"<clickhouse-password>"`: The ClickHouse password. 
- `"<bedrock-endpoint-arn>"`: The Bedrock endpoint ARN. 

```python
import json
import boto3
import pandas as pd
import clickhouse_connect

def handler(event, context):
    """
    Generate zero-shot forecasts with Chronos-Bolt Amazon Bedrock endpoint using data stored in ClickHouse.

    Parameters:
    ========================================================================================================
    event: dict.
        A dictionary with the following keys:
        
        initialization_timestamp: str.
            The first timestamp for which the forecasts are generated, in ISO format (YYYY-MM-DD HH:mm:ss).
            
        frequency: int.
            The frequency of the time series, in minutes.
        
        context_length: int.
            The number of past time steps to use as context.
        
        prediction_length: int.
            The number of future time steps to forecast.

    context: AWS Lambda context object, see https://docs.aws.amazon.com/lambda/latest/dg/python-context.html.
    
    Returns:
    ========================================================================================================
    A JSON body with the following keys:
        
        timestamp: str.
            The future timestamp.
            
        mean: float:
            The predicted mean.
        
        0.1: float.
            The predicted 10th percentile.
        
        0.5: float.
            The predicted median.
            
        0.9: float.
            The predicted 90th percentile.
    """
    # Create the ClickHouse client
    clickhouse_client = clickhouse_connect.get_client(
        host="<clickhouse-host>",
        user="<clickhouse-user>",
        password="<clickhouse-password>",
        secure=True
    )
    
    # Create the Bedrock client
    bedrock_runtime_client = boto3.client(
        service_name="bedrock-runtime"
    )
    
    # Load the input data from ClickHouse
    df = clickhouse_client.query_df(
        f"""
            select
                timestamp,
                total_load
            from
                total_load_data
            where
                timestamp < toDateTime('{event['initialization_timestamp']}')
            and
                timestamp >= toDateTime('{event['initialization_timestamp']}') - INTERVAL {int(event['frequency']) * int(event['context_length'])} MINUTES
            order by
                timestamp asc
        """
    )
    print(f"""
        Loaded {len(df)} records from ClickHouse.
        Start timestamp: {min(df['timestamp'])}.
        End timestamp: {max(df['timestamp'])}.
    """)
    
    # Invoke the Bedrock endpoint with the ClickHouse data
    response = bedrock_runtime_client.invoke_model(
        modelId="<bedrock-endpoint-arn>",
        body=json.dumps({
            "inputs": [{
                "target": df["total_load"].values.tolist(),
            }],
            "parameters": {
                "prediction_length": event["prediction_length"],
                "quantile_levels": [0.1, 0.5, 0.9],
            }
        })
    )
    
    # Extract the forecasts
    predictions = json.loads(response["body"].read()).get("predictions")[0]
    
    # Add the timestamps to the forecasts
    predictions["timestamp"] = [
        x.strftime("%Y-%m-%d %H:%M:%S")
        for x in pd.date_range(
            start=event["initialization_timestamp"],
            periods=event["prediction_length"],
            freq=f"{event['frequency']}min",
        )
    ]
    
    print(f"""
        Received {len(predictions['mean'])} forecasts from Bedrock.
        Start timestamp: {min(predictions['timestamp'])}.
        End timestamp: {max(predictions['timestamp'])}.
    """)
    
    # Return the forecasts
    return {
        "statusCode": 200,
        "body": json.dumps(predictions)
    }
```

##### 2.1.2 `requirements.txt`

The `requirements.txt` file with the list of dependencies is reported below.

```
boto3==1.34.84
clickhouse_connect==0.8.18
pandas==2.3.1
```

##### 2.1.3 `Dockerfile`

The standard `Dockerfile` using the Python 3.12 AWS base image for Lambda is also reported below. 

```
FROM amazon/aws-lambda-python:3.12

COPY requirements.txt  .

RUN pip3 install -r requirements.txt --target "${LAMBDA_TASK_ROOT}"

COPY app.py ${LAMBDA_TASK_ROOT}

CMD ["app.handler"]
```

#### 2.2 Build the Docker image and push it to ECR

When all the files are ready, we can build the Docker image and push it to ECR 
with the AWS-CLI as shown in the `build_and_push.sh` script below.

**Note:** Before running the script, make sure to replace the following variables:  

- `"aws-account-id>"`: The AWS account number. 
- `"<ecr-repository-region>"`:  The region of the ECR repository. 
- `"<ecr-repository-name>"`: The name of the ECR repository. 


```commandline
aws_account_id="<aws-account-id>"
region="<ecr-repository-region>"
algorithm_name="<ecr-repository-name>"

aws ecr get-login-password --region $region | docker login --username AWS --password-stdin $aws_account_id.dkr.ecr.$region.amazonaws.com

docker build -t $algorithm_name .

docker tag $algorithm_name:latest $aws_account_id.dkr.ecr.$region.amazonaws.com/$algorithm_name:latest

docker push $aws_account_id.dkr.ecr.$region.amazonaws.com/$algorithm_name:latest
```

#### 2.3 Create the Lambda function

After the Docker image has been pushed to ECR, we can create the Lambda function using Boto3, the AWS-CLI or directly from the Lambda console.

### 3. Invoke the Lambda function and generate the forecasts

After the Lambda function has been created, we can invoke it to generate the forecasts.

The code below defines a Python function which invokes the Lambda function with the 
inputs discussed in the previous section and casts the Lambda function's JSON output to Pandas Dataframe.

Next, the code makes two invocations: the first time it requests the forecasts over a 
past time window for which historical data is already available, which allows us to assess how 
close the forecasts are to the actual data, while the second time it requests the forecasts 
over a future time window for which the data is not yet available. 

In both cases, the Lambda function is invoked with a context window of 3 weeks to generate 1-day-ahead forecasts.

```python
import io
import json
import boto3
import pandas as pd

def invoke_lambda_function(
    initialization_timestamp,
    frequency,
    context_length,
    prediction_length,
    function_name
):
    """
    Invoke the Lambda function that generates zero-shot forecasts with Chronos-Bolt
    Amazon Bedrock endpoint using the data stored in ClickHouse.
    
    Parameters:
    ========================================================================================================
    initialization_timestamp: str.
        The first timestamp for which the forecasts are generated, in ISO format (YYYY-MM-DD HH:mm:ss).
        
    frequency: int.
        The frequency of the time series, in minutes.
    
    context_length: int.
        The number of past time steps to use as context.
    
    prediction_length: int.
        The number of future time steps to forecast.
    
    function_name: str.
        The name of the Lambda function.
    
    Returns:
    ========================================================================================================
    forecasts: pd.DataFrame.
    
        A data frame with the following columns:
        
        timestamp: str.
            The future timestamp.
            
        mean: float:
            The predicted mean.
        
        0.1: float.
            The predicted 10th percentile.
        
        0.5: float.
            The predicted median.
            
        0.9: float.
            The predicted 90th percentile.
    """
    # Create the Lambda client
    lambda_client = boto3.client("lambda")
    
    # Invoke the Lambda function
    response = lambda_client.invoke(
        FunctionName=function_name,
        Payload=json.dumps({
            "initialization_timestamp": initialization_timestamp,
            "frequency": frequency,
            "prediction_length": prediction_length,
            "context_length": context_length
        })
    )
    
    # Extract the forecasts in a data frame
    forecasts = pd.read_json(io.StringIO(json.loads(response["Payload"].read())["body"]))
    
    # Return the forecasts
    return forecasts

# Define the Lambda function name and input parameters
frequency=15
context_length=24 * 4 * 7 * 3
prediction_length=24 * 4
function_name = "chronos-bedrock"

# Generate the forecasts over a past time window
predictions = invoke_lambda_function(
    initialization_timestamp="2025-08-17 00:00:00",
    frequency=frequency,
    context_length=context_length,
    prediction_length=prediction_length,
    function_name=function_name
)

# Generate the forecasts over a future time window
forecasts = invoke_lambda_function(
    initialization_timestamp="2025-08-18 00:00:00",
    frequency=frequency,
    context_length=context_length,
    prediction_length=prediction_length,
    function_name=function_name
)
```

**`predictions`**

<image src="https://clickhouse-aws-ml-blog.s3.eu-west-2.amazonaws.com/chronos_bedrock/chronos_bedrock_predictions_table.png" style="width:70%">
</image>

**`forecasts`**

<image src="https://clickhouse-aws-ml-blog.s3.eu-west-2.amazonaws.com/chronos_bedrock/chronos_bedrock_forecasts_table.png" style="width:70%">
</image>

### 4. Compare the forecasts to the historical data stored in ClickHouse
Now that the forecasts have been generated, we can compare them to the historical data stored in ClickHouse. 
We again use ClickHouse Connect to query the database and retrieve the results directly into a Pandas DataFrame. 

```python
import pandas as pd
import clickhouse_connect

# Create the ClickHouse client
clickhouse_client = clickhouse_connect.get_client(
    host="<clickhouse-host>",
    user="<clickhouse-user>",
    password="<clickhouse-password>",
    secure=True
)

# Load the historical data from ClickHouse
df = clickhouse_client.query_df(
    """
    select
        timestamp,
        total_load
    from
        total_load_data
    where
        timestamp >= toDateTime('2025-08-18 23:45:00') - INTERVAL 14 DAYS
    order by
        timestamp asc
    """
)

# Outer join the historical data with the model outputs 
output = pd.merge(
    left=df,
    right=pd.concat([predictions, forecasts], axis=0),
    on="timestamp",
    how="outer"
)
```

The results show that the forecasts are closely aligned with the actual data, 
demonstrating the model’s ability to generalize effectively in a zero-shot setting.
Despite a holiday occurring on the last Friday of the context window, 
the model produces accurate forecasts for the subsequent Sunday 
and correctly anticipates an increase in energy demand on the following Monday,
highlighting its strength in capturing complex temporal patterns. 

<image src="https://clickhouse-aws-ml-blog.s3.eu-west-2.amazonaws.com/chronos_bedrock/chronos_bedrock_zero_shot_forecasts.png" style="width:90%">
</image>
