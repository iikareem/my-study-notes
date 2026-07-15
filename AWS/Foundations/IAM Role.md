---
tags:
  - aws
  - certification
  - iam
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[IAM Basics]] · [[IAM]]

# AWS IAM Roles, Trust Policy, Permissions Policy, and Use Cases

## IAM Role

- **Definition**: An IAM role is an AWS identity with permissions that can be assumed temporarily by trusted entities (e.g., AWS services, users, or external accounts) to access AWS resources.
- **Key Features**:
    - Uses temporary credentials via AWS Security Token Service (STS), enhancing security.
    - Defined by a **trust policy** (who can assume) and **permissions policy** (what they can do).
    - Not tied to a specific user or entity; can be assumed by multiple entities.
- **Why Use?**: Ideal for secure, temporary access without permanent credentials, unlike IAM users with static access keys.

## Trust Policy

- **Definition**: A JSON document that specifies **who** (principals) can assume the IAM role.
- **Purpose**: Controls **authentication** by defining trusted entities (e.g., AWS services, accounts, or federated users) allowed to assume the role via STS (e.g., `sts:AssumeRole`).
- **Key Components**:
    - **Principal**: Entity allowed to assume the role (e.g., `ec2.amazonaws.com`, specific account ARN).
    - **Action**: Typically `sts:AssumeRole`, `sts:AssumeRoleWithSAML`, etc.
    - **Effect**: Usually `Allow`.
    - **Condition** (optional): Restricts access (e.g., MFA, IP address).
- **Example** (EC2 can assume role):
    
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": { "Service": "ec2.amazonaws.com" },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```
    
- **Why Limit Principals?**: Prevents unauthorized entities from assuming the role, ensuring security (e.g., only specific services or accounts).

## Permissions Policy

- **Definition**: A JSON document that defines **what** actions the role can perform on which AWS resources.
- **Purpose**: Controls **authorization** by specifying allowed or denied actions (e.g., `s3:GetObject`) and resources (e.g., S3 bucket ARN).
- **Types**:
    - **Managed Policies**: Reusable (AWS-managed like `AmazonS3ReadOnlyAccess` or customer-managed).
    - **Inline Policies**: Embedded in the role, non-reusable.
- **Key Components**:
    - **Effect**: `Allow` or `Deny`.
    - **Action**: AWS service actions (e.g., `s3:PutObject`).
    - **Resource**: Specific resources via ARNs (e.g., `arn:aws:s3:::my-bucket/*`).
    - **Condition** (optional): Contextual restrictions (e.g., time, IP).
- **Example** (S3 read access):
    
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3:::my-bucket/*"
        }
      ]
    }
    ```
    
- **Managed vs. Inline**: Prefer managed policies for reusability and easier management; use inline for unique, one-off permissions.

## How Trust and Permissions Policies Work Together

- **Trust Policy**: Determines **who** can assume the role (gatekeeper).
- **Permissions Policy**: Defines **what** the role can do once assumed.
- **Workflow**:
    1. Entity (e.g., EC2, user) attempts to assume the role.
    2. AWS checks the trust policy to verify the entity is allowed.
    3. If allowed, STS issues temporary credentials.
    4. The entity uses credentials to perform actions allowed by the permissions policy.

## Use Cases of IAM Roles

1. **AWS Services Accessing Resources**:
    - **Example**: EC2 instance reading from S3 bucket.
    - **Role**: Trust policy allows `ec2.amazonaws.com`; permissions policy grants `s3:GetObject`.
    - **Benefit**: No need to hardcode access keys; temporary credentials enhance security.
2. **Cross-Account Access**:
    - **Example**: Account A allows Account B to manage its S3 bucket.
    - **Role**: Trust policy specifies Account B’s ARN; permissions policy grants S3 access.
    - **Benefit**: Secure, temporary access without sharing credentials.
3. **Federated Access (Single Sign-On)**:
    - **Example**: Employees access AWS via corporate credentials (SAML/Okta).
    - **Role**: Trust policy allows SAML provider; permissions policy defines AWS access.
    - **Benefit**: No need to create IAM users for each employee; simplifies management.
4. **Temporary Access for Applications/Users**:
    - **Example**: CI/CD pipeline (e.g., Jenkins) deploys to AWS.
    - **Role**: Trust policy allows pipeline’s account; permissions policy grants deployment actions.
    - **Benefit**: Temporary credentials reduce risk compared to static user keys.
5. **Delegated Access for IAM Users**:
    - **Example**: Developer needs temporary admin access for troubleshooting.
    - **Role**: Trust policy allows specific user ARN; permissions policy grants admin rights.
    - **Benefit**: Temporary privilege escalation without permanent user changes.

## Why IAM Roles Are Often Better Than IAM Users

- **Security**: Temporary credentials (STS) vs. permanent access keys; reduces leakage risk.
- **Service Integration**: Designed for AWS services (e.g., EC2, Lambda) to assume dynamically.
- **Federation**: Supports SSO with external identity providers, avoiding user creation.
- **Cross-Account**: Simplifies secure access across accounts without sharing keys.
- **Scalability**: One role can be assumed by multiple entities, reducing management overhead.
- **When to Use Users**: For individual, long-term access (e.g., developer console login).

## Best Practices

- **Trust Policy**: Specify only necessary principals; use conditions (e.g., MFA) for security.
- **Permissions Policy**: Use managed policies for reusability; follow least privilege.
- **General**: Audit roles with IAM Access Analyzer; avoid broad principals (`*`) in trust policies.





![[Pasted image 20250811212414.png]]

STS in Amazon refers to **AWS Security Token Service**.

### What is AWS STS?

AWS STS is a web service that **enables you to request temporary, limited-privilege credentials** for AWS Identity and Access Management (IAM) users or for users that you authenticate (federated users).

### Key points about AWS STS:

- **Temporary credentials**: Instead of using long-term access keys, STS provides short-lived credentials (access key ID, secret access key, and session token) that expire after a set duration (from a few minutes to hours).
    
- **Use cases**:
    
    - Granting temporary access to AWS resources for users or applications without creating IAM users.
        
    - Allowing federated users (e.g., users authenticated via a corporate directory, or a web identity like Google or Facebook) to access AWS resources.
        
    - Delegating permissions across accounts.
        
- **Common STS APIs**:
    
    - `AssumeRole`: Get temporary credentials to assume an IAM role.
        
    - `GetSessionToken`: Get temporary credentials for the IAM user.
        
    - `AssumeRoleWithWebIdentity`: Get temporary credentials for users authenticated through a web identity provider.
        
    - `AssumeRoleWithSAML`: Get temporary credentials using SAML authentication.
        

### Why use STS?

- Enhances security by **avoiding permanent credentials**.
    
- Enables **fine-grained access control** with temporary tokens.
    
- Facilitates **cross-account access** and **federated access**.
    

---

If you want, I can explain any particular STS API or give you an example of how to use STS in AWS!
