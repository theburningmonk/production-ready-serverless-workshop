# Module 12: SSM Parameter Store

## Use SSM parameter store to configure resource names, URLs, etc.

**Goal:** Configurations are managed in SSM parameter store

<details>
<summary><b>Add dev configurations</b></summary><p>

1. Go to EC2 console

2. Go to `Parameter Store` (bottom left)

3. Click `Create Parameter`

4. Use the name `/{service-name}/dev/table_name` where `service-name` is `workshop-` followed by your name, e.g. `workshop-yancui`

![](/images/mod12-001.png)

5. Click `Create Parameter`

6. Repeat step 3-5 to create another `/{service-name}/dev/cognito_user_pool_id` parameter with your Cognito User Pool's ID

7. Repeat step 3-5 to create another `/{service-name}/dev/cognito_web_client_id` parameter with your Cognito User Pool's `web` client app ID

8. Repeat step 3-5 to create another `/{service-name}/dev/cognito_server_client_id` parameter with your Cognito User Pool's `server` client app ID

9. Repeat step 3-5 to create another `/{service-name}/dev/url` parameter with your the root URL for your deployed API without the ending `/`, e.g. `https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev`

</p></details>

<details>
<summary><b>Use SSM to parameterise serverless.yml</b></summary><p>

1. Replace the `restaurants_table` environment variable for `get-restaurants` and `search-restaurants` functions with `${ssm:/${self:service}/${opt:stage}/table_name}`. Whilst we're here, also update the `arn` for the `search-restaurants`'s `authorizer` to `${ssm:/${self:service}/${opt:stage}/cognito_user_pool_id}`

```yml
get-restaurants:
  handler: functions/get-restaurants.handler
  events:
    - http:
        path: /restaurants/
        method: get
        authorizer: aws_iam
  environment:
    restaurants_table: ${ssm:/${self:service}/${opt:stage}/table_name}

search-restaurants:
  handler: functions/search-restaurants.handler
  events:
    - http:
        path: /restaurants/search
        method: post
        authorizer:
          arn: arn:aws:cognito-idp:#{AWS::Region}:#{AWS::AccountId}:userpool/${ssm:/${self:service}/${opt:stage}/cognito_user_pool_id}
  environment:
    restaurants_table: ${ssm:/${self:service}/${opt:stage}/table_name}
```

2. Replace the `TableName` for the DynamoDB table with `${ssm:/${self:service}/${opt:stage}/table_name}`

```yml
resources:
  Resources:
    restaurantsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${ssm:/${self:service}/${opt:stage}/table_name}
        AttributeDefinitions:
          - AttributeName: name
            AttributeType: S
        KeySchema:
          - AttributeName: name
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
```

3. Replace the `cognito_user_pool_id` and `cognito_client_id` environment variables for the `get-index` function with `${ssm:/${self:service}/${opt:stage}/cognito_user_pool_id}` and `${ssm:/${self:service}/${opt:stage}/cognito_web_client_id}` respectively

```yml
get-index:
  handler: functions/get-index.handler
  events:
    - http:
        path: /
        method: get
  environment:
    restaurants_api: 
      Fn::Join:
        - ''
        - - "https://"
          - Ref: ApiGatewayRestApi
          - ".execute-api.${opt:region}.amazonaws.com/${opt:stage}/restaurants"
    cognito_user_pool_id: ${ssm:/${self:service}/${opt:stage}/cognito_user_pool_id}
    cognito_client_id: ${ssm:/${self:service}/${opt:stage}/cognito_web_client_id}
```

4. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

and go to the Lambda console to check that the environment variables are updated correctly.

</p></details>

<details>
<summary><b>Use SSM to parameterise seed-restaurants script</b></summary><p>

1. Modify the `seed-restaurants.js` script to the following

```javascript
const { REGION, STAGE } = process.env

const AWS = require('aws-sdk')
AWS.config.region = REGION
const dynamodb = new AWS.DynamoDB.DocumentClient()
const ssm = new AWS.SSM()

let restaurants = [
  { 
    name: "Fangtasia", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/fangtasia.png", 
    themes: ["true blood"] 
  },
  { 
    name: "Shoney's", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/shoney's.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Freddy's BBQ Joint", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/freddy's+bbq+joint.png", 
    themes: ["netflix", "house of cards"] 
  },
  { 
    name: "Pizza Planet", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/pizza+planet.png", 
    themes: ["netflix", "toy story"] 
  },
  { 
    name: "Leaky Cauldron", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/leaky+cauldron.png", 
    themes: ["movie", "harry potter"] 
  },
  { 
    name: "Lil' Bits", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/lil+bits.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Fancy Eats", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/fancy+eats.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Don Cuco", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/don%20cuco.png", 
    themes: ["cartoon", "rick and morty"] 
  },
];

const getTableName = async () => {
  console.log('getting table name...')
  const req = {
    Name: `/{replace this with what you used in SSM}/${STAGE}/table_name`
  }
  const ssmResp = await ssm.getParameter(req).promise()
  return ssmResp.Parameter.Value
}

const run = async () => {
  const tableName = await getTableName()

  console.log(`table name: `, tableName)

  let putReqs = restaurants.map(x => ({
    PutRequest: {
      Item: x
    }
  }))
  
  const req = { 
    RequestItems: {}
  }
  req.RequestItems[tableName] = putReqs
  await dynamodb.batchWrite(req).promise()
}

run().then(() => console.log("all done")).catch(err => console.error(err.message))
```

**REMINDER**: don't forget to replace the SSM parameter path with what you used in your `serverless.yml`

2. Rerun the script

`STAGE=dev REGION=us-east-1 node seed-restaurants.js`

and go to DynamoDB console to see that the newly created stage-specific table is now populated

</p></details>

<details>
<summary><b>Use SSM to parameterise tests</b></summary><p>

1. Modify `steps/init.js` to the following

```javascript
const _ = require('lodash')
const { promisify } = require('util')
const awscred = require('awscred')
const { REGION, STAGE } = process.env
const AWS = require('aws-sdk')
AWS.config.region = REGION
const SSM = new AWS.SSM()

let initialized = false

const getParameters = async (keys) => {
  const prefix = `/workshop-yancui/${STAGE}/`
  const req = {
    Names: keys.map(key => `${prefix}${key}`)
  }
  const resp = await SSM.getParameters(req).promise()
  return _.reduce(resp.Parameters, function(obj, param) {
    obj[param.Name.substr(prefix.length)] = param.Value
    return obj
   }, {})
}

const init = async () => {
  if (initialized) {
    return
  }

  const params = await getParameters([
    'table_name', 
    'cognito_user_pool_id', 
    'cognito_web_client_id',
    'cognito_server_client_id',
    'url'
  ])

  console.log('SSM params loaded')

  process.env.TEST_ROOT                = params.url
  process.env.restaurants_api          = `${params.url}/restaurants`
  process.env.restaurants_table        = params.table_name
  process.env.AWS_REGION               = REGION
  process.env.cognito_user_pool_id     = params.cognito_user_pool_id
  process.env.cognito_client_id        = params.cognito_web_client_id
  process.env.cognito_server_client_id = params.cognito_server_client_id
  
  const { credentials } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID     = credentials.accessKeyId
  process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey

  if (credentials.sessionToken) {
    process.env.AWS_SESSION_TOKEN = credentials.sessionToken
  }

  console.log('AWS credential loaded')

  initialized = true
}

module.exports = {
  init
}
```

2. Rerun the integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all the tests are passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (421ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (405ms)

  Given an authenticated user
[test-Gilbert-Caselli-cRsp(Egv] - user is created
[test-Gilbert-Caselli-cRsp(Egv] - initialised auth flow
[test-Gilbert-Caselli-cRsp(Egv] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
      ✓ Should return an array of 4 restaurants (248ms)
[test-Gilbert-Caselli-cRsp(Egv] - user deleted


  3 passing (3s)
```

3. Rerun the acceptance tests

`STAGE=dev REGION=us-east-1 npm run acceptance`

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (916ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (341ms)

  Given an authenticated user
[test-Viola-Brewer-keQeBQHj] - user is created
[test-Viola-Brewer-keQeBQHj] - initialised auth flow
[test-Viola-Brewer-keQeBQHj] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants/search
      ✓ Should return an array of 4 restaurants (1514ms)
[test-Viola-Brewer-keQeBQHj] - user deleted


  3 passing (5s)
```

4. Commit and push your changes to see that they're still passing on CodePipeline too

</p></details>

## Exercises

<details>
<summary><b>Add staging configurations</b></summary><p>

To make the pipeline work for both dev and staging, we also need to add the parameters for the staging environment too.

One tricky thing is that we don't know the deployed API's URL for staging until we have deployed it!

One workaround is to use `serverless-domain-manager` plugin to create predictable domain names for each stage:

* dev.bigmouth.com

* staging.bigmouth.com

* prod.bigmouth.com

So we can assign them ahead of them.

See [here](https://www.npmjs.com/package/serverless-domain-manager) for more details.

</p></details>