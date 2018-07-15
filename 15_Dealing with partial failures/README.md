# Module 15: Dealing with partial failures

## Process failed events out of band with SNS

**Goal:** Implement mechanism to retry failed events out of band with SNS

<details>
<summary><b>Encapsulate notify-restaurant logic into shared lib module</b></summary><p>

1. Add a file `notify.js` to the `lib` folder

2. Modify `lib/notify.js` to the following

```javascript
const _ = require('lodash')
const AWS = require('aws-sdk')
const sns = new AWS.SNS()
const kinesis = new AWS.Kinesis()
const chance  = require('chance').Chance()

const streamName = process.env.order_events_stream
const topicArn = process.env.restaurant_notification_topic
const failureRate = process.env.failure_rate || 75 // 75% chance of failure

const restaurantOfOrder = async (order) => {
  if (chance.bool({likelihood: failureRate})) {
    throw new Error("boom")
  }

  const snsReq = {
    Message: JSON.stringify(order),
    TopicArn: topicArn
  }
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

module.exports = {
  restaurantOfOrder
}
```

This is basically lifted from the `notify-restaurant` function, and we have introduced a default 75% error rate so we can force messages to go down the retry path.

3. Add a file `retry.js` to the `lib` folder

4. Modify `lib/retry.js` to the following

```javascript
const AWS = require('aws-sdk')
const sns = new AWS.SNS()

const retryTopicArn = process.env.restaurant_notification_retry_topic

const restaurantNotification = async (order) => {
  let snsReq = {
    Message: JSON.stringify(order),
    TopicArn: retryTopicArn
  };
  await sns.publish(snsReq).promise()
  console.log(`order [${order.orderId}]: queued restaurant notification for retry`)
}

module.exports = {
  restaurantNotification
}
```

This module depends on a new environment variable `restaurant_notification_retry_topic` whose value needs to be parameterised via SSM parameter store. 

5. Go to EC2 console

6. Go to Parameter Store (bottom left)

7. Create a new parameter for `/{service-name}/dev/restaurant_retry_topic_name` with the value `restaurant-notification-retry-dev-` followed by your name, e.g. `restaurant-notification-retry-dev-yancui`

![](/images/mod15-005.png)

8. Create another parameter for `/{service-name}/dev/restaurant_dlq_topic_name` with the value `restaurant-notification-dlq-dev-` followed by your name, e.g. `restaurant-notification-dlq-dev-yancui`

9. Modify `functions/notify-restaurant.js` to the following

```javascript
const { getRecords } = require('../lib/kinesis')
const notify = require('../lib/notify')
const retry = require('../lib/retry')

module.exports.handler = async (event, context) => {
  const records = getRecords(event)
  const orderPlaced = records.filter(r => r.eventType === 'order_placed')

  for (let order of orderPlaced) {
    try {
      await notify.restaurantOfOrder(order)
    } catch (err) {
      console.log(`failed to notify restaurant of order [${order.orderId}], queuing for retry...`)
      await retry.restaurantNotification(order)
    }
  }
}
```

Notice how we have moved all the logif for notifying the restaurant into a shared module in the `lib` folder, which can be used from another Lambda function during retry.

10. Modify `steps/init.js` and set failure rate to 0 so our acceptance tests don't fail

```javascript
process.env.failure_rate = 0
```

</p></details>

<details>
<summary><b>Add retry-notify-restaurant function</b></summary><p>

1. Add a file `retry-notify-restaurant.js` to the `functions` folder

2. Modify `functions/retry-notify-restaurant.js` to the following

```javascript
const notify = require('../lib/notify')

module.exports.handler = async (event, context) => {
  const order = JSON.parse(event.Records[0].Sns.Message)
  order.retried = true

  await notify.restaurantOfOrder(order)
}
```

3. Modify `serverless.yml` to add two new SNS topics for retry and DLQ, under the `resources` section

```yml
restaurantNotificationRetryTopic:
  Type: AWS::SNS::Topic
  Properties: 
    DisplayName: ${ssm:/workshop-yancui/dev/restaurant_retry_topic_name}
    TopicName: ${ssm:/workshop-yancui/dev/restaurant_retry_topic_name}

restaurantNotificationDLQTopic:
  Type: AWS::SNS::Topic
  Properties: 
    DisplayName: ${ssm:/workshop-yancui/dev/restaurant_dlq_topic_name}
    TopicName: ${ssm:/workshop-yancui/dev/restaurant_dlq_topic_name}
```

Notice that we are referencing the SSM parameters we created previously.

4. Modify `serverless.yml` to add the new environment variable `restaurant_notification_retry_topic`, used by the `retry` module, and reference the new `restaurantNotificationRetryTopic` SNS topic we added in the previous step

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
    restaurant_notification_retry_topic: 
      Ref: restaurantNotificationRetryTopic
```

5. Modify `serverless.yml` to add the new `retry-notify-restaurant` function

```yml
retry-notify-restaurant:
  handler: functions/retry-notify-restaurant.handler
  events:
    - sns: 
        arn: 
          Ref: restaurantNotificationRetryTopic
        topicName: ${ssm:/workshop-yancui/dev/restaurant_retry_topic_name}
  environment:
    order_events_stream: ${ssm:/workshop-yancui/dev/stream_name}
    restaurant_notification_topic: 
      Ref: restaurantNotificationTopic
  onError: 
    Ref: restaurantNotificationDLQTopic
```

The serverless framework would normally create the SNS topic for you, but in this case we need to reference the retry topic from several places. Hence why I decided to create the topic in `resources` and then reference it here.

To subscribe the function to an existing SNS topic, you need to specify both the `arn` and the `topicName` where the `topicName` must match what's in the `arn`. Hence the setup above.

6. Modify `serverless.yml` to add `sns:Publish` permission for the two new SNS topics, to the `provider.iamRoleStatements` section

```yml
- Effect: Allow
  Action: sns:Publish
  Resource: 
    - Ref: restaurantNotificationTopic
    - Ref: restaurantNotificationRetryTopic
    - Ref: restaurantNotificationDLQTopic
```

7. Run integration tests to make sure they still pass

`STAGE=dev REGION=us-east-1 npm run test`

8. Deploy the project

`npm run sls -- deploy -s dev -r us-east-1`

</p></details>

<details>
<summary><b>Subscribe yourself to the SNS topics</b></summary><p>

1. Go to SNS console

2. Go to your restaurant notification topic

3. Click `Create subscription`

4. Choose `Email` for `Protocol` and enter your email

5. Click `Create subscription`

![](/images/mod15-002.png)

6. Check your email, and look for an email from `AWS Notification - Subscription Confirmation`

![](/images/mod15-003.png)

7. Click the `Confirm subscription` link

![](/images/mod15-004.png)

8. Repeat step 3-7 for your restaurant notification retry topic

9. Repeat step 3-7 for your restaurant notification DLQ topic

![](/images/mod15-001.png)

Ok, now you have subscribed yourself to all the SNS topics, so we can see how the messages look when they have been processed, retried, or dropped into the dead letter queue.

</p></details>

Great! Reload the landing page in the browser and try to place a few orders.

You should receive a number of emails from SNS of three variants:

* successfully processed by Kinesis function (no `retry`)

![](/images/mod15-006.png)

* successfully retried by SNS function (`retried` = `true`)

![](/images/mod15-007.png)

* dead-letter queued

![](/images/mod15-008.png)

## Exercises

<details>
<summary><b>Subscribe yourself to the SNS topics</b></summary><p>

</p></details>