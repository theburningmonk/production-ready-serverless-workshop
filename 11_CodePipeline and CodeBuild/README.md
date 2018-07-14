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
<summary><b>Setup buildspec.yml</b></summary><p>

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
```



</p></details>