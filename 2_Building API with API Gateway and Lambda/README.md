# Module 2: Building API with API Gateway and Lambda

## Returning HTML from Lambda

**Goal:** Create and deploy an API that returns HTML instead of JSON

<details>
<summary><b>Create a serverless project</b></summary><p>

1. Create a directory for your serverless project.

    ```
    mkdir workshop
    cd workshop
    ```

2. Initialise the project:
    
    `npm init`
    
    Name the project accordingly and you can accept the rest of the defaults.

3. Install the `Serverless` framework as dev dependency.

    `npm install --save-dev serverless`

    Add `sls` to npm scripts by editing your `package.json` so your `scripts` section looks like this:

    ```json
      "scripts": {
        "sls": "serverless"
      },
    ```
    
    Now you can run serverless using `npm run sls [-- <args>...]`

4. Create nodejs Serverless project using one of the default templates:

    `npm run sls -- create --template aws-nodejs`

    See more information about `serverless create` command on [CLI documentation](https://serverless.com/framework/docs/providers/aws/cli-reference/create/) page.

5. Delete the `handler.js` file at the root
</p></details>

<details>
<summary><b>Add an API function</b></summary><p>

1. Modify the `serverless.yml` to the following, change the name of `service` to `workshop-` followed by your name, e.g. `workshop-yancui`

```yml
service: workshop-yancui

provider:
  name: aws
  runtime: nodejs8.10

functions:
  get-index:
    handler: functions/get-index.handler
    events:
      - http:
          path: /
          method: get
```

2. Add a folder called `functions`

3. Add a file `get-index.js` under the newly created `functions` folder

4. Modify the `get-index.js` file to the following:

```javascript
const fs = require("fs")

let html

function loadHtml () {
  if (!html) {
    console.log('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    console.log('loaded')
  }
  
  return html
}

module.exports.handler = async (event, context) => {
  const html = loadHtml()
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

</p></details>

<details>
<summary><b>Add the static HTML</b></summary><p>

1. Add a folder called `static`

2. Add a file `index.html` under the newly created `static` folder

3. Modify the `index.html` file to the following:

```xml
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>
    
    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;
      }

      .column-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: -webkit-box;
        display: -moz-box;
        display: -ms-flexbox;
        display: -webkit-flex;
        display: flex;
        flex-flow: column;
        justify-content: center;
      }

      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        justify-content: center;
      }

      input {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }

      button {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
    </style>

    <script>
    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="search()">Find Restaurants</button>
        </li>
      </ul>
  </div>
  </body>

</html>
```

</p></details>

<details>
<summary><b>Deploy the serverless project</b></summary><p>

1. Run `deploy` command:

    `npm run sls -- deploy`

    See more information about `deploy` command on [CLI documentation](https://serverless.com/framework/docs/providers/aws/cli-reference/deploy/) page.

2. In the output you should see something like this:

```
endpoints:
  GET - https://xxxxx.execute-api.us-east-1.amazonaws.com/dev/
```

Load the endpoint in the browser and see something like the following:

![](/images/mod02-001.png)


</p></details>

## Creating the restaurants API

**Goal:** Create and deploy the `/restaurants` endpoint

<details>
<summary><b>Add a function to return restaurants</b></summary><p>

1. Add a file `get-restaurants.js` file to the `functions` folder

2. Install the `aws-sdk` package

`npm install --save aws-sdk`

3. Modify `get-restaurants.js` to the following:

```javascript
const AWS = require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()

const defaultResults = process.env.defaultResults || 8
const tableName = process.env.restaurants_table

const getRestaurants = async (count) => {
  const req = {
    TableName: tableName,
    Limit: count
  }

  const resp = await dynamodb.scan(req).promise()
  return resp.Items
}

module.exports.handler = async (event, context) => {
  const restaurants = await getRestaurants(defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
}
```

This function depends on two environment variables:

* `defaultResults` [optional] : how many restaurants to return

* `restaurants_table` [required] : name of the restaurants DynamoDB table

4. Modify `serverless.yml` to add the new function and its environment variables:

```yml
get-restaurants:
  handler: functions/get-restaurants.handler
  events:
    - http:
        path: /restaurants/
        method: get
  environment:
    restaurants_table: restaurants-yancui
```
</p></details>

<details>
<summary><b>Configure IAM permission</b></summary><p>

1. Modify `serverless.yml` and add an `iamRoleStatements` section under `provider`:

```yml
provider:
  name: aws
  runtime: nodejs8.10

  iamRoleStatements:
    - Effect: Allow
      Action: dynamodb:scan
      Resource:
        Fn::GetAtt:
          - restaurantsTable
          - Arn
```

This adds the `dynamodb:scan` permission to the Lambda execution role.

2. Deploy the project

`npm run sls -- deploy`
</p></details>

<details>
<summary><b>Add a DynamoDB table in the serverless.yml</b></summary><p>

1. Add DynamoDB table to `serverless.yml`

```yml
resources:
  Resources:
    restaurantsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: restaurants-yancui
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

When you deploy the serverless project next, the DynamoDB table would be provisioned.

2. Add a file `seed-restaurants.js` to the project root

3. Modify `seed-restaurants.js` to the following (make sure you change `restaurants-yancui` to your table):

```javascript
const AWS = require('aws-sdk')
AWS.config.region = 'us-east-1'
const dynamodb = new AWS.DynamoDB.DocumentClient()

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

let putReqs = restaurants.map(x => ({
  PutRequest: {
    Item: x
  }
}))

let req = {
  RequestItems: {
    'restaurants-yancui': putReqs
  }
}
dynamodb.batchWrite(req).promise().then(() => console.log("all done"))
```

4. Run the `seed-restaurants.js` script

`node seed-restaurants.js`

</p></details>

## Displaying restaurants in the landing page

**Goal:** Displays the restaurants in `index.html`

<details>
<summary><b>Update index.html to allow templating with Mustache</b></summary><p>

1. Modify `index.html` to the following:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>
    
    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;
      }
      .restaurantsDiv {
        background-color: #ffffff;
        width: 100%;
        height: auto;
      }
      .dayOfWeek {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 32px;
        padding: 10px;
        height: auto;
        display: flex;
        justify-content: center;
      }
      .column-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: column;
        flex-wrap: wrap;
        justify-content: center;
      }
      .row-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .restaurant {
        background-color: #00a8f7;
        border-radius: 10px;
        padding: 5px;
        height: auto;
        width: auto;
        margin-left: 40px;
        margin-right: 40px;
        margin-top: 15px;
        margin-bottom: 0px;
        display: flex;
        justify-content: center;
      }
      .restaurant-name {
        font-size: 24px;
        font-family:Arial, Helvetica, sans-serif;
        color: #ffffff;
        padding: 10px;
        margin: 0px;
      }
      .restaurant-image {
        padding-top: 0px;
        margin-top: 0px;
      }
      input {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
      button {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
    </style>

    <script>      
    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="search()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul class="row-container">
              {{#restaurants}}
              <li class="restaurant">
                <ul class="column-container">
                    <li class="item restaurant-name">{{name}}</li>
                    <li class="item restaurant-image">
                      <img src="{{image}}">
                    </li>
                </ul>
              </li>
              {{/restaurants}}
            </ul> 
          </div>
        </li>
      </ul>
  </div>
  </body>

</html>
```
</p></details>

<details>
<summary><b>Render the index.html with restaurants</b></summary><p>

1. Install `mustache` as dependency

`npm install --save mustache`

2. Install `superagent` as dependency

`npm install --save superagent`

3. Install `superagent-promise` as dependency

`npm install --save superagent-promise`

4. Modify `get-index.js` to the following:

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)

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
  return (await http.get(restaurantsApiRoot)).body
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

After this change, the `get-index` function needs the `restaurants_api` environment variable to know where the `/restaurants` endpoint is.

5. Modify the `serverless.yml` to add an environment variable to the `get-index` function:

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
```

6. Deploy the project

`npm run sls -- deploy -r us-east-1 -s dev`

Reload the `index.html` and you should see something like the following

![](/images/mod02-002.png)

</p></details>