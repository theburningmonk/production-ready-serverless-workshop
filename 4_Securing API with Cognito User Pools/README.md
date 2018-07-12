# Module 4: Securing API with Cognito User Pools

## Create POST /restaurants/search endpoint

**Goal:** There is a `/restaurants/search` endpoint

<details>
<summary><b>Add search-restaurants function</b></summary><p>

1. Modify `serverless.yml` to add a `search-restaurants` function

```yml
  search-restaurants:
    handler: functions/search-restaurants.handler
    events:
      - http:
          path: /restaurants/search
          method: post
    environment:
      restaurants_table: restaurants-yancui
```

2. Add `search-restaurants.js` to the `functions` folder

3. Modify `search-restaurants.js` to the following:

```javascript
const AWS = require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()

const defaultResults = process.env.defaultResults || 8
const tableName = process.env.restaurants_table

const findRestaurantsByTheme = async (theme, count) => {
  const req = {
    TableName: tableName,
    Limit: count,
    FilterExpression: "contains(themes, :theme)",
    ExpressionAttributeValues: { ":theme": theme }
  }

  const resp = await dynamodb.scan(req).promise()
  return resp.Items
}

module.exports.handler = async (event, context) => {
  const req = JSON.parse(event.body)
  const theme = req.theme
  const restaurants = await findRestaurantsByTheme(theme, defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
}
```

4. Deploy the serverless project

`npm run sls -- deploy -s dev -r eu-central-1`

</p></details>

## Securing search-restaurants endpoint with Cognito User Pools

**Goal:** The `/restaurants/search` endpoint is protected by Cognito User Pools



