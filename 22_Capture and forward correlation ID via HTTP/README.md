# Module 22: Capture and forward correlation ID via HTTP

## Capture and forward correlation IDs via HTTP

<details>
<summary><b>Add a module to record correlation IDs</b></summary><p>

1. Add a file `correlation-ids.js` to the `lib` folder

2. Modify `lib/correlation-ids.js` to the following

```javascript
const clearAll = () => global.CONTEXT = undefined

const replaceAllWith = ctx => global.CONTEXT = ctx

const set = (key, value) => {
  if (!key.startsWith("x-correlation-")) {
    key = "x-correlation-" + key
  }

  if (!global.CONTEXT) {
    global.CONTEXT = {}
  }

  global.CONTEXT[key] = value;
}

const get = () => global.CONTEXT || {}

module.exports = {
  clearAll: clearAll,
  replaceAllWith: replaceAllWith,
  set: set,
  get: get
}
```

</p></details>

<details>
<summary><b>Include captured correlation IDs in all logs</b></summary><p>

1. Modify `lib/log.js` to require the new `correlation-id` module at the top

`const CorrelationIds = require('./correlation-ids')`

2. Modify `lib/log.js` to add a bunch of contextual information from the environment variables

```javascript
// most of these are available through the Node.js execution environment for Lambda, see the following for details
// https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html
const DEFAULT_CONTEXT = {
  awsRegion: process.env.AWS_REGION || process.env.AWS_DEFAULT_REGION,
  functionName: process.env.AWS_LAMBDA_FUNCTION_NAME,
  functionVersion: process.env.AWS_LAMBDA_FUNCTION_VERSION,
  functionMemorySize: process.env.AWS_LAMBDA_FUNCTION_MEMORY_SIZE,
  stage: process.env.ENVIRONMENT || process.env.STAGE
}
```

This adds a new environment variable `STAGE`, which we should add as a shared environment variable to all our functions.

3. Modify `serverless.yml` and add the `STAGE` environment variable under the `provider.environment` section

```yml
environment:
  log_level: ${self:custom.logLevel.${opt:stage}, self:custom.logLevel.default}
  STAGE: ${opt:stage}
```

4. Modify `lib/log.js` to add a new `getContext` function to

* load the captured correlation IDs

* merge them with the `DEFAULT_CONTEXT`

to give us all the additional attributes that should be included in every log message

```javascript
function getContext () {
  // if there's a global variable for all the current request context then use it
  const context = CorrelationIds.get()
  if (context) {
    // note: this is a shallow copy, which is ok as we're not going to mutate anything
    return Object.assign({}, DEFAULT_CONTEXT, context)
  }

  return DEFAULT_CONTEXT
}
```

5. Modify `lib/log.js` and update the `log` function to the following

```javascript
function log (levelName, message, params) {
  if (!isEnabled(LogLevels[levelName])) {
    return
  }

  let context = getContext()
  let logMsg = Object.assign({}, context, params)
  logMsg.level = levelName
  logMsg.message = message

  console.log(JSON.stringify(logMsg))
}
```

Here we merge the additional attributes (correlation IDs + `DEFAULT_CONTEXT`) with our log message before writing it to `stdout` as JSON.

</p></details>

<details>
<summary><b>Follow debug logging decisions from upstream</b></summary><p>

So far, our `sample-logging` middleware enables debugging logging for 1% of invocation for each function. But to help debug a call chain involving multiple functions, we need to propagate that debug logging decision along.

For that, we'll introduce a special correlation ID called `debug-log-enabled` and pass it along with other correlation IDs.

1. Modify `middleware/sample-logging.js` and require the `correlation-ids` module at the top

`const CorrelationIds = require('../lib/correlation-ids')`

2. Modify `middleware/sample-logging.js` and update the `isDebugEnabled` function to the following

```javascript
const isDebugEnabled = () => {
  const context = CorrelationIds.get()
  if (context['debug-log-enabled'] === 'true') {
    return true
  }

  return sampleRate && Math.random() <= sampleRate
}
```

Now, whenever we see the `debug-log-enabled` correlation ID, and it's set to `true`, then we'll also enable debug logging for the current invocation.

</p></details>

<details>
<summary><b>Auto-capture incoming correlation IDs via API Gateway events</b></summary><p>

We can use a middleware to

* inspect the incoming invocation event

* extract the correlation IDs

* store the correlation IDs with the `correlation-ids` module

1. Add a file `capture-correlation-ids.js` to the `middleware` folder

2. Modify `middleware/capture-correlation-ids.js` to the following

```javascript
const CorrelationIds = require('../lib/correlation-ids')
const Log = require('../lib/log')

function captureHttp(headers, awsRequestId, sampleDebugLogRate) {
  if (!headers) {
    Log.warn(`Request ${awsRequestId} is missing headers`)
    return
  }

  let context = { awsRequestId }
  for (const header in headers) {
    if (header.toLowerCase().startsWith('x-correlation-')) {
      context[header] = headers[header]
    }
  }
 
  if (!context['x-correlation-id']) {
    context['x-correlation-id'] = awsRequestId
  }

  // forward the original User-Agent on
  if (headers['User-Agent']) {
    context['User-Agent'] = headers['User-Agent']
  }

  if (headers['debug-log-enabled']) {
    context['debug-log-enabled'] = headers['debug-log-enabled']
  } else {
    context['debug-log-enabled'] = Math.random() < sampleDebugLogRate ? 'true' : 'false'
  }

  CorrelationIds.replaceAllWith(context)
}

function isApiGatewayEvent(event) {
  return event.hasOwnProperty('httpMethod')
}

module.exports = (config) => {
  const sampleDebugLogRate = config ? config.sampleDebugLogRate || 0.01 : 0.01 // defaults to 1%

  return {
    before: (handler, next) => {      
      CorrelationIds.clearAll()

      if (isApiGatewayEvent(handler.event)) {
        captureHttp(handler.event.headers, handler.context.awsRequestId, sampleDebugLogRate)
      }
      
      next()
    }
  }
}
```

3. Add the `capture-correlation-ids` middleware to `lib/wrapper.js`. Bear in mind that order matters, we need to capture the correlation IDs first, before the `sample-logging` middleware can use it to turn on debug logging, so it has to go first. 

Update `lib/wrapper.js` to the following

```javascript
const middy = require('middy')
const sampleLogging = require('../middleware/sample-logging')
const captureCorrelationIds = require('../middleware/capture-correlation-ids')

module.exports = (f) => {
  return middy(f)
    .use(captureCorrelationIds({ sampleDebugLogRate: 0.01 }))
    .use(sampleLogging({ sampleRate: 0.01 }))
}
```

4. Modify `functions/get-restaurants.js` to require the `log` module

`const Log = require('../lib/log')`

add a log message after we loaded restaurants from DynamoDB

```javascript
const restaurants = await getRestaurants(defaultResults)
Log.debug(`fetched ${restaurants.length} restaurants`)
```

</p></details>

<details>
<summary><b>Auto-forward captured correlation IDs via HTTP</b></summary><p>

We need to propagate the correlation IDs we have captured so far and pass it along whenever we make a HTTP request to another service.

1. Add a file `https.js` to the `lib` folder

2. Modify `lib/https.js` to the following

```javascript
const CorrelationIds = require('./correlation-ids')
const AWSXRay = require('aws-xray-sdk-core')
const https = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureHTTPs(require('https'))
  : require('https')

// options: { 
//    hostname : string
//    method   : GET | POST | PUT | HEAD
//    path     : string
//    headers  : object
//  }
// for all intents and purposes you can think of this as `https.request`
const Req = (options, cb) => {
  const context = CorrelationIds.get()

  // copy the provided headers last so it overrides the values from the context
  const headers = Object.assign({}, context, options.headers || {})

  options.headers = headers

  return https.request(options, cb)
}

module.exports = Req
```

This is a wrapper around the built-in `https` module, and injects the captured correlation IDs as headers.

3. Modify `functions/get-index.js` to require our custom `https` module instead

replace 

```javascript
const AWSXRay = require('aws-xray-sdk-core')
const https = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureHTTPs(require('https'))
  : require('https')
```

with

`const https = require('../lib/https')`

4. Modify `functions/get-index.js` and update the `getRestaurants` function to use our custom `https` module

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

    const req = https(options, res => {
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

5. Run the integration tests to make sure they're still passing

`STAGE=dev REGION=us-east-1 npm run test`

6. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

7. Load up the landing page, then head to the Logz.io to check your logs. You should see that correlation IDs are propagated across both `get-index` and `get-restaurants` functions

![](/images/mod22-001.png)

8. Go to the X-Ray console and see that we haven't broken the traces either.

![](/images/mod22-002.png)

</p></details>