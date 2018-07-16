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

8. Modify `steps/when.js` to look for `content-type` instead of `Content-Type`

```javascript
const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler
  console.log(`invoking via handler function ${functionName}`)

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

9. Run the acceptance test

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
<summary><b>Modify search-restaurants test case for acceptance testing</b></summary><p>

1. Install `chance` as a dev dependency

`npm install --save-dev chance`

2. Add a file `given.js` to `steps` folder

3. Modify `steps/given.js` to the following

```javascript
const AWS = require('aws-sdk')
AWS.config.region = 'us-east-1'
const cognito = new AWS.CognitoIdentityServiceProvider()
const chance  = require('chance').Chance()

// needs number, special char, upper and lower case
const random_password = () => `${chance.string({ length: 8})}B!gM0uth`

const an_authenticated_user = async () => {
  const userpoolId = process.env.cognito_user_pool_id
  const clientId = process.env.cognito_server_client_id

  const firstName = chance.first()
  const lastName  = chance.last()
  const username  = `test-${firstName}-${lastName}-${chance.string({length: 8})}`
  const password  = random_password()
  const email     = `${firstName}-${lastName}@big-mouth.com`

  const createReq = {
    UserPoolId        : userpoolId,
    Username          : username,
    MessageAction     : 'SUPPRESS',
    TemporaryPassword : password,
    UserAttributes    : [
      { Name: "given_name",  Value: firstName },
      { Name: "family_name", Value: lastName },
      { Name: "email",       Value: email }
    ]
  }
  await cognito.adminCreateUser(createReq).promise()

  console.log(`[${username}] - user is created`)
  
  const req = {
    AuthFlow        : 'ADMIN_NO_SRP_AUTH',
    UserPoolId      : userpoolId,
    ClientId        : clientId,
    AuthParameters  : {
      USERNAME: username,    
      PASSWORD: password
    }
  }
  const resp = await cognito.adminInitiateAuth(req).promise()

  console.log(`[${username}] - initialised auth flow`)

  const challengeReq = {
    UserPoolId          : userpoolId,
    ClientId            : clientId,
    ChallengeName       : resp.ChallengeName,
    Session             : resp.Session,
    ChallengeResponses  : {
      USERNAME: username,
      NEW_PASSWORD: random_password()
    }
  }
  const challengeResp = await cognito.adminRespondToAuthChallenge(challengeReq).promise()
  
  console.log(`[${username}] - responded to auth challenge`)

  return {
    username,
    firstName,
    lastName,
    idToken: challengeResp.AuthenticationResult.IdToken
  }
}

module.exports = {
  an_authenticated_user
}
```

This introduces a new env var `cognito_server_client_id` for the tests that we need to add to the `init` step. For this to work, we also need to use a real Cognito User Pool ID.

4. Modify `steps/init.js` to use the real Cognito User Pool ID, and add the new env var, e.g.

```javascript
process.env.cognito_user_pool_id = "us-east-1_16bnZr2X5"
process.env.cognito_client_id    = "test_cognito_client_id"
process.env.cognito_server_client_id = "45ukim39plteivmrq49elgqn3v"
```

5. Add a file `tearDown.js` to the `steps` folder

6. Modify `steps/tearDown.js` to the following

```javascript
const AWS = require('aws-sdk')
AWS.config.region = 'us-east-1'
const cognito = new AWS.CognitoIdentityServiceProvider()

const an_authenticated_user = async (user) => {
  let req = {
    UserPoolId: process.env.cognito_user_pool_id,
    Username: user.username
  }
  await cognito.adminDeleteUser(req).promise()
  
  console.log(`[${user.username}] - user deleted`)
}

module.exports = {
  an_authenticated_user
}
```

7. Modify `steps/when.js` so that when we search restaurants, we would do so as an authenticated user

```javascript
const we_invoke_search_restaurants = async (user, theme) => {
  const body = JSON.stringify({ theme })
  const auth = user.idToken

  const res = 
    mode === 'handler'
      ? viaHandler({ body }, 'search-restaurants')
      : viaHttp('restaurants/search', 'POST', { body, auth })

  return res
}
```

8. Modify `test_cases/search-restaurants.js` so we would search restaurants as an authenticated user

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')
const tearDown = require('../steps/tearDown')
const given = require('../steps/given')

describe('Given an authenticated user', () => {
  let user

  before(async () => {
    await init()
    user = await given.an_authenticated_user()
  })

  after(async () => {
    await tearDown.an_authenticated_user(user)
  })

  describe(`When we invoke the POST /restaurants/search endpoint with theme 'cartoon'`, () => {
    before(async () => await init())
  
    it(`Should return an array of 4 restaurants`, async () => {
      let res = await when.we_invoke_search_restaurants(user, 'cartoon')
  
      expect(res.statusCode).to.equal(200)
      expect(res.body).to.have.lengthOf(4)
  
      for (let restaurant of res.body) {
        expect(restaurant).to.have.property('name')
        expect(restaurant).to.have.property('image')
      }
    })
  })
})
```

9. Run the acceptance tests

`npm run acceptance`

and see that all 3 tests are passing

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (454ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (312ms)

  Given an authenticated user
[test-Evelyn-Capecchi-f!!O[cz*] - user is created
[test-Evelyn-Capecchi-f!!O[cz*] - initialised auth flow
[test-Evelyn-Capecchi-f!!O[cz*] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants/search
      ✓ Should return an array of 4 restaurants (1443ms)
[test-Evelyn-Capecchi-f!!O[cz*] - user deleted


  3 passing (4s)
```

10. Run the integration tests

and see that all 3 tests are still passing as well

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via handler function get-index
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (881ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (1281ms)

  Given an authenticated user
[test-Leonard-West-N0%4KVxS] - user is created
[test-Leonard-West-N0%4KVxS] - initialised auth flow
[test-Leonard-West-N0%4KVxS] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
      ✓ Should return an array of 4 restaurants (281ms)
[test-Leonard-West-N0%4KVxS] - user deleted


  3 passing (5s)
```

</p></details>