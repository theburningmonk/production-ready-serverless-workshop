# Module 19: Structured logging

## Apply structured logging to the project

<details>
<summary><b>Create a simple logger</b></summary><p>

1. Add a file `log.js` to the `lib` folder

2. Modify `lib/log.js` to the following

```javascript
const LogLevels = {
  DEBUG : 0,
  INFO  : 1,
  WARN  : 2,
  ERROR : 3
}

// default to debug if not specified
const logLevelName = process.env.log_level || 'DEBUG'

const isEnabled = (level) => level >= LogLevels[logLevelName]

function appendError(params, err) {
  if (!err) {
    return params
  }

  return Object.assign(
    { },
    params || { }, 
    { errorName: err.name, errorMessage: err.message, stackTrace: err.stack }
  )
}

function log (levelName, message, params) {
  if (!isEnabled(LogLevels[levelName])) {
    return
  }

  let logMsg = Object.assign({}, params)
  logMsg.level = levelName
  logMsg.message = message

  console.log(JSON.stringify(logMsg))
}

module.exports = {
  debug: (msg, params) => log('DEBUG', msg, params),
  info: (msg, params) => log('INFO',  msg, params),
  warn: (msg, params, error) => log('WARN',  msg, appendError(params, error)),
  error: (msg, params, error) => log('ERROR', msg, appendError(params, error))
}
```

3. Modify `functions/get-index.js` and replace `console.log` with use of the logger

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')
const Log = require('../lib/log')
```

```javascript
function loadHtml () {
  if (!html) {
    Log.info('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    Log.info('loaded')
  }
  
  return html
}
```

4. Modify `functions/place-order.js` and replace `console.log` with use of the logger

```javascript
const _ = require('lodash')
const AWS = require('aws-sdk')
const kinesis = new AWS.Kinesis()
const chance = require('chance').Chance()
const Log = require('../lib/log')
const streamName = process.env.order_events_stream
```

```javascript
const userEmail = _.get(event, 'requestContext.authorizer.claims.email')
if (!userEmail) {
  Log.error('user email is not found')
  return UNAUTHORIZED
}

const orderId = chance.guid()
Log.debug(`placing order ID [${orderId}] to [${restaurantName}] for user [${userEmail}]`)
```

```javascript
await kinesis.putRecord(req).promise()

Log.debug(`published 'order_placed' event into Kinesis`)

const response = {
  statusCode: 200,
  body: JSON.stringify({ orderId })
}
```

5. Modify `functions/notify-restaurant.js` and replace `console.log` with use of the logger

```javascript
const { getRecords } = require('../lib/kinesis')
const notify = require('../lib/notify')
const retry = require('../lib/retry')
const Log = require('../lib/log')
```

```javascript
try {
  await notify.restaurantOfOrder(order)
} catch (err) {
  Log.warn(`failed to notify restaurant of order [${order.orderId}], queuing for retry...`)
  await retry.restaurantNotification(order)
}
```

6. Modify `lib/notify.js` and replace `console.log` with use of the logger

```javascript
const _ = require('lodash')
const AWS = require('aws-sdk')
const sns = new AWS.SNS()
const kinesis = new AWS.Kinesis()
const chance  = require('chance').Chance()
const Log = require('./log')
```

```javascript
await sns.publish(snsReq).promise()
Log.debug(`notified restaurant [${order.restaurantName}] of order [${order.orderId}]`)
```

```javascript
await kinesis.putRecord(kinesisReq).promise()
  Log.debug(`published 'restaurant_notified' event to Kinesis`)
```

7. Modify `lib/retry.js` and replace `console.log` with use of the logger

```javascript
const AWS = require('aws-sdk')
const sns = new AWS.SNS()
const Log = require('../lib/log')
```

```javascript
await sns.publish(snsReq).promise()
Log.info(`order [${order.orderId}]: queued restaurant notification for retry`)
```

8. Run the integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that the functions are now logging in JSON

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
{"level":"INFO","message":"loading index.html..."}
{"level":"INFO","message":"loaded"}
    ✓ Should return the index page with 8 restaurants (1642ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (1321ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
{"level":"DEBUG","message":"notified restaurant [Fangtasia] of order [4ec5038c-5181-50ba-a706-2c6a57757b69]"}
{"level":"DEBUG","message":"published 'restaurant_notified' event to Kinesis"}
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  Given an authenticated user
[test-Ina-Louis-N0LIpIfa] - user is created
[test-Ina-Louis-N0LIpIfa] - initialised auth flow
[test-Ina-Louis-N0LIpIfa] - responded to auth challenge
    When we invoke the POST /orders endpoint
invoking via handler function place-order
{"level":"DEBUG","message":"placing order ID [aeeac0e1-7d72-5144-b8c5-ab9eef70c657] to [Fangtasia] for user [test-Ina-Louis-N0LIpIfa@test.com]"}
{"level":"DEBUG","message":"published 'order_placed' event into Kinesis"}
      ✓ Should return 200
      ✓ Should publish a message to Kinesis stream
[test-Ina-Louis-N0LIpIfa] - user deleted

  Given an authenticated user
[test-Eugene-Forconi-qRvWv)m]] - user is created
[test-Eugene-Forconi-qRvWv)m]] - initialised auth flow
[test-Eugene-Forconi-qRvWv)m]] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
      ✓ Should return an array of 4 restaurants (248ms)
[test-Eugene-Forconi-qRvWv)m]] - user deleted


  7 passing (8s)
```

</p></details>

<details>
<summary><b>Disable debug logging in production</b></summary><p>

1. Modify `serverless.yml` to add a `custom` section

```yml
custom:
  logLevel:
    prod: INFO
    default: DEBUG

provider:
  name: aws
  runtime: nodejs8.10
  environment:
    log_level: ${self:custom.logLevel.${opt:stage}, self:custom.logLevel.default}
```

This applies the `log_level` environment variable (used to what level the logger should log at) to all the functions in the project (since it's specified under `provider`).

It references the `custom.logLevel` section (with `self:`), and also references the `stage` deployment option. So when the deployment stage is `prod`, it resolves to `self:custom.logLevel.prod` and `log_level` would be set to `INFO`.

The second argument, `self:custom.logLevel.default` is the fallback if the first path is not found. If the deployment stage is `dev`, it'll see that `self:custom.logLevel.dev` doesn't exist, and therefore use the fallback `self:custom.logLevel.default` and set `log_level` to `DEBUG` in that case.

This is a nice trip to specify a stage-specific override, but then fall back to some default value otherwise.

</p></details>