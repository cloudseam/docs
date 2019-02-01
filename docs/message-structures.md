
There are two basic types of messages being sent through the system. By understanding their structure, you can craft your own messages and send them directly to test various situations.

- [Event Messages](#event-messages)
- [Task Messages](#task-messages)


## Event Messages

Event messages are either system or user generated events that invoke some sort of action on the state machine. 

### Message Structure

Each messages are JSON objects that can contain the following:

- `stackId` (required) - the ID of the stack to act on
- `action` (required) - the action to perform/apply to the state machine
- `machine` (required on first appearance) - the name of the machine this stack should use. _MUST_ be provided on the first event for a stack, but not required on all subsequent events. Names are simply the YML filename, minus the `.yml` file extension.
- `metadata` (optional) - any arbitrary metadata to associate with the stack. The object _must_ be flat, so no nested objects/arrays. The metadata will be persisted with the stack and made available during task completion.


### System Actions

There are a few reserved actions, as noted in the [State Machine Overview](/state-machine-overview). As such, you may see log entries for events beyond just the ones you created yourself.


### Example Messages

The following message is the first event for a stack named `master`, so includes the required `machine` identifier. It also has additional metadata, which might represent a container image tag to be used later when running a container.

```json
{
    "stackId": "master",
    "machine": "qa-env",
    "action": "LAUNCH",
    "metadata": {
        "apiImageTag": "e724e901125e90eda014beb5fb7e6e9560e4153c"
    }
}
```

The following message will trigger the `DESTROY` event on the current state for the `master` stack.

```json
{
    "stackId": "master",
    "action": "DESTROY"
}
```


### Task Messages

Task messages are produced by the state machine and submitted to the task queue for completion.

### Message Structure

Each messages are JSON objects that can contain the following:

- `task` (required) - the machine-specified configuration for the task
- `stack` (required) - current state for the stack being operated on. Contains the `name` and `metadata`, which can be used as desired by the 


### System Actions

There are a few reserved actions, as noted in the [State Machine Overview](/state-machine-overview). As such, you may see log entries for events beyond just the ones you created yourself.


### Example Message

The following message is a request to execute a Lambda function named `db-setup` to satisfy the `db-setup` task for the `master` stack. It has the same metadata from the previous `LAUNCH` event example.

```json
{
    "task": {
        "executor": "lambda",
        "config": {
            "name": "db-setup"
        },
        "name": "db-setup"
    },
    "stack": {
        "id": "master",
        "previousState": "INIT",
        "machine": "qa-env",
        "_state": "PROVISION",
        "tasks": [
            {
                "name": "db-setup",
                "status": "PENDING"
            }
        ],
        "metadata": {
            "apiImageTag": "e724e901125e90eda014beb5fb7e6e9560e4153c",
        }
    }
}
```
