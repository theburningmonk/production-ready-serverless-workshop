# Module 21: X-Ray

## Collect invocation traces in X-Ray

<details>
<summary><b>Integrating with X-Ray</b></summary><p>

1. Install `serverless-plugin-tracing` as dev dependency

`npm install --save-dev serverless-plugin-tracing`

2. Modify `serverless.yml` to add `serverless-plugin-tracing` as a plugin (under the `plugins` section)

```yml
plugins:
  - serverless-pseudo-parameters
  - serverless-iam-roles-per-function
  - serverless-plugin-tracing
```

3. Modify `serverless.yml` to add `tracing: true` to the `provider` section

```yml
provider:
  name: aws
  runtime: nodejs8.10
  tracing: true
```

This enables X-Ray tracing for all the functions in this project. However, we still need to give each function the IAM permission for `xray:PutTraceSegments` and `xray:PutTelemetryRecords`.

4. Modify `serverless.yml` to add these permissions to the `provider` section

```yml
provider:
  name: aws
  runtime: nodejs8.10
  tracing: true
  environment:
    log_level: ${self:custom.logLevel.${opt:stage}, self:custom.logLevel.default}
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - "xray:PutTraceSegments"
        - "xray:PutTelemetryRecords"
      Resource:
        - "*"
```

And we need the functions to inherit the permissions from this default IAM role.

5. Modify `serverless.yml` to add the following to the `custom` section

```yml
  serverless-iam-roles-per-function:
    defaultInherit: true
```

This is courtesy of the `serverless-iam-roles-per-function` plugin, and lets you easily share common permissions that all functions should have.

6. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

7. Load up the landing page, and place an order. Then head to the X-Ray console and see what you get.

![](/images/mod21-001.png)

![](/images/mod21-002.png)

![](/images/mod21-003.png)

</p></details>

<details>
<summary><b>Instrumenting AWSSDK</b></summary><p>

At the moment we're not getting a lot of value out of X-Ray. We can get much more information about what's happening in our code if we instrument the various steps.

To begin with, we can instrument the AWS SDK so we track how long calls to DynamoDB and SNS takes in the traces.

1. Install `aws-xray-sdk-core` as dependency

`npm install --save aws-xray-sdk-core`

2. Modify `functions/get-restaurants.js` and replace `const AWS = require('aws-sdk')` with the following

```javascript
const AWSXRay = require('aws-xray-sdk-core')
const AWS = AWSXRay.captureAWS(require('aws-sdk'))
```

3. Repeat step 2 for

* `functions/place-order.js`

* `functions/search-restaurants.js`

* `lib/notify.js`

* `lib/retry.js`

4. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

5. Load up the landing page, and place an order. Then head to the X-Ray console and see what you get now.

![](/images/mod21-004.png)

![](/images/mod21-005.png)

![](/images/mod21-006.png)

![](/images/mod21-007.png)

</p></details>

<details>
<summary><b>Instrumenting HTTP calls</b></summary><p>

We can get a lot value if we could see the traces for `get-index` function and the corresponding trace for the `get-restaurants` function in one screen.

![](/images/mod21-008.png)

Then it's proper distributed tracing! It's not very helpful if you're restricted to only what happens inside one function.

Fortunately, you can instrument the built-in `https` module with the X-Ray SDK, unfortunately, you have to use it instead of other HTTP clients..

1. Modify `functions/get-index.js` and replace 

`const http = require('superagent-promise')(require('superagent'), Promise)`

with 

```javascript
const AWSXRay = require('aws-xray-sdk-core')
const https = AWSXRay.captureHTTPs(require('https'))
```

2. Modify `functions/get-index.js` and replace the `getRestaurants` function with the following

```javascript
const getRestaurants = () => {
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname, 
    path: url.pathname
  }

  aws4.sign(opts)

  return new Promise((resolve, reject) => {
    const options = {
      hostname: url.hostname,
      port: 443,
      path: url.pathname,
      method: 'GET',
      headers: opts.headers
    }

    const req = https.request(options, res => {
      res.on('data', buffer => {
        const body = buffer.toString('utf8')
        resolve(JSON.parse(body))
      })
    })

    req.on('error', err => reject(err))

    req.end()
  })
}
```

It uses the `https` module to make the HTTP request to the `/restaurants` endpoint instead.

3. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

4. Load up the landing page, and place an order. Then head to the X-Ray console and now you can see the traces for `get-index` and `get-restaurants` function in one place.

</p></details>

<details>
<summary><b>Fix broken tests</b></summary><p>

If you run the integration tests now 

`STAGE=dev REGION=us-east-1 npm run test`

then you'll see the tests are broken...

This is because the X-Ray SDK expects some context and root segment to be provided by the Lambda service's runtime. Which we won't have when running locally.

1. Modify `steps/init.js` to add this along with other environment variables

`process.env.AWS_XRAY_CONTEXT_MISSING = 'LOG_ERROR'`

This stops the X-Ray SDK from erroring when it doesn't find the context

Rerun the integration tests, and the tests are still broken, with errors like this

```
  2) When we invoke the GET /restaurants endpoint
       Should return an array of 8 restaurants:
     TypeError: Service.prototype.customizeRequests is not a function
      at Object.captureAWS (node_modules/aws-xray-sdk-core/lib/patchers/aws_p.js:37:25)
      at Object.<anonymous> (functions/get-restaurants.js:5:21)
      at require (internal/module.js:11:18)
      at viaHandler (tests/steps/when.js:79:34)
      at Object.we_invoke_get_restaurants (tests/steps/when.js:103:15)
      at Context.it (tests/test_cases/get-restaurants.js:9:26)
```

2. The best bad way to work around this (except just giving up on the X-Ray SDK altogether) is to not use it when executing locally. When the function is running in the Lambda execution environment, it has a number of environment variables, including one called `LAMBDA_RUNTIME_DIR`

![](/images/mod21-009.png)

Go back to `functions/get-index.js` and replace

`const https = AWSXRay.captureHTTPs(require('https'))`

with 

```javascript
const https = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureHTTPs(require('https'))
  : require('https')
```

3. Similarly, modify `functions/get-restaurants.js` and replace

`const AWS = AWSXRay.captureAWS(require('aws-sdk'))`

with 

```javascript
const AWS = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureAWS(require('aws-sdk'))
  : require('aws-sdk')
```

4. Repeat step 3 with

* `functions/place-order.js`

* `functions/search-restaurants.js`

* `lib/notify.js`

* `lib/retry.js`

5. Rerun the integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all the tests should be passing now

</p></details>