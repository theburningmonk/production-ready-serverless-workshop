# Module 3: Securing API with IAM authorization

## Securing get-restaurants endpoint with IAM-authorization

**Goal:** Protect the `get-restaurants` endpoint is with IAM

<details>
<summary><b>Use AWS_IAM authorizer for get-restaurants endpoint</b></summary><p>

1. Modify `serverless.yml` to add `authorizer: aws_iam` to the `get-restaurants` endpoint

```yml
get-restaurants:
  handler: functions/get-restaurants.handler
  events:
    - http:
        path: /restaurants/
        method: get
        authorizer: aws_iam
  environment:
    restaurants_table: restaurants-yancui
```
</p></details>

<details>
<summary><b>Add execute-api:Invoke to the IAM execution role</b></summary><p>

1. Install `serverless-pseudo-parameters` as dev dependency

`npm install --save-dev serverless-pseudo-parameters`

Take a look at the [github repo](https://github.com/svdgraaf/serverless-pseudo-parameters) to see how this plugin works

2. Modify `serverless.yml` to add a `plugins` section

```yml
plugins:
  - serverless-pseudo-parameters
```

3. Modify `serverless.yml` to update the `iamRoleStatements` property

```yml
iamRoleStatements:
  - Effect: Allow
    Action: dynamodb:scan
    Resource:
      Fn::GetAtt:
        - restaurantsTable
        - Arn
  - Effect: Allow
    Action: execute-api:Invoke
    Resource: arn:aws:execute-api:#{AWS::Region}:#{AWS::AccountId}:*/*/GET/restaurants
```

</p></details>

<details>
<summary><b>Signing request with IAM role</b></summary><p>

1. Install `aws4` as dependency

`npm install --save aws4`

2. Modify `get-index.js` to the following

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']

let html

function loadHtml () {
  if (!html) {
    console.log('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    console.log('loaded')
  }
  
  return html
}

const getRestaurants = async () => {
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname, 
    path: url.pathname
  }

  aws4.sign(opts)

  return (await http
    .get(restaurantsApiRoot)
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])
    .set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  ).body
}

module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const html = Mustache.render(template, { dayOfWeek, restaurants })
  const response = {
    statusCode: 200,
    headers: {
      'Content-Type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

3. Deploy the project

`npm run sls -- deploy -r us-east-1 -s dev`

Reload the `index.html` and it should still work. If you curl the `/restaurants` endpoint you should see

```
{
  message: "Missing Authentication Token"
}
```

</p></details>