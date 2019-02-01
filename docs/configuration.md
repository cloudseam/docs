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
- `POSTGRES_CONNECTION_SECRET` - the name of an AWS Secrets Manager secret containing connection details. See [Database Configuration](#database-configuration) below for more information.


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

Currently, state machine persistence is only accomplished by storing the data into a PostgreSQL database. However, extending to other backends is possible.

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

