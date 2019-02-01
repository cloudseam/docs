There are two different Lambda functions to be deployed. Eventually, we hope to have an installer script to make this easier. Until then, it's a fairly manual process.

The installation steps consist of:

1. [Deploy the Lambda functions](#deploy-the-lambda-functions)
2. [Create your machine specifications](#create-your-machine-specifications)
3. [Define your tasks](#define-your-tasks)
4. [Publish Events](#publish-events)


## Deploy the Lambda functions

Deploying Lambda functions tends to use the following process:

1. Download the ZIP files from the [Releases page](https://github.com/cloudseam/cloudseam/releases).
2. Verify the signatures (ensure they weren't tampered). 

    `gpg --verify task-satisfier-0.1.0.zip.asc task-satisfier-0.1.0.zip`

3. Push the ZIPs to an S3 bucket in your account.
4. Create the IAM policies needed for the functions ([State Machine](#state-manager-permissions) and [Task Satisfier](#task-satisfier-permissions)).
5. Deploy the functions with the appropriate environment variables (see [Configuration](/configuration)).


### State Manager Permissions

The State Manager needs the following permissions:

- Permission to read and delete messages from the event queue
- Permission to send messages to the task queue
- Permission to access the credential to access the database. See [Database Configuration](/configuration/#database-configuration) for more info.
- Permission to read the machine specifications from their S3 bucket
- Other permissions needed to start a Lambda function (logging, connect to VPC)


The following AWS IAM Policy provides the necessary permissions for the state manager. The `Sid` for each statement explains the reasoning for the permission. 

!!! info "Sample IAM Policy"
    Note that there are placeholders in the policy below for SQS queue arns, machine bucket names, and the Secrets Manager arn.


    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "ReadAndProcessMessagesFromEventQueue",
                "Effect": "Allow",
                "Action": [
                    "sqs:DeleteMessage",
                    "sqs:ReceiveMessage",
                    "sqs:GetQueueAttributes"
                ],
                "Resource": "$SQS_EVENT_QUEUE_URL-arn"
            },
            {
                "Sid": "SendMessagesToTaskQueue",
                "Effect": "Allow",
                "Action": [
                    "sqs:SendMessage"
                ],
                "Resource": "$SQS_TASK_QUEUE_URL-arn"
            },
            {
                "Sid": "CreateLogsAccess",
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Resource": [
                    "arn:aws:logs:*:*:*"
                ]
            },
            {
                "Sid": "StateMachineSpecBucketAccess",
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::$MACHINE_S3_BUCKET/*",
                    "arn:aws:s3:::$MACHINE_S3_BUCKET"
                ]
            },
            {
                "Sid": "ConnectToVPC",
                "Effect": "Allow",
                "Action": [
                    "ec2:CreateNetworkInterface",
                    "ec2:DescribeNetworkInterfaces",
                    "ec2:DeleteNetworkInterface"
                ],
                "Resource": "*"
            },
            {
                "Sid": "PostgresConnectionSecretsAccess",
                "Effect": "Allow",
                "Action": [
                    "secretsmanager:GetSecretValue"
                ],
                "Resource": "$POSTGRES_CONNECTION_SECRET-arn"
            }
        ]
    }
    ```



### Task Satisfier Permissions

The Task Satisfier needs whatever permissions are needed to accomplish the tasks. This will vary greatly, based on what you need to be done. At a minimum, it will need the following permissions:

- Permission to read and delete messages from the task queue
- Permission to send messages to the task queue

If you have any tasks using the Terraform executor, you will need:

- Permission to obtain Terraform scripts from S3 (buckets/keys defined in the machine spec)

If you have any tasks using the Lambda executor, you will need:

- Permission to invoke each expected function (`lambda:InvokeFunction`)


!!! info "Sample IAM Policy"
    **NOTE:** The following policy grants ONLY the minimum permissions, not anything needed by executors/tasks.


    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "ReadAndProcessMessagesFromTaskQueue",
                "Effect": "Allow",
                "Action": [
                    "sqs:DeleteMessage",
                    "sqs:ReceiveMessage",
                    "sqs:GetQueueAttributes"
                ],
                "Resource": "$SQS_TASK_QUEUE_URL-arn"
            },
            {
                "Sid": "SendMessagesToEventQueue",
                "Effect": "Allow",
                "Action": [
                    "sqs:SendMessage"
                ],
                "Resource": "$SQS_EVENT_QUEUE_URL-arn"
            },
            {
                "Sid": "CreateLogsAccess",
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Resource": [
                    "arn:aws:logs:*:*:*"
                ]
            }
        ]
    }
    ```


## Create your machine specifications

Now, you're ready to define your actual machines! Instead of repeating everything here, use these resources:

- [State Machine Overview](/state-machine-overview)
- [Machine 1.0 Specification](/specs/version-1-0/)
- [Machine Editor](https://editor.cloudseam.app/)

Once you have your state machine and you've validated it meets the spec (the editor will tell you this), upload your machines to your S3 bucket and ensure your State Machine function has permission to access it.



## Define your tasks

For each task, you need either a Terraform script or Lambda function. It's up to you write, test, and validate those. However, a few things to note...

- All `metadata` for a stack will be made available as either variables (Terraform) or execution context (Lambda).



### Terraform Scripts

With Terraform, you are able to execute any changes you'd like to your cloud environment. 

!!! info
    The Terraform executor currently only supports the `s3` backend, which is required (as state persistence is necessary).

When each Terraform-based task is executed, the following sequence occurs:

1. The Terraform scripts are pulled from S3 into a temp working directory
2. In the temp working directory, `terraform init` is executed with `-backend-config="key=${machineName}/${stackId}/${task.config.source.key}`
3. Either `terraform apply` or `terraform destroy` is executed with the following:
    - `-auto-approve -input=false` to prevent interaction requirements
    - The stack id as `stack_id` (`-var='stack_id=${stackId}'`)
    - Each metadata expanded out as separate variables. 
  
    For example, if your stack has metadata `{"apiImage":"latest"}` for an `apply` tasks, the following command would be executed

    ```
    terraform apply -auto-approve -input=false -var='stack_id=${stackId}' -var='apiImage=latest'
    ```

4. If Terraform fails (non-zero exit code), the task fails.

### Lambda Functions

Each Lambda function, when invoked, is given a payload with the following contents:

```json
{
    "stackId": "the-name-of-the-stack",
    "metadata": {
        // The metadata currently on the stack     
    }
}
```

If the Lambda function fails, the task execution will be reported as a failure.

Once your functions are deployed, you can test them using the [Task message structure](/message-structures#task-messages). If they pass via manual interaction, you should be good to go (assuming your Task Satisfier has permissions).


## Publish Events

At this point, you're ready to start publishing events. While testing, it might be best to go directly into the SQS Console and publish messages to the Event Queue (see the [Event message structure](/message-structures#event-messages)). Once you're feeling confident, start publishing events via automated means (CI/CD pipelines, other Lambda functions, etc.). 
