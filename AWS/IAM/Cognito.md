---
tags:
  - aws
  - certification
  - iam
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[IAM]] · [[ARN]]

# The Complete Guide to Amazon Cognito

Amazon Cognito is a robust Customer Identity and Access Management (CIAM) service provided by AWS. It allows you to add user sign-up, sign-in, and access control to your web and mobile apps quickly and easily. Cognito scales to millions of users and supports sign-in with social identity providers, such as Apple, Facebook, Google, and Amazon, as well as enterprise identity providers via SAML 2.0 and OpenID Connect (OIDC).

---

## 1. The Core Components: User Pools vs. Identity Pools

The most common point of confusion with Cognito is understanding the difference between its two main components. They can be used separately or together.

### **Cognito User Pools (Authentication)**
A User Pool is a user directory. It is responsible for **Identity and Authentication (Who are you?)**.
* **Purpose:** Handles sign-up, sign-in, account recovery, and MFA.
* **Tokens:** Returns standard JSON Web Tokens (JWTs) — **ID Token, Access Token, and Refresh Token** upon successful authentication.
* **Federation:** Can act as an Identity Provider (IdP) itself, or federate through Google, Facebook, Apple, or SAML/OIDC enterprise directories.

### **Cognito Identity Pools (Authorization)**
An Identity Pool (formerly Federated Identities) is responsible for **Access and Authorization (What AWS resources can you access?)**.
* **Purpose:** Exchanges identity tokens (from User Pools, Google, etc.) for temporary, limited-privilege AWS IAM credentials.
* **Tokens:** Does *not* return JWTs. It returns **AWS Credentials** (`AccessKeyId`, `SecretAccessKey`, `SessionToken`).
* **Guest Access:** Supports unauthenticated (guest) identities, allowing anonymous users to access specific AWS resources.

---

## 2. Deep Dive: Cognito User Pools

### Key Features
1.  **Hosted UI:** Cognito provides a customizable, out-of-the-box web UI for user sign-up and sign-in. This saves you from writing complex auth flows from scratch.
2.  **Advanced Security Features (ASF):** Offers risk-based authentication to block suspicious logins and checks for compromised credentials.
3.  **Multi-Factor Authentication (MFA):** Supports SMS text messages and Time-based One-Time Passwords (TOTP) via authenticator apps.
4.  **Lambda Triggers:** You can customize the authentication workflow by hooking AWS Lambda functions at different stages:
    * *Pre sign-up:* Auto-confirm users or deny sign-ups based on custom validation (e.g., specific email domains).
    * *Pre authentication:* Block sign-ins based on custom logic.
    * *Post authentication:* Log sign-in events or fetch additional user data.
    * *Custom message:* Dynamically customize email/SMS verification messages.
    * *Token generation:* Add custom claims to the ID or Access JWTs.

### The Token Trio (JWT)
When a user authenticates against a User Pool, they receive three tokens:
1.  **ID Token:** Contains claims about the identity of the authenticated user (e.g., `name`, `email`, `custom_attributes`). Used by your application UI.
2.  **Access Token:** Contains scopes and groups. Used to authorize API calls (e.g., passing it to API Gateway in the `Authorization` header).
3.  **Refresh Token:** Used to obtain new ID and Access tokens when they expire (default expiration is 1 hour). Refresh tokens can be valid for up to 10 years.

---

## 3. Deep Dive: Cognito Identity Pools

Identity pools map users to **IAM Roles**. 

### Role Mapping
You configure two primary IAM roles for an Identity Pool:
1.  **Authenticated Role:** The IAM role assumed by users who have successfully logged in.
2.  **Unauthenticated Role:** The IAM role assumed by guest users.

*Example Scenario:* You have a mobile app where users upload profile pictures directly to an S3 bucket. 
* An *unauthenticated* user gets a role that only allows `s3:GetObject` to read public assets.
* An *authenticated* user gets a role with `s3:PutObject`. By using IAM Policy variables (e.g., `${cognito-identity.amazonaws.com:sub}`), you can restrict users to only upload files to their specific folder in the S3 bucket.

---

## 4. Common Architectural Patterns

### Pattern A: Modern Web App with APIs (User Pool Only)
*Best for: SPAs (React, Vue, Angular) calling custom backend APIs.*
1. User logs in via Cognito Hosted UI or custom frontend.
2. Cognito User Pool authenticates and returns JWTs (ID and Access Tokens) to the frontend.
3. Frontend makes HTTP requests to an **Amazon API Gateway** endpoint, passing the Access Token in the header.
4. API Gateway validates the token using a **Cognito Authorizer** before passing the request to the backend backend (e.g., AWS Lambda).

### Pattern B: Direct AWS Resource Access (User Pool + Identity Pool)
*Best for: Mobile/Web apps needing direct access to AWS services (S3, DynamoDB, AppSync) without an API proxy.*
1. User logs in via Cognito User Pool and receives an ID Token.
2. Frontend passes the ID Token to the Cognito Identity Pool.
3. Identity Pool validates the token and returns temporary AWS IAM credentials.
4. Frontend uses these IAM credentials with the AWS SDK to directly call AWS services (e.g., uploading a file to Amazon S3).

---

## 5. Security and Best Practices

* **Never hardcode credentials:** Always use Identity Pools to vend temporary credentials if your app needs direct AWS access.
* **Scope your tokens:** Use OAuth 2.0 scopes (via Resource Servers in Cognito) to limit what an Access Token can do (e.g., `read:profile` vs `write:admin`).
* **Enforce MFA:** Enable MFA for all users, preferring TOTP over SMS for better security and lower costs.
* **Use AWS WAF:** Attach AWS Web Application Firewall (WAF) to your Cognito User Pool to protect against rate-based attacks and malicious IP addresses.
* **Backup & Migration limitations:** Cognito does *not* allow you to export user passwords. If you ever need to migrate users into or out of Cognito, you must use a "User Migration Lambda Trigger" which migrates users one-by-one as they log in, or force a password reset.

## 6. Getting Started Checklist

1.  Navigate to the AWS Console -> Amazon Cognito.
2.  **Create a User Pool:** Choose sign-in options (Email, Username).
3.  **Configure Security:** Set password policies and MFA requirements.
4.  **App Client:** Create an App Client (do NOT generate a client secret if using a frontend SPA or mobile app).
5.  **Configure Domain:** Set up a Cognito domain name if you plan to use the Hosted UI.
6.  **Integration:** Use AWS Amplify Auth (`aws-amplify`) or the AWS SDK to integrate Cognito into your frontend codebase.

---
*Generated by Google Gemini.*
