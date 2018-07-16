# Module 4: Cognito 101

## Create a new Cognito User Pool

**Goal:** Set up a new Cognito User Pool

<details>
<summary><b>Add a new Cognito User Pool</b></summary><p>

1. Go to the Cognito console

2. Click `Create a user pool`

3. Use the name `workshop-` followed by your name, e.g. `workshop-yancui`

![](/images/mod04-001.png)

4. Click `Step through settings`

5. Tick `Also allow sign in with verified email address`, `family name` and `given name`

![](/images/mod04-002.png)

6. Click `Next step`

7. Accept all the default settings in the next screen

![](/images/mod04-003.png)

8. Click `Next step`

9. Accept all the default settings in the next screen

![](/images/mod04-004.png)

10. Click `Next step`

11. Accept all the default settings in the next screen

![](/images/mod04-005.png)

12. Click `Next step`

13. Ignore tags, and click `Next step`

14. Choose `No` to remember user devices

![](/images/mod04-006.png)

15. Click `Next step`

</p></details>

<details>
<summary><b>Add client for web</b></summary><p>

![](/images/mod04-007.png)

1. Click `Add an app client`

2. Name the app client `web`

3. Untick `Generate client secret`, because for now the Javascript SDK does not support client secrets.

![](/images/mod04-008.png)

4. Remove the permission for Writable Attributes for `Address` and `Profile`

![](/images/mod04-009.png)

5. Click `Create app client`

</p></details>

<details>
<summary><b>Add client for server</b></summary><p>

1. Click `Add an app client`

2. Name the app client `server`

3. Untick `Generate client secret`, because for now the Javascript SDK does not support client secrets.

4. Tick `Enable sign-in API for server-based authentication (ADMIN_NO_SRP_AUTH)`

![](/images/mod04-010.png)

5. Leave all the permissions as is

6. Click `Create app client`

</p></details>

<details>
<summary><b>Complete the creation process</b></summary><p>

1. Click `Next step`

2. Leave all the workflow customizations

3. Click `Next step`

4. Review the settings

![](/images/mod04-011.png)

5. Click `Create pool`

6. Note the `Pool Id`, and the app client IDs for both `web` and `server`

![](/images/mod04-012.png)

![](/images/mod04-013.png)

</p></details>