# Module 18: Log aggregation

## Ship all Lambda logs to Logz.io

**Goal:** All the logs from Lambda function are shipped to Logz.io

<details>
<summary><b>Sign up to Logz.io</b></summary><p>

1. Go to https://logz.io/

2. Sign up for a free trial

![](/images/mod18-001.png)

3. Take a note of your `token`

4. Go to EC2 console

5. Go to Parameter Store (bottom left)

6. Create a new parameter `/${service_name}/dev/logzio_token`, choose `SecureString` and use the default KMS key

![](/images/mod18-002.png)

</p></details>

<details>
<summary><b>Use Lambda function to ship logs to Logzio</b></summary><p>

1. Come out of the `workshop` folder

1. Create a new serverless project using my template

`sls create --template-url https://github.com/theburningmonk/lambda-logging-demo --path cloudwatch-logs-to-logzio`

which saves the project in a folder called `cloudwatch-logs-to-logzio`

2. Read the `readme` to see how to deploy the functions

3. Open the `serverless.yml` and specify Logz.io's host `listener.logz.io` and port `5050`

4. Set the `token` environment variable to `${ssm:/{service_name}/dev/logzio_token~true}`

5. Deploy the functions

`./build.sh deploy dev`

Congratulations! Whenever you deploy a new function, the log group would be subscribed to the log shipping function and all your logs would be shipped to Logz.io automatically.

6. Modify `process_all.js` script and change the configurations to match your account

7. Run `node process_all.js` to subscribe all existing functions' logs to the log shipping function

</p></details>
