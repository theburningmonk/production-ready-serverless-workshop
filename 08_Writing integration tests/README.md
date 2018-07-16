# Module 8: Writing integration tests

## Add integration tests

**Goal:** Write integration tests

<details>
<summary><b>Prepare tests</b></summary><p>

1. Add a `tests` folder to the project root

2. Add a `test_cases` folder under `tests`

3. Add a `steps` folder under `tests`

4. Install `chai` as a dev dependency

`npm install --save-dev chai`

5. Install `mocha` as a dev dependency

`npm install --save-dev mocha`

6. Install `cheerio` as a dev dependency

`npm install --save-dev cheerio`

7. Install `awscred` as a dependency

`npm install --save awscred`

8. Install `lodash` as a dependency

`npm install --save lodash`

</p></details>

<details>
<summary><b>Add test case for get-index</b></summary><p>

1. Add `get_index.js` file under `test_cases`

2. Modify `get_index.js` to the following

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')

describe(`When we invoke the GET / endpoint`, () => {
  it(`Should return the index page with 8 restaurants`, async () => {
    const res = await when.we_invoke_get_index()

    expect(res.statusCode).to.equal(200)
    expect(res.headers['Content-Type']).to.equal('text/html; charset=UTF-8')
    expect(res.body).to.not.be.null

    const $ = cheerio.load(res.body)
    const restaurants = $('.restaurant', '#restaurantsUl')
    expect(restaurants.length).to.equal(8)
  })
})
```

3. Add `when.js` file under `steps`

4. Modify `when.js` to the following

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')

const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.Content-Type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}

const we_invoke_get_index = () => viaHandler({}, 'get-index')

module.exports = {
  we_invoke_get_index
}
```

5. Modify `test_cases/get-index.js` to require the `when` module

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')
const when = require('../steps/when')

describe(`When we invoke the GET / endpoint`, () => {
```

6. Modify the `package.json` and add a `test` script

```json
"scripts": {
  "sls": "serverless",
  "test": "./node_modules/.bin/mocha tests/test_cases --reporter spec"
}
```

7. Run the integration test

`npm run test`

and see that the test fails with the error 

```
When we invoke the GET / endpoint
loading index.html...
loaded
(node:51636) UnhandledPromiseRejectionWarning: TypeError: Parameter "url" must be a string, not undefined
```

The `get-index` function needs a number of environment variables.

8. Add `init.js` under `steps` folder

9. Modify `init.js` to the following (using the deployed API Gateway url for the `restaurants_api` environment variable, and use the DynamoDB table you created)

```javascript
const { promisify } = require('util')
const awscred = require('awscred')

let initialized = false

const init = async () => {
  if (initialized) {
    return
  }

  process.env.restaurants_api      = "https://xxx.execute-api.us-east-1.amazonaws.com/dev/restaurants"
  process.env.restaurants_table    = "restaurants-yancui"
  process.env.AWS_REGION           = "us-east-1"
  process.env.cognito_user_pool_id = "test_cognito_user_pool_id"
  process.env.cognito_client_id    = "test_cognito_client_id"
  
  const { credentials } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID     = credentials.accessKeyId
  process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey

  console.log('AWS credential loaded')

  initialized = true
}

module.exports = {
  init
}
```

10. Modify `test_cases/get-index.js` to require the `init` module

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')
const when = require('../steps/when')
const { init } = require('../steps/init')

describe(`When we invoke the GET / endpoint`, () => {
```

11. Modify `test_cases/get-index.js` to add a `before` case

```javascript
describe(`When we invoke the GET / endpoint`, () => {
  before(async () => await init())

  it(`Should return the index page with 8 restaurants`, async () => {
```

12. Run the integration test

`npm run test`

and see that the test fails with the error 

```
When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
(node:51753) UnhandledPromiseRejectionWarning: Error: "value" required in setHeader("X-Amz-Security-Token", value)
```

13. Modify `functions/get-index.js` to only set the `X-Amz-Security-Token` header if it's applicable

```javascript
const getRestaurants = async () => {
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname, 
    path: url.pathname
  }

  aws4.sign(opts)

  const httpReq = http
    .get(restaurantsApiRoot)
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])
    
  if (opts.headers['X-Amz-Security-Token']) {
    httpReq.set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  }

  return (await httpReq).body
}
```

14. Run the integration test

`npm run test`

and see that the test passes

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (449ms)


  1 passing (467ms)
```

</p></details>

If you find that some tests fails sporadically, they might just need longer to run. Try raising the test timeout with `--timeout 5000`, the default timeout is 2s.

## Exercises

<details>
<summary><b>Add test case for get-restaurants</b></summary><p>

1. Add `get-restaurants.js` under `test_cases`

2. Modify `get-restaurants.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the GET /restaurants endpoint`, () => {
  before(async () => await init())

  it(`Should return an array of 8 restaurants`, async () => {
    let res = await when.we_invoke_get_restaurants()

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(8)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Modify `when.js` to add a `we_invoke_get_restaurants` function

```javascript
const we_invoke_get_restaurants = () => viaHandler({}, 'get-restaurants')

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants
}
```

4. Run the integration test

`npm run test`

and see that the tests pass

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (371ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (451ms)


  2 passing (839ms)
```

</p></details>

<details>
<summary><b>Add test case for search-restaurants</b></summary><p>

1. Add `search-restaurants.js` under `test_cases`

2. Modify `search-restaurants.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the POST /restaurants/search endpoint with theme 'cartoon'`, () => {
  before(async () => await init())

  it(`Should return an array of 4 restaurants`, async () => {
    let res = await when.we_invoke_search_restaurants('cartoon')

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(4)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Modify `when.js` to add a `we_invoke_search_restaurants` function

```javascript
const we_invoke_search_restaurants = theme => {
  let event = { 
    body: JSON.stringify({ theme })
  }
  return viaHandler(event, 'search-restaurants')
}

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants
}
```

4. Run the integration test

`npm run test`

and see that the tests pass

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (435ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (440ms)

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
    ✓ Should return an array of 4 restaurants (249ms)


  3 passing (1s)
```

</p></details>

<details>
<summary><b>What other test cases would you add?</b></summary><p>

</p></details>