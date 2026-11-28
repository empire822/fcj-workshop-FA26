---
title : "Authentication & Authorization"
date : 2024-01-01
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

In this section, we will configure an **Amazon Cognito User Pool** to authenticate staff members and assign permissions based on their roles in **SecureTable**.

**Note:** Customers (Guests) **do not use Cognito**. Guests receive a Session Token through an API when their table has an open bill.

## 1. Create a Cognito User Pool

1. Open **Amazon Cognito** in the AWS Console and select **Asia Pacific (Singapore) – `ap-southeast-1`** as the Region.
2. Select **User pools → Create user pool**.
3. Select **Single-page application (SPA)** as the application type for the HTML/JavaScript frontend.
4. Name the User Pool, for example `SecureTable-Staff`, and use **Username** and password for sign-in.
5. Disable **Self-registration** so that only administrators can create staff accounts, then complete **Create user pool**.

![Cognito User Pool setup](../../images/5-Workshop/5.5-Auth/cog-cf.png)

**Note:** There is no need to create a User Pool for Guests.

## 2. Configure the App Client & Login Page

1. In the User Pool, open **App clients** and create or select the App Client for the Staff website.
2. Choose **Single-page application (SPA)** and **do not use a Client Secret**.
3. Enable **Authorization code grant** with **PKCE**; select at least the `openid` OAuth scope (add `email` and `profile` if required by the interface).
4. Configure the following URLs, replacing `<cloudfront-domain>` with the domain from Section 5.3:
   - **Callback URL:** `https://<cloudfront-domain>/callback.html`
   - **Sign-out URL:** `https://<cloudfront-domain>/login.html`
5. Under **Domain**, create a **Cognito domain** and configure the User Pool's login page (**Managed login** or **Hosted UI**).
6. Save the configuration and record the **User Pool ID**, **App Client ID**, and **Cognito domain**.

![Cognito App Client and callback URLs](../../images/5-Workshop/5.5-Auth/cog-url.png)
![Cognito domain](../../images/5-Workshop/5.5-Auth/cog-domain.png)

**Note:** The Callback URL must exactly match the website URL, including `https://` and `/callback.html`. Do not include a Client Secret in frontend JavaScript.

## 3. Create Permission Groups & Staff Accounts

1. In the User Pool, select **Groups → Create group**.
2. Create the following groups:

| Cognito Group | Function |
|---|---|
| `RECEPTION` | Manage tables and open or close bills |
| `KITCHEN` | Receive and process orders |
| `INVENTORY` | Record stock receipts and adjust ingredients |
| `ADMIN` | View financial reports and Audit Logs |

![Cognito Groups](../../images/5-Workshop/5.5-Auth/cog-group.png)

3. Select **Users → Create user** to create staff accounts.

![Cognito Users](../../images/5-Workshop/5.5-Auth/cog-user.png)

4. Add each staff member to the appropriate group under **Group membership**.

![Add User to Group example](../../images/5-Workshop/5.5-Auth/cog-groupuser.png)

**Note:** Cognito includes group information in the `cognito:groups` claim in the ID Token. Redirecting users based on their roles **does not replace backend authorization checks**. Staff APIs will be protected by a **Cognito Authorizer**, with additional permission checks in Lambda in Section 5.6.

## 4. Connect Cognito to the Frontend

1. Open the SecureTable frontend's `config.js` file.
2. Update the Cognito configuration:

```javascript
window.SECURETABLE_CONFIG = Object.freeze({
  cognitoDomain: 'https://<your-cognito-domain>',
  clientId: '<your-app-client-id>',
  userPoolId: '<your-user-pool-id>'
});
```

3. Upload `config.js` to the **frontend S3 bucket**, overwriting the existing file.
4. Create a **CloudFront Invalidation** for `/config.js` if the website still loads the old configuration.
5. Open `https://<cloudfront-domain>/login.html`, sign in with a Staff account, and verify that the system redirects to `callback.html` and then to the correct interface for the user's group.

![Staff login](../../images/5-Workshop/5.5-Auth/login.png)
![Cognito login](../../images/5-Workshop/5.5-Auth/cog-login.png)

**Note:** At this stage, we are verifying **login and interface redirection**. Business APIs will become fully functional after the backend is deployed in Section 5.6.

After completing these steps, **Cognito is configured for Staff**, and the permission groups and staff accounts are ready.
