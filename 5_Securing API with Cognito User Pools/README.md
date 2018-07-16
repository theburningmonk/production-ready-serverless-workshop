# Module 5: Securing API with Cognito User Pools

## Create POST /restaurants/search endpoint

**Goal:** Set up a `/restaurants/search` endpoint for searching restaurants by theme.

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

`npm run sls -- deploy -s dev -r us-east-1`

5. Curl the `/restaurants/search` endpoint for the `cartoon` theme

`curl -d '{"theme":"cartoon"}' -H "Content-Type: application/json" -X POST https://xxx-api.us-east-1.amazonaws.com/dev/restaurants/search`

and see that the response is

```json
[
  {
    "name": "Shoney's",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/shoney's.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  },
  {
    "name": "Lil' Bits",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/lil+bits.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  },
  {
    "name": "Fancy Eats",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/fancy+eats.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  },
  {
    "name": "Don Cuco",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/don%20cuco.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  }
]
```

</p></details>

## Securing search-restaurants endpoint with Cognito User Pools

**Goal:** The `/restaurants/search` endpoint is protected by Cognito User Pools

<details>
<summary><b>Protect the search-restaurants function with Cognito User Pools</b></summary><p>

1. Modify the `serverless.yml` and set the `authorizer` for the `search-restaurants` function to the Cognito User Pools, using the pool ID from the last module

```yml
search-restaurants:
  handler: functions/search-restaurants.handler
  events:
    - http:
        path: /restaurants/search
        method: post
        authorizer:
          arn: arn:aws:cognito-idp:#{AWS::Region}:#{AWS::AccountId}:userpool/xxx
  environment:
    restaurants_table: restaurants
```

2. Deploy the project

`npm run sls -- deploy -r us-east-1 -s dev`

3. Curl the `/restaurants/search` endpoint for the `cartoon` theme

`curl -d '{"theme":"cartoon"}' -H "Content-Type: application/json" -X POST https://xxx-api.us-east-1.amazonaws.com/dev/restaurants/search`

and see the response

```json
{ 
  "message": "Unauthorized"
}
```

</p></details>

<details>
<summary><b>Pass the Cognito User Pool id to the index.html</b></summary><p>

1. Modify `serverless.yml` and update the `get-index` function to add `cognito_user_pool_id` and `cognito_client_id` environment variables with the `pool Id` and the `web` app client Id from the last module

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
    cognito_user_pool_id: xxx
    cognito_client_id: xxx
```

2. Modify the `get-index` function to the following

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']

const awsRegion = process.env.AWS_REGION
const cognitoUserPoolId = process.env.cognito_user_pool_id
const cognitoClientId = process.env.cognito_client_id

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
  const view = { 
    awsRegion,
    cognitoUserPoolId,
    cognitoClientId,
    dayOfWeek, 
    restaurants,
    searchUrl: `${restaurantsApiRoot}/search`
  }
  const html = Mustache.render(template, view)
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

3. Modify the `index.html` to the following

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
                <ul class="column-container">
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
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="searchRestaurants()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul id="restaurantsUl" class="row-container">
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

4. Deploy the project

`npm run sls -- deploy -r us-east-1 -s dev`

5. Go to the `index.html` in the browser

![](/images/mod05-001.png)

6. Click `Register`

![](/images/mod05-002.png)

7. Click `Create an account`

8. Check your email, and note the verification code

![](/images/mod05-003.png)

9. Go back to the page and fill in your verification code

10. Click `Confirm registration`, and now you're registered!

![](/images/mod05-004.png)

11. Click `Sign in`

![](/images/mod05-005.png)

12. Enter `cartoon` in the search box and click `Find Restaurants`, and see that the results are returned

![](/images/mod05-006.png)

</p></details>