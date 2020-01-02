# Configuration

CloudSeam consists of two deployable functions. Each function is configured independently, currently only through the use of environment variables.

- [Configuring the State Machine](#state-machine)
- [Configuring the Task Satisfier](#task-satisfier)



## State Machine

The state machine supports several areas of configuration. Currently, all configuration is made using only environment variables.


### Required Attributes

Each of the following attributes are required in order to run the application in production.

- `MACHINE_LOCATOR` - the strategy to locate and load machine specifications. See [Machine Location](#machine-location) below for more info
- `SQS_TASK_QUEUE_URL` - the SQS queue URL to be used for sending task events

#### Database Attributes

One of the following is **required** to be specified in order to persist state data.

- `POSTGRES_CONNECTION_SECRET` - the name of an AWS Secrets Manager secret containing connection details. See [Database Configuration](#database-configuration) below for more information.
- `DYNAMODB_TABLE` - the name of an existing DynamoDB table to store stacks. See additional details below regarding the structure of the table.


### Optional Attributes

Most of the attributes listed below are available, but typically used only during local development.

- `LOCAL_MODE` - configures the AWS client to send messages to ElasticMQ (http://sqs:9324) and use a local PostgreSQL database (bypassing `POSTGRES_CONNECTION_SECRET`)
- `SQS_EVENT_QUEUE_URL` - the SQS queue URL the state machine will be receiving events from. This is used during local development to poll and simulate Lambda function execution.



### Machine Location

Supported values for `MACHINE_LOCATOR` are `S3` and `LOCAL_FS`.



#### S3 Locator

Using the locator of `S3`, the following env variables are also required:

- `MACHINE_S3_BUCKET` - the name of the S3 bucket that contains the machine specifications
- `MACHINE_S3_KEY` - the name of the S3 key/prefix for machine specifications. If the specified key is a directory, all containing objects are checked out.


#### Local FS Locator

!!! warning
    The `LOCAL_FS` locator is mostly intended for local development/testing, rather than usage in production. This allows you to continue to use upstream releases without needing to bundle your own functions.

When using the local FS locator, the following env variables are also required:

- `MACHINE_SPEC_DIR` - an absolute path to the folder container spec files


### Database Configuration

Currently, state machine data persistence is supported in PostgreSQL and DynamoDB.

#### Persisting using DynamoDB

In order to use DynamoDB for persistence, the table is required to exist (the machine will not create the table). The following command will create the table. It is recommended to use "pay per use", as the needs will not be predictable and the low usage will keep costs low as is. The table **must** have a key attribute named `id`. 

```bash
aws dynamodb create-table \
    --table-name Stacks \
    --attribute-definitions AttributeName=id,AttributeType=S \
    --key-schema AttributeName=id,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```

Since the table is created outside of the app, the IAM role for the Lambda function needs only put and delete item support. The following policy will grant minimal access to the table.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PutUpdateDeleteOnStacks",
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:UpdateItem",
                "dynamodb:DeleteItem"
            ],
            "Resource": "your-dynamodb-table-arn"
        }
    ]
}
```



#### Persisting using PostgreSQL

To configure database connection details (host, username, password, and db), the state manager relies on AWS Secrets Manager. The name of the secret is made available through the `POSTGRES_CONNECTION_SECRET` environment variable. The secret must have the following structure:

```json
{
    "username": "database-username",
    "password": "secret-password",
    "host": "db.example.com",
    "dbname": "database-name",
    "port": 5432
}
```

!!! info
    The format above is the default format when creating a database secret using the Secrets Manager console.

 &nbsp;

---


## Task Satisfier

The state machine supports several areas of configuration. Currently, all configuration is made using only environment variables.


### Required Attributes

Each of the following attributes are required in order to run the application in production.

- `SQS_EVENT_QUEUE_URL` - the SQS queue URL to be used for sending task completion notices. Task completion errors will also be sent to the queue.



### Optional Attributes

Most of the attributes listed below are available, but typically used only during local development.

- `LOCAL_MODE` - configures the AWS client to send messages to ElasticMQ (http://sqs:9324) and use a local PostgreSQL database (bypassing `POSTGRES_CONNECTION_SECRET`)
- `SQS_TASK_QUEUE_URL` - the SQS queue URL the task satisfier will be receiving events from. This is used during local development to poll and simulate Lambda function execution.

