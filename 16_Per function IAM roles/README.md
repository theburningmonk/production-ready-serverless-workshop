# Module 16: Per function IAM roles

## Give each function a dedicated IAM role

**Goal:** Each function has a dedicated IAM role and therefore minimise attack surface

<details>
<summary><b>Install serverless-iam-roles-per-function plugin</b></summary><p>

1. Install `serverless-iam-roles-per-function` as dev dependency

`npm install --save-dev serverless-iam-roles-per-function`

2. Modify `serverless.yml` and add it as a plugin

```yml
plugins:
  - serverless-pseudo-parameters
  - serverless-iam-roles-per-function
```

</p></details>

<details>
<summary><b>Issue individual permissions</b></summary><p>

1. Modify `serverless.yml` and delete the `iamRoleStatements` section

2. Modify `serverless.yml` and give the `get-index` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: execute-api:Invoke
    Resource: arn:aws:execute-api:#{AWS::Region}:#{AWS::AccountId}:*/*/GET/restaurants
```

3. Modify `serverless.yml` and give the `get-restaurants` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: dynamodb:scan
    Resource:
      Fn::GetAtt:
        - restaurantsTable
        - Arn
```

4. Modify `serverless.yml` and give the `search-restaurants` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: dynamodb:scan
    Resource:
      Fn::GetAtt:
        - restaurantsTable
        - Arn
```

5. Modify `serverless.yml` and give the `place-order` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: kinesis:PutRecord
    Resource: 
      Fn::GetAtt:
        - orderEventsStream
        - Arn
```

6. Modify `serverless.yml` and give the `notify-restaurant` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: kinesis:PutRecord
    Resource: 
      Fn::GetAtt:
        - orderEventsStream
        - Arn
  - Effect: Allow
    Action: sns:Publish
    Resource: 
      - Ref: restaurantNotificationTopic
      - Ref: restaurantNotificationRetryTopic
```

7. Modify `serverless.yml` and give the `retry-notify-restaurant` function its own IAM role statements

```yml
iamRoleStatements:
  - Effect: Allow
    Action: kinesis:PutRecord
    Resource: 
      Fn::GetAtt:
        - orderEventsStream
        - Arn
  - Effect: Allow
    Action: sns:Publish
    Resource: 
      - Ref: restaurantNotificationTopic
      - Ref: restaurantNotificationDLQTopic
```

8. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

9. Run the acceptance tests to make sure they're still working

`STAGE=dev REGION=us-east-1 npm run acceptance`

</p></details>