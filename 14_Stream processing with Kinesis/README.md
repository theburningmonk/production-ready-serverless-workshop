# Module 14: Stream processing with Kinesis

## Process events in realtime with Kinesis and Lambda

**Goal:** Implemented the order flow using Kinesis

<details>
<summary><b>Add place-order function</b></summary><p>

1. Modify `serverless.yml` to add a new `place-order` function

```yml
place-order:
  handler: functions/place-order.handler
  events:
    - http:
        path: /orders
        method: post
        authorizer:
          arn: arn:aws:cognito-idp:#{AWS::Region}:#{AWS::AccountId}:userpool/${ssm:/workshop-yancui/dev/cognito_user_pool_id}
  environment:
    order_events_stream: ${ssm:/workshop-yancui/dev/stream_name}
```

Notice that this new function references a Kinesis stream, whose name will be parameterised in SSM parameter store. This function also uses the same Cognito User Tool for authorization, as it'll be called directly by the client app.

2. Modify `serverless.yml` to add the Kinesis stream as a new resource under the `resources` section

```yml
orderEventsStream:
  Type: AWS::Kinesis::Stream
  Properties: 
    Name: ${ssm:/workshop-yancui/dev/stream_name}
    ShardCount: 1
```

3. Modify `serverless.yml` to add the permission for `kinesis:PutRecord` under `provider.iamRoleStatements`

```yml
- Effect: Allow
  Action: kinesis:PutRecord
  Resource: 
    Fn::GetAtt:
      - orderEventsStream
      - Arn
```

4. Move `chance` from a dev dependency to application dependency

5. Add a file `place-order.js` to the `functions` folder

6. Modify `place-order.js` to the following

```javascript
const _          = require('lodash')
const AWS        = require('aws-sdk')
const kinesis    = new AWS.Kinesis()
const chance     = require('chance').Chance()
const streamName = process.env.order_events_stream

const UNAUTHORIZED = {
  statusCode: 401,
  body: "unauthorized"
}

module.exports.handler = async (event, context) => {
  const restaurantName = JSON.parse(event.body).restaurantName

  const userEmail = _.get(event, 'requestContext.authorizer.claims.email')
  if (!userEmail) {
    console.error('user email is not found')
    return UNAUTHORIZED
  }

  const orderId = chance.guid()
  console.log(`placing order ID [${orderId}] to [${restaurantName}] for user [${userEmail}]`)

  const data = {
    orderId,
    userEmail,
    restaurantName,
    eventType: 'order_placed'
  }

  const req = {
    Data: JSON.stringify(data), // the SDK would base64 encode this for us
    PartitionKey: orderId,
    StreamName: streamName
  }

  await kinesis.putRecord(req).promise()

  console.log(`published 'order_placed' event into Kinesis`)

  const response = {
    statusCode: 200,
    body: JSON.stringify({ orderId })
  }

  return response
}
```

7. Go to EC2 console

8. Go to Parameter Store (bottom left)

9. Add a new parameter `/${service-name}/dev/stream_name` with the value `orders-dev-` followed by your name, e.g. `orders-dev-yancui`

![](/images/mod14-001.png)

</p></details>

<details>
<summary><b>Add integration test for place-order function</b></summary><p>

1. Add a file `place-order.js` to `test_cases` folder

2. Modify `steps/init.js` to load `stream_name` from SSM parameter store and set the `order_events_stream` environment variable (used by the `place-order` function)

```javascript
const params = await getParameters([
  'stream_name',    
  'table_name', 
  'cognito_user_pool_id', 
  'cognito_web_client_id',
  'cognito_server_client_id',
  'url'
])
```

```javascript
process.env.order_events_stream = params.stream_name
```

3. Install `mock-aws` as dev dependency

`npm install --save-dev mock-aws`

4. Modify `test_cases/place-order.js` to the following

```javascript
const { expect } = require('chai')
const when = require('../steps/when')
const given = require('../steps/given')
const tearDown = require('../steps/tearDown')
const { init } = require('../steps/init')
const AWS = require('mock-aws')

describe('Given an authenticated user', () => {
  let user

  before(async () => {
    await init()
    user = await given.an_authenticated_user()
  })

  after(async () => {
    await tearDown.an_authenticated_user(user)
  })

  describe(`When we invoke the POST /orders endpoint`, () => {
    let isEventPublished = false
    let resp
  
    before(async () => {
      AWS.mock('Kinesis', 'putRecord', (req) => {
        isEventPublished = 
          req.StreamName === process.env.order_events_stream &&
          JSON.parse(req.Data).eventType === 'order_placed'

        return {
          promise: async () => {}
        }
      })

      resp = await when.we_invoke_place_order(user, 'Fangtasia')
    })

    after(() => AWS.restore('Kinesis', 'putRecord'))
  
    it(`Should return 200`, async () => {
      expect(resp.statusCode).to.equal(200)
    })
  
    it(`Should publish a message to Kinesis stream`, async () => {
      expect(isEventPublished).to.be.true
    })
  })
})
```

5. Modify `steps/when.js` to add a new `we_invoke_place_order` function

```javascript
const we_invoke_place_order = async (user, restaurantName) => {
  const body = JSON.stringify({ restaurantName })
  const requestContext = {
    authorizer: {
      claims: {
        email: `${user.username}@test.com`
      }
    }
  }
  const auth = user.idToken

  const res = 
    mode === 'handler'
      ? await viaHandler({ body, requestContext }, 'place-order')
      : await viaHttp('orders', 'POST', { body, auth })
  
  return res
}

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order
}
```

6. Run integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all 5 tests are passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (1277ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (176ms)

  Given an authenticated user
[test-Verna-Chamberlain] - user is created
[test-Verna-Chamberlain] - initialised auth flow
[test-Verna-Chamberlain] - responded to auth challenge
    When we invoke the POST /orders endpoint
placing order ID [c861c9fa-b1ed-52a6-9e60-24ba455e8a4d] to [Fangtasia] for user [test-Verna-Chamberlain@test.com]
published 'order_placed' event into Kinesis
      ✓ Should return 200
      ✓ Should publish a message to Kinesis stream
[test-Verna-Chamberlain] - user deleted

  Given an authenticated user
[test-Jack-Paci] - user is created
[test-Jack-Paci] - initialised auth flow
[test-Jack-Paci] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
      ✓ Should return an array of 4 restaurants (91ms)
[test-Jack-Paci] - user deleted


  5 passing (4s)
```

7. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

</p></details>

<details>
<summary><b>Add acceptance test for place-order function</b></summary><p>

When executing the deployed `place-order` function via API Gateway, the function would publish an `order_placed` event to the real Kinesis stream.

To verify that the event is published as expected, you have some options:

* If events are streamed and backed up in S3 (e.g. via Kinesis Firehose, so all events are recorded in a persistent storage), then you can poll S3 for new events. However, this approach can be time-consuming depending on the Firehose configuration, if data are batched in 5 mins intervals then this approach becomes infeasible.

* If events are streamed to another BI platform, such as Google Big Query, in real time, then that is a far better option - to query Big Query for the expected event.

* You can use the AWS SDK to fetch Kinesis records with `kinesis.getRecords`, but this is clumsy as it's a multi-step process that requires you to describe shards and get shard iterator first, and when there are more than 1 shard in the stream it also becomes infeasible to keep polling every shard until you have found the expected event.

For this workshop, we'll take a short-cut and only validate Kinesis was called when executing as an integration test.

1. Modify `test_cases/place-order.js` so the test case no longer validates Kinesis event is published when running as an acceptance test

```javascript
it(`Should return 200`, async () => {
  expect(resp.statusCode).to.equal(200)
})

if (process.env.TEST_MODE === 'handler') {
  it(`Should publish a message to Kinesis stream`, async () => {
    expect(isEventPublished).to.be.true
  })
}
```

2. Run acceptance test

`STAGE=dev REGION=us-east-1 npm run acceptance`

and see that all 4 tests are passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (550ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (380ms)

  Given an authenticated user
[test-Mario-Hughes-RfIs4]%e] - user is created
[test-Mario-Hughes-RfIs4]%e] - initialised auth flow
[test-Mario-Hughes-RfIs4]%e] - responded to auth challenge
    When we invoke the POST /orders endpoint
invoking via HTTP POST https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/orders
      ✓ Should return 200
[test-Mario-Hughes-RfIs4]%e] - user deleted

  Given an authenticated user
[test-Walter-Bravo-lT$Q^nuk] - user is created
[test-Walter-Bravo-lT$Q^nuk] - initialised auth flow
[test-Walter-Bravo-lT$Q^nuk] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants/search
      ✓ Should return an array of 4 restaurants (394ms)
[test-Walter-Bravo-lT$Q^nuk] - user deleted


  4 passing (6s)
```

</p></details>

<details>
<summary><b>Update web client to support placing order</b></summary><p>

1. Modify `static/index.html` to the following

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>

    <script src="https://sdk.amazonaws.com/js/aws-sdk-2.149.0.min.js"></script>
    <script src="https://d2qt42rcwzspd6.cloudfront.net/manning/aws-cognito-sdk.min.js"></script>
    <script src="https://d2qt42rcwzspd6.cloudfront.net/manning/amazon-cognito-identity.min.js"></script>
    <script src="https://code.jquery.com/jquery-3.2.1.min.js" 
            integrity="sha256-hwg4gsxgFZhOsEEamdOYGBf13FyQuiTwlAQgxVSNgt4="
            crossorigin="anonymous"></script>
    <script src="https://code.jquery.com/ui/1.12.1/jquery-ui.min.js" 
            integrity="sha384-Dziy8F2VlJQLMShA6FHWNul/veM9bCkRUaLqr199K94ntO5QUrLJBEbYegdSkkqX" 
            crossorigin="anonymous"></script>
    <link rel="stylesheet" href="https://code.jquery.com/ui/1.12.1/themes/base/jquery-ui.css">

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
        padding: 5px;
        margin: 5px;
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
      .row-container-left {
        list-style: none;
        display: flex;
        flex-flow: row;
        justify-content: flex-start;
      }
      .menu-text {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 24px;
        font-weight: bold;
        color: white;
      }
      .text-trail-space {
        margin-right: 10px;
      }
      .hidden {
        display: none;
      }

      lable, button, input {
        display:block;
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
      
      fieldset { 
        padding:0; 
        border:0; 
        margin-top:25px; 
      }

    </style>

    <script>
      const AWS_REGION = '{{awsRegion}}';
      const COGNITO_USER_POOL_ID = '{{cognitoUserPoolId}}';
      const CLIENT_ID = '{{cognitoClientId}}';
      const SEARCH_URL = '{{& searchUrl}}';
      const PLACE_ORDER_URL = '{{& placeOrderUrl}}';

      var regDialog, regForm;
      var verifyDialog;
      var regCompleteDialog;
      var signInDialog;
      var userPool, cognitoUser;
      var idToken;

      function toggleSignOut (enable) {
        enable === true ? $('#sign-out').show() : $('#sign-out').hide();
      }

      function toggleSignIn (enable) {
        enable === true ? $('#sign-in').show() : $('#sign-in').hide();
      }

      function toggleRegister (enable) {
        enable === true ? $('#register').show() : $('#register').hide();
      }

      function init() {
        AWS.config.region = AWS_REGION;
        AWSCognito.config.region = AWS_REGION;

        var data = { 
          UserPoolId : COGNITO_USER_POOL_ID, 
          ClientId : CLIENT_ID
        };
        userPool = new AWSCognito.CognitoIdentityServiceProvider.CognitoUserPool(data);
        cognitoUser = userPool.getCurrentUser();

        if (cognitoUser != null) {          
          cognitoUser.getSession(function(err, session) {
            if (err) {
                alert(err);
                return;
            }

            idToken = session.idToken.jwtToken;
            console.log('idToken: ' + idToken);
            console.log('session validity: ' + session.isValid());
          });

          toggleSignOut(true);
          toggleSignIn(false);
          toggleRegister(false);
        } else {
          toggleSignOut(false);
          toggleSignIn(true);
          toggleRegister(true);
        }
      }

      function addUser() {
        var firstName = $("#first-name")[0].value;
        var lastName = $("#last-name")[0].value;
        var username = $("#username")[0].value;
        var password = $("#password")[0].value;
        var email = $("#email")[0].value;

        var attributeList = [
          new AWSCognito.CognitoIdentityServiceProvider.CognitoUserAttribute({ 
            Name : 'email', Value : email
          }),
          new AWSCognito.CognitoIdentityServiceProvider.CognitoUserAttribute({ 
            Name : 'given_name', Value : firstName
          }),
          new AWSCognito.CognitoIdentityServiceProvider.CognitoUserAttribute({ 
            Name : 'family_name', Value : lastName
          }),
        ];

        userPool.signUp(username, password, attributeList, null, function(err, result){
          if (err) {
            alert(err);
            return;
          }
          cognitoUser = result.user;
          console.log('user name is ' + cognitoUser.getUsername());

          regDialog.dialog("close");
          verifyDialog.dialog("open");
        });
      }

      function confirmUser() {
        var verificationCode = $("#verification-code")[0].value;
        cognitoUser.confirmRegistration(verificationCode, true, function(err, result) {
          if (err) {
            alert(err);
            return;
          }
          console.log('verification call result: ' + result);

          verifyDialog.dialog("close");
          regCompleteDialog.dialog("open");
        });
      }

      function authenticateUser() {
        var username = $("#sign-in-username")[0].value;
        var password = $("#sign-in-password")[0].value;

        var authenticationData = {
          Username : username,
          Password : password,
        };
        var authenticationDetails = new AWSCognito.CognitoIdentityServiceProvider.AuthenticationDetails(authenticationData);
        var userData = {
          Username : username,
          Pool : userPool
        };
        var cognitoUser = new AWSCognito.CognitoIdentityServiceProvider.CognitoUser(userData);

        cognitoUser.authenticateUser(authenticationDetails, {
          onSuccess: function (result) {
            console.log('access token : ' + result.getAccessToken().getJwtToken());
            /*Use the idToken for Logins Map when Federating User Pools with Cognito Identity or when passing through an Authorization Header to an API Gateway Authorizer*/
            idToken = result.idToken.jwtToken;
            console.log('idToken : ' + idToken);

            signInDialog.dialog("close");
            toggleRegister(false);
            toggleSignIn(false);
            toggleSignOut(true);
          },

          onFailure: function(err) {
            alert(err);
          }
        });
      }

      function signOut() {
        if (cognitoUser != null) {
          cognitoUser.signOut();
          toggleRegister(true);
          toggleSignIn(true);
          toggleSignOut(false);
        }
      }

      function searchRestaurants() {
        var theme = $("#theme")[0].value;

        var xhr = new XMLHttpRequest();
        xhr.open('POST', SEARCH_URL, true);
        xhr.setRequestHeader("Content-Type", "application/json");
        xhr.setRequestHeader("Authorization", idToken);
        xhr.send(JSON.stringify({ theme }));
        
        xhr.onreadystatechange = function (e) {
          if (xhr.readyState === 4 && xhr.status === 200) {
            var restaurants = JSON.parse(xhr.responseText);
            var restaurantsList = $("#restaurantsUl");
            restaurantsList.empty();

            for (var restaurant of restaurants) {
              restaurantsList.append(`
              <li class="restaurant">
                <ul class="column-container" onclick='placeOrder("${restaurant.name}")'>
                    <li class="item restaurant-name">${restaurant.name}</li>
                    <li class="item restaurant-image">
                      <img src="${restaurant.image}">
                    </li>
                </ul>
              </li>
              `);
            }

          } else if (xhr.readyState === 4) {
            alert(xhr.responseText);
          }
        };
      }

      function placeOrder(restaurantName) {
        var xhr = new XMLHttpRequest();
        xhr.open('POST', PLACE_ORDER_URL, true);
        xhr.setRequestHeader("Content-Type", "application/json");
        xhr.setRequestHeader("Authorization", idToken);
        xhr.send(JSON.stringify({ restaurantName }));

        xhr.onreadystatechange = function (e) {
          if (xhr.readyState === 4 && xhr.status === 200) {
            alert("your order has been placed, we'll let you know once it's been accepted by the restaurant!");
          } else if (xhr.readyState === 4) {
            alert(xhr.responseText);
          }
        };
      }

      $(document).ready(function() {
        regDialog = $("#reg-dialog-form").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            "Create an account": addUser,
            Cancel: function() {
              regDialog.dialog("close");
            }
          },
          close: function() {
            regForm[0].reset();
          }
        });

        regForm = regDialog.find("form").on("submit", function(event) {
          event.preventDefault();
          addUser();
        });
        
        $("#register").on("click", function() {
          regDialog.dialog("open");
        });

        verifyDialog = $("#verify-dialog-form").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            "Confirm registration": confirmUser,
            Cancel: function() {
              verifyDialog.dialog("close");
            }
          },
          close: function() {
            $(this).dialog("close");
          }
        });

        regCompleteDialog = $("#registered-message").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            Ok: function() {
              $(this).dialog("close");
            }
          }
        });

        signInDialog = $("#sign-in-form").dialog({
          autoOpen: false,
          modal: true,
          buttons: {
            "Sign in": authenticateUser,
            Cancel: function() {
              signInDialog.dialog("close");
            }
          },
          close: function() {
            $(this).dialog("close");
          }
        });

        $("#sign-in").on("click", function() {
          signInDialog.dialog("open");
        });

        $("#sign-out").on("click", function() {
          signOut();
        })

        init();
      });

    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li>
          <ul class="row-container-left">
            <li id="register" class="item text-trail-space hidden">
              <a class="menu-text" href="#">Register</a>
            </li>
            <li id="sign-in" class="item menu-text text-trail-space hidden">
              <a class="menu-text" href="#">Sign in</a>
            </li>
            <li id="sign-out" class="item menu-text text-trail-space hidden">
              <a class="menu-text" href="#">Sign out</a>
            </li>
          </ul>
        </li>
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. rick and morty"/>
          <button onclick="searchRestaurants()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul id="restaurantsUl" class="row-container">
              {{#restaurants}}
              <li class="restaurant">
                <ul class="column-container" onclick='placeOrder("{{name}}")'>
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

    <div id="reg-dialog-form" title="Register">       
      <form>
        <fieldset>
          <label for="first-name">First Name</label>
          <input type="text" id="first-name" class="text ui-widget-content ui-corner-all">
          <label for="last-name">Last Name</label>
          <input type="text" id="last-name" class="text ui-widget-content ui-corner-all">
          <label for="email">Email</label>
          <input type="text" name="email" id="email" class="text ui-widget-content ui-corner-all">
          <label for="username">Username</label>
          <input type="text" name="username" id="username" class="text ui-widget-content ui-corner-all">
          <label for="password">Password</label>
          <input type="password" name="password" id="password" class="text ui-widget-content ui-corner-all">
        </fieldset>
      </form>
    </div>

    <div id="verify-dialog-form" title="Verify">
      <form>
        <fieldset>
            <label for="verification-code">Verification Code</label>
            <input type="text" id="verification-code" class="text ui-widget-content ui-corner-all">
        </fieldset>
      </form>
    </div>

    <div id="registered-message" title="Registration complete!">
      <p>
        <span class="ui-icon ui-icon-circle-check" style="float:left; margin:0 7px 50px 0;"></span>
        You are now registered!
      </p>
    </div>

    <div id="sign-in-form" title="Sign in">
      <form>
          <fieldset>            
            <label for="sign-in-username">Username</label>
            <input type="text" id="sign-in-username" class="text ui-widget-content ui-corner-all">
            <label for="sign-in-password">Password</label>
            <input type="password" id="sign-in-password" class="text ui-widget-content ui-corner-all">
          </fieldset>
        </form>
    </div>

  </body>

</html>
```

2. Modify `functions/get-index.js` to fetch the URL endpoint to place orders (from a new `orders_api` environment variable)

```javascript
const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']
const ordersApiRoot = process.env.orders_api
```

3. Modify `functions/get-index.js` to pass it along to the updated `index.html` template

```javascript
const view = { 
  awsRegion,
  cognitoUserPoolId,
  cognitoClientId,
  dayOfWeek, 
  restaurants,
  searchUrl: `${restaurantsApiRoot}/search`,
  placeOrderUrl: `${ordersApiRoot}`
}
```

4. Modify `serverless.yml` to add the new environment variable for the `get-index` function

```yml
environment:
  restaurants_api: 
    Fn::Join:
      - ''
      - - "https://"
        - Ref: ApiGatewayRestApi
        - ".execute-api.${opt:region}.amazonaws.com/${opt:stage}/restaurants"
  orders_api: 
    Fn::Join:
      - ''
      - - "https://"
        - Ref: ApiGatewayRestApi
        - ".execute-api.${opt:region}.amazonaws.com/${opt:stage}/orders"
  cognito_user_pool_id: ${ssm:/workshop-yancui/dev/cognito_user_pool_id}
  cognito_client_id: ${ssm:/workshop-yancui/dev/cognito_web_client_id}
```

5. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

Load the landnig page in the browser and click on one of the restaurants to order (if your login token has expired then you'll have to sign in again)

![](/images/mod14-002.png)

</p></details>

<details>
<summary><b>Add notify-restaurant function</b></summary><p>

1. Modify `serverless.yml` to add a new SNS topic for notifying restaurants, under the `resources` section

```yml
restaurantNotificationTopic:
  Type: AWS::SNS::Topic
  Properties: 
    DisplayName: ${ssm:/workshop-yancui/dev/restaurant_topic_name}
    TopicName: ${ssm:/workshop-yancui/dev/restaurant_topic_name}
```

2. Add a new parameter `/${service-name}/dev/restaurant_topic_name` in SSM parameter store with the value `restaurant-notification-dev-` followed by your name, e.g. `restaurant-notification-dev-yancui`

![](/images/mod14-003.png)

3. Add a `lib` folder to the project root

4. Add a file `kinesis.js` to the `lib` folder

5. Modify `lib/kinesis.js` to the following

```javascript
function parsePayload (record) {
  const json = new Buffer(record.kinesis.data, 'base64').toString('utf8')
  return JSON.parse(json)
}

const getRecords = (event) => event.Records.map(parsePayload)

module.exports = {
  getRecords
}
```

6. Add a file `notify-restaurant.js` in the `functions` folder

7. Modify `functions/notify-restaurant.js` to the following

```javascript
const _ = require('lodash')
const { getRecords } = require('../lib/kinesis')
const AWS = require('aws-sdk')
const kinesis = new AWS.Kinesis()
const sns = new AWS.SNS()

const streamName = process.env.order_events_stream
const topicArn = process.env.restaurant_notification_topic

module.exports.handler = async (event, context) => {
  const records = getRecords(event)
  const orderPlaced = records.filter(r => r.eventType === 'order_placed')

  for (let order of orderPlaced) {
    const snsReq = {
      Message: JSON.stringify(order),
      TopicArn: topicArn
    };
    await sns.publish(snsReq).promise()
    console.log(`notified restaurant [${order.restaurantName}] of order [${order.orderId}]`)

    const data = _.clone(order)
    data.eventType = 'restaurant_notified'

    const kinesisReq = {
      Data: JSON.stringify(data), // the SDK would base64 encode this for us
      PartitionKey: order.orderId,
      StreamName: streamName
    }
    await kinesis.putRecord(kinesisReq).promise()
    console.log(`published 'restaurant_notified' event to Kinesis`)
  }  
}
```

8. Modify `serverless.yml` to add a new `notify-restaurant` function

```yml
notify-restaurant:
  handler: functions/notify-restaurant.handler
  events:
    - stream:
        type: kinesis
        arn: 
          Fn::GetAtt:
            - orderEventsStream
            - Arn
  environment:
    order_events_stream: ${ssm:/workshop-yancui/dev/stream_name}
    restaurant_notification_topic: 
      Ref: restaurantNotificationTopic
```

9. Modify `serverless.yml` to add the permission to `sns:Publish` to the SNS topic, under `provider.iamRoleStatements`

```yml
- Effect: Allow
  Action: sns:Publish
  Resource: 
    Ref: restaurantNotificationTopic
```

10. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

11. Commit and push your changes to see that the pipeline still works

</p></details>

<details>
<summary><b>Add integration test for notify-restaurant function</b></summary><p>

1. Modify `steps/init.js` to load `restaurant_topic_name` from SSM parameter store

```javascript
const params = await getParameters([
  'restaurant_topic_name',
  'stream_name',    
  'table_name', 
  'cognito_user_pool_id', 
  'cognito_web_client_id',
  'cognito_server_client_id',
  'url'
])
```

and use it to set `restaurant_notification_topic` environment variable the `notify-restaurant` function depends on

```javascript
process.env.restaurant_notification_topic = params.restaurant_topic_name
```

2. Modify `steps/when.js` to add a `we_invoke_notify_restaurant` function

```javascript
const we_invoke_notify_restaurant = async (...events) => {
  if (mode === 'handler') {
    await viaHandler(toKinesisEvent(events), 'notify-restaurant')
  } else {
    throw new Error('not supported')
  }
}

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order,
  we_invoke_notify_restaurant
}
```

3. Modify `steps/when.js` to add a `toKinesisEvent` function

```javascript
const toKinesisEvent = events => {
  const records = events.map(event => {
    const data = Buffer.from(JSON.stringify(event)).toString('base64')
    return {
      "eventID": "shardId-000000000000:49545115243490985018280067714973144582180062593244200961",
      "eventVersion": "1.0",
      "kinesis": {
        "approximateArrivalTimestamp": 1428537600,
        "partitionKey": "partitionKey-3",
        "data": data,
        "kinesisSchemaVersion": "1.0",
        "sequenceNumber": "49545115243490985018280067714973144582180062593244200961"
      },
      "invokeIdentityArn": "arn:aws:iam::EXAMPLE",
      "eventName": "aws:kinesis:record",
      "eventSourceARN": "arn:aws:kinesis:EXAMPLE",
      "eventSource": "aws:kinesis",
      "awsRegion": "us-east-1"
    }
  })

  return {
    Records: records
  }
}
```

4. Add a file `notify-restaurant.js` to the `test_cases` folder

5. Modify `test_cases/notify-restaurant.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')
const AWS = require('mock-aws')
const chance = require('chance').Chance()

describe(`When we invoke the notify-restaurant function`, () => {
  let isEventPublished = false
  let isNotified = false

  before(async () => {
    await init()

    AWS.mock('Kinesis', 'putRecord', (req) => {
      isEventPublished = 
        req.StreamName === process.env.order_events_stream &&
        JSON.parse(req.Data).eventType === 'restaurant_notified'

      return {
        promise: async () => {}
      }
    })

    AWS.mock('SNS', 'publish', (req) => {
      isNotified = 
        req.TopicArn === process.env.restaurant_notification_topic &&
        JSON.parse(req.Message).eventType === 'order_placed'

      return {
        promise: async () => {}
      }
    })

    const event = {
      orderId: chance.guid(),
      userEmail: chance.email(),
      restaurantName: 'Fangtasia',
      eventType: 'order_placed'
    }
    await when.we_invoke_notify_restaurant(event)
  })

  after(() => {
    AWS.restore('Kinesis', 'putRecord')
    AWS.restore('SNS', 'publish')
  })

  if (process.env.TEST_MODE === 'handler') {
    it(`Should publish message to SNS`, async () => {
      expect(isNotified).to.be.true
    })

    it(`Should publish event to Kinesis`, async () => {
      expect(isEventPublished).to.be.true
    })
  }
})
```

6. Run integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that the new test is failing

```
1) When we invoke the notify-restaurant function
      "before all" hook:
    TypeError: Cannot read property 'body' of undefined
    at viaHandler (tests/steps/when.js:84:16)
    at <anonymous>
```

because our `notify-restaurant` doesn't return any response because it doesn't need to.

7. Modify `steps/when.js` to update the `viaHandler` function to handle this

```javascript
const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler
  console.log(`invoking via handler function ${functionName}`)

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (_.get(response, 'body') && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

8. Rerun integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all tests are passing now

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (367ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (532ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
notified restaurant [Fangtasia] of order [5e8f5bd3-234d-582c-b138-73d31afbb3fe]
published 'restaurant_notified' event to Kinesis
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  Given an authenticated user
[test-Lina-Catarzi-0%sG^VPl] - user is created
[test-Lina-Catarzi-0%sG^VPl] - initialised auth flow
[test-Lina-Catarzi-0%sG^VPl] - responded to auth challenge
    When we invoke the POST /orders endpoint
invoking via handler function place-order
placing order ID [6ebe25d7-9d2d-5549-89c3-24b22766440f] to [Fangtasia] for user [test-Lina-Catarzi-0%sG^VPl@test.com]
published 'order_placed' event into Kinesis
      ✓ Should return 200
      ✓ Should publish a message to Kinesis stream
[test-Lina-Catarzi-0%sG^VPl] - user deleted

  Given an authenticated user
[test-Bradley-Kuiper-rJtlGV5T] - user is created
[test-Bradley-Kuiper-rJtlGV5T] - initialised auth flow
[test-Bradley-Kuiper-rJtlGV5T] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
      ✓ Should return an array of 4 restaurants (253ms)
[test-Bradley-Kuiper-rJtlGV5T] - user deleted


  7 passing (5s)
```

</p></details>

<details>
<summary><b>Acceptance test for notify-restaurant function</b></summary><p>

We can publish a Kinesis event via the AWS SDK to execute the deployed `notify-restaurant` function. Since this function publishes to both SNS and Kinesis, we have the same conumdrum in verifying that it's producing the expected side-effects as the `place-order` function.

The same options we discussed earlier apply here, with regards to verifying the `restaurant_notified` event is published to Kinesis.

But how do we verify that SNS message has notified? And what if we had used SES as we intended initially?

To verify that an email was received, we could subscribe a test email address to the SNS topic (or whitelist it in the case of SES). Then we can programmatically (e.g. Gmail has an API which we can use to read our emails) check our inbox and see if we had received the notification email.

For this workshop, we'll take a short-cut and skip the test altogether.

1. Modify `test_cases/notify-restaurant.js` so the test case only executes as integration test

```javascript
if (process.env.TEST_MODE === 'handler') {
  describe(`When we invoke the notify-restaurant function`, () => {
    ...
  })
}
```

2. Run acceptance test

`STAGE=dev REGION=us-east-1 npm run acceptance`

and see that all the tests are passing

</p></details>