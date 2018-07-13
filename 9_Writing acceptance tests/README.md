# Module 9: Writing acceptance tests

## Add acceptance tests

**Goal:** Write acceptance tests

<details>
<summary><b>Reuse get-index test case for acceptance testing</b></summary><p>

1. Modify `when.js` to add new functions to invoke functions remotely via API Gateway

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')
const aws4 = require('aws4')
const URL = require('url')
const http = require('superagent-promise')(require('superagent'), Promise)
const mode = process.env.TEST_MODE

const respondFrom = async (httpRes) => {
  const contentType = _.get(httpRes, 'headers.content-type', 'application/json')
  const body = 
    contentType === 'application/json'
      ? httpRes.body
      : httpRes.text

  return { 
    statusCode: httpRes.status,
    body: body,
    headers: httpRes.headers
  }
}

const signHttpRequest = (url, httpReq) => {
  const urlData = URL.parse(url)
  const opts = {
    host: urlData.hostname, 
    path: urlData.pathname
  }

  aws4.sign(opts)

  httpReq
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])

  if (opts.headers['X-Amz-Security-Token']) {
    httpReq.set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  }
}

const viaHttp = async (relPath, method, opts) => {
  const root = process.env.TEST_ROOT
  const url = `${root}/${relPath}`
  console.log(`invoking via HTTP ${method} ${url}`)

  try {
    const httpReq = http(method, url)

    const body = _.get(opts, "body")
    if (body) {      
      httpReq.send(body)
    }

    if (_.get(opts, "iam_auth", false) === true) {
      signHttpRequest(url, httpReq)
    }

    const authHeader = _.get(opts, "auth")
    if (authHeader) {
      httpReq.set('Authorization', authHeader)
    }

    const res = await httpReq
    return respondFrom(res)
  } catch (err) {
    if (err.status) {
      return {
        statusCode: err.status,
        headers: err.response.headers
      }
    } else {
      throw err
    }
  }
}
```

2. Modify `when.we_invoke_get_index` to toggle between invoking function locally and remotely

```javascript
const we_invoke_get_index = async () => {
  const res = 
    mode === 'handler' 
      ? await viaHandler({}, 'get-index')
      : await viaHttp('', 'GET')

  return res
}
```

3. Modify `init.js` to add a new `TEST_ROOT` environment variable, using the API Gateway endpoint you have deployed

```javascript
const init = async () => {
  if (initialized) {
    return
  }

  process.env.TEST_ROOT = "https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev"
```

4. Modify `package.json` to add `TEST_MODE` to integration test script, and add an acceptance test script

```json
"scripts": {
  "sls": "serverless",
  "test": "TEST_MODE=handler ./node_modules/.bin/mocha tests/test_cases --reporter spec",
  "acceptance": "TEST_MODE=http ./node_modules/.bin/mocha tests/test_cases --reporter spec"
}
```

5. Run the acceptance test

`npm run acceptance`

and see that the `get-index` function is failing

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/
    1) Should return the index page with 8 restaurants

  1) When we invoke the GET / endpoint
       Should return the index page with 8 restaurants:
     AssertionError: expected undefined to equal 'text/html; charset=UTF-8'
      at Context.it (tests/test_cases/get-index.js:13:44)
      at <anonymous>
      at process._tickCallback (internal/process/next_tick.js:188:7)
```

This is because the HTTP client `superagent` lower-cases the `Content-Type` automatically.

6. Modify `test_cases/get-index.js` to look for `content-type` instead of `Content-Type`

```javascript
expect(res.headers['content-type']).to.equal('text/html; charset=UTF-8')
```

7. Modify `functions/get-index.js` to return `content-type` header instead of `Content-Type`

```javascript
const response = {
  statusCode: 200,
  headers: {
    'content-type': 'text/html; charset=UTF-8'
  },
  body: html
}
```

8. Run the acceptance test

`npm run acceptance`

and see that the `get-index` function is now passing

</p></details>

<details>
<summary><b>Reuse get-restaurants test case for acceptance testing</b></summary><p>

1. Modify `when.we_invoke_get_restaurants` to toggle between invoking function locally and remotely

```javascript
const we_invoke_get_restaurants = async () => {
  const res =
    mode === 'handler' 
      ? await viaHandler({}, 'get-restaurants')
      : await viaHttp('restaurants', 'GET', { iam_auth: true })

  return res
}
```

2. Run the acceptance test

`npm run acceptance`

and see that both `get-index` and `get-restaurants` tests are passing

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (632ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (380ms)
```

</p></details>

<details>
<summary><b>Reuse search-restaurants test case for acceptance testing</b></summary><p>


</p></details>