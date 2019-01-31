
All machines using the first version of the specification are denoted as:

```yaml
version: 1.0
```

## Full Example

!!! example
    ```yaml
    version: 1.0

    events:
      - LAUNCH
      - DESTROY

    states:
      INIT:
        on:
          LAUNCH:
            - action: advance
              state: PROVISION

      PROVISION:
        tasks:
          - db-setup
          - provision-machines
        on:
          NEXT:
            - action: advance
              state: RUN

      RUN:
        on:
          DESTROY:
            - action: advance
              state: TEARDOWN

      TEARDOWN:
        terminal: true
        tasks:
          - db-teardown
          - teardown-machines

    tasks:
      db-setup:
        executor: lambda
        config:
          name: db-setup

      db-teardown:
        executor: lambda
        config:
          name: db-teardown

      provision-machines:
        executor: terraform
        config:
          source:
            type: s3
            bucket: my-terraform-scripts
            key: machines.tf
          action: apply
          variables:
            desired_capacity: 1

      teardown-machines:
        executor: terraform
        config:
          source:
            type: s3
            bucket: my-terraform-scripts
            key: machines.tf
          action: destroy
          variables:
            desired_capacity: 0
    ```

## `events`

The `.events` key defines all external events to the state machine. These are used to validate in the `.states.on` keys below.

- All event names must be unique
- All event names must match the pattern `[A-Z0-9-]`

!!! example
    ```yaml
    events:
        - LAUNCH
        - DESTROY
    ```

## `states`

The collection of `.states` contains a map of state names with additional details. Each state defines a new state within the machine.

- State names must be unique
- State names must match the `[A-Z0-9-]` regex pattern
- Every machine must define an `INIT` state, which is the default state when creating a new machine.
- At least one state must be defined as a `terminal` state.

!!! example
    ```yaml
    states:
      INIT:
        on:
          ...
      PROVISION:
        on:
          ...
        tasks:
          ...
      RUN:
        on:
          ...
      DESTROY:
        terminal: true
    ```

### `terminal`

When a state includes `terminal: true`, when it is satisfied (all tasks completed), the state machine will be removed from persistent storage.

### `on`

Within the `on` key, we list the events that this state will respond to, as well as the actions to perform for each action. While most states will perform only one action, multiple actions are supported.

- Each event **must** be either defined at `.events` or a system event (`NEXT`)

!!! example
    ```yaml
    states:
      INIT:
        on:
          LAUNCH:
            - action: advance
              state: PROVISION
    ```

#### Supported actions

There are currently two supported actions:

##### advance

With `action: advance`, the state machine will transfer to a new state. A `state` key is required, who's value is the name of another state in the machine.

!!! example
    ```yaml
    states:
      INIT:
        on:
          LAUNCH:
            - action: advance
              state: PROVISION
      PROVISION:
      ...
    ```

##### no-op

With `action: no-op`, the state machine will do nothing upon this event. Why is this useful? It prevents an error from being thrown if the event is triggered while in the specified state. 

!!! info
    No-op actions are extremely useful for events that are only updating `metadata` state on the stack. For example, marking a "last accessed" timestamp for a stack.

!!! example
    ```yaml
    states:
      RUN:
        on:
          MARK_ACCESS:
            - action: no-op
      ...
    ```


### `tasks`

For each state, a collection of tasks can be identified to indicate what needs to be completed in order to transition to the next state. Upon completion of all tasks, the state machine will automatically call the `NEXT` event.

- All non-terminal states that define tasks are required to have a `NEXT` event handler.
- Each task that is listed **must** have a corresponding definition at `.tasks`.

!!! example
    ```yaml
    states:
      PROVISION:
        tasks:
          - cert-provision
        on:
          NEXT:
          ...
    ```


## `tasks`

All tasks that are needed for completion in order for advancement must be defined with the `.tasks` key. 


### `executor`

Currently, only two task executors are supported (`terraform` and `lambda`). The configuration for each depends on the selected executor.


#### `terraform`

The `terraform` executor allows you to perform tasks using TF scripts. The `config` object allows for the following configuration:

- `action` (required) - allows either `apply` or `destroy`. Determines the command being used when running the task.
- `source` (required) - the location to pull the scripts from
  - `type` - allows `s3` or `local`. 
    - `s3` - both `bucket` and `key` options are required. If the specified key is a directory, all objects are checked out.
    - `local` - `location` is required, which is an absolute path to the files (most often used during testing).
- `variables` - additional variables to include with the script execution. The example below will add `-var 'desired_capacity=1'` to the command execution.

!!! example
    ```yaml
    tasks:
      provision-machines:
        executor: terraform
        config:
          source:
            type: s3
            bucket: my-terraform-scripts
            key: machines.tf
          action: apply
          variables:
            desired_capacity: 1
    ```


#### `lambda`

The `lambda` executor allows you to execute an arbitrary function packaged as a Lambda function. The `config` object accepts the following:

- `name` (required) - the name of the Lambda function to execute

!!! example
    ```yaml
    tasks:
      db-setup:
        executor: lambda
        config:
          name: db-setup
    ```
