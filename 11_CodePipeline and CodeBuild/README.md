# Module 11: CodePipeline and CodeBuild

## Set up a continuous deployment pipeline with CodePipeline and CodeBuild

**Goal:** Have a continous deployment pipeline

<details>
<summary><b>Setup CodePipeline</b></summary><p>

1. Go to the AWS Console

2. Go to the CodePipeline console

3. Click `Create Pipeline`

4. Call your pipeline `workshop-` followed by your name, e.g. `workshop-yancui`

![](/images/mod11-001.png)

5. Click `Next step`

6. Choose `GitHub` as `Source provider`

7. Click `Connect to GitHub`

8. Authorize the app

9. Select your repo, and branch

![](/images/mod11-002.png)

10. Click `Next step`

11. Choose `AWS CodeBuild` as `Build provider`

12. Choose `Create a new build project` and call the build project `workshop-dev-` followed by your name, e.g. `workshop-dev-yancui`

![](/images/mod11-003.png)

13. Choose `Use an image managed by AWS CodeBuild` and choose `Ubuntu`, `Node.js` and `aws/codebuild/nodejs:8.11.0`

14. Choose `Use the buildspec.yml in the source code root directory`

![](/images/mod11-004.png)

15. Leave the `Caching` and `VPC` configurations as they are

16. Under `Advanced`, `Environment variables`, add the environment variables `STAGE` and `REGION`. We'll use them to control which stage and region our functions would be deployed to.

![](/images/mod11-005.png)

17. Cick `Save build project`

![](/images/mod11-006.png)

18. Click `Next step`

19. Choose `No Deployment` for `Deployment provider`. We'll deploy from our build script

![](/images/mod11-007.png)

20. Click `Next step`

21. Click `Create role`

22. In the new window, accept the default permissions and click `Allow`

23. Click `Next step`

24. Click `Create pipeline`

![](/images/mod11-008.png)

Congratulations! You have created your first pipeline!

Fortunately, this first build is doomed to fail because we haven't created a `buildspec.yml` yet.

![](/images/mod11-009.png)

</p></details>

<details>
<summary><b>Add buildspec.yml</b></summary><p>

1. Add a file `buildspec.yml` under the project's root folder

2. Modify the `buildspec.yml` to the following

```yml
version: 0.2

phases:
  build:
    commands:
      - npm install
      - npm test
      - npm run sls -- deploy -s $STAGE -r $REGION
      - npm run acceptance
```

3. Modify the `steps/init.js` to add another env variable `AWS_SESSION_TOKEN`, as the build steps are executed in an ECS task (which uses temp credentials by assuming the assigned IAM role).

```javascript
process.env.AWS_ACCESS_KEY_ID     = credentials.accessKeyId
process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey

if (credentials.sessionToken) {
  process.env.AWS_SESSION_TOKEN = credentials.sessionToken
}
```

4. CodeBuild would require many permissions in order to deploy our serverless project. Go to the IAM console, and look for the role that starts with `code-build-workshop-dev-`.

5. Click `Attach policies`

6. Select `AdministratorAccess` and click `Attach policy`

7. Commit and push your code changes to kick off the pipeline.

If all goes well, you should see the pipeline execute successfully and the functions will be deployed by the pipeline.

![](/images/mod11-011.png)

![](/images/mod11-010.png)

</p></details>

<details>
<summary><b>Extend the pipeline to deploy to staging environment</b></summary><p>

1. Go back to your pipeline screen, click `Edit`

2. Add a stage after `Build`, call it `Staging`

3. Choose `Build` for `Action category`

4. Use `CodeBuild` for `Action name`

5. Choose `AWS CodeBuild` for `Build provider`

6. Choose `Create a new build project`

7. Use the name `workshop-staging-` followed by your name, e.g. `workshop-staging-yancui`

![](/images/mod11-012.png)

8. Choose `Use an image managed by AWS CodeBuild`, and use `Ubuntu`, `Node.js` and `aws/codebuild/nodejs:8.11.0`

![](/images/mod11-013.png)

9. Choose `Use the buildspec.yml in the source code root directory`

10. Leave `Cache` and `VPC` options as the default

11. Under `Advanced`, `Environment variables`, add the `STAGE` and `REGION` environment variables to make the npm build script deploy to a `staging` stage in `us-east-1` region instead.

![](/images/mod11-014.png)

12. Click `Save build project`

13. Choose `MyApp` as `Input artifacts`

14. Click `Add action`

15. Click `Save pipeline changes`, and then `Save and continue` in the subsequent popup

16. Go to the IAM console

17. CodeBuild would require many permissions in order to deploy our serverless project. Go to the IAM console, and look for the role that starts with `code-build-workshop-staging-`.

18. Click `Attach policies`

19. Select `AdministratorAccess` and click `Attach policy`

20. Go back to the CodePipeline console, and go into our `workshop-yancui` pipeline

21. Click `Release change` to kick off the pipeline

22. This will fail because the DynamoDB table is not parameterised by stage

![](/images/mod11-015.png)

Even if we parameterise everything in the `serverless.yml`, we'd have to parameterise the tests as well.

</p></details>

## Exercise

<details>
<summary><b>Separate steps for test, deploy and acceptance</b></summary><p>

Instead of relying on `buildspec.yml` to do everything - integration tests, deploy and acceptance tests - in one giant step, how about we separate them into multiple steps for each environment?

Delete the existing `Build` step, and create a `Dev` that has 3 clear steps:

* IntegrationTest

* Deploy

* AcceptanceTest

![](/images/mod11-016.png)

</p></details>