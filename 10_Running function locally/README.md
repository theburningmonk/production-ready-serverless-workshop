# Module 10: Running function locally

There is currently an open PR to support `async` handler functions :

https://github.com/serverless/serverless/pull/4912

with the Serverless framework.

In the mean time if you run

`npm run sls -- invoke local -f get-restaurants -s dev -r us-east-1`

it'll still exercise the function locally, but it won't report the response of the function.

One workaround is the modify the handler signature to include the `callback` function as well.

For example, if you change `get-restaurants.handler` to

```javascript
module.exports.handler = async (event, context, callback) => {
  const restaurants = await getRestaurants(defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  callback(null, response)
}
```

and run 

`npm run sls -- invoke local -f get-restaurants -s dev -r us-east-1`

You will see the following as output

```
{
    "statusCode": 200,
    "body": "[{\"name\":\"Fangtasia\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/fangtasia.png\",\"themes\":[\"true blood\"]},{\"name\":\"Shoney's\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/shoney's.png\",\"themes\":[\"cartoon\",\"rick and morty\"]},{\"name\":\"Freddy's BBQ Joint\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/freddy's+bbq+joint.png\",\"themes\":[\"netflix\",\"house of cards\"]},{\"name\":\"Pizza Planet\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/pizza+planet.png\",\"themes\":[\"netflix\",\"toy story\"]},{\"name\":\"Leaky Cauldron\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/leaky+cauldron.png\",\"themes\":[\"movie\",\"harry potter\"]},{\"name\":\"Lil' Bits\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/lil+bits.png\",\"themes\":[\"cartoon\",\"rick and morty\"]},{\"name\":\"Fancy Eats\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/fancy+eats.png\",\"themes\":[\"cartoon\",\"rick and morty\"]},{\"name\":\"Don Cuco\",\"image\":\"https://d2qt42rcwzspd6.cloudfront.net/manning/don%20cuco.png\",\"themes\":[\"cartoon\",\"rick and morty\"]}]"
}
```