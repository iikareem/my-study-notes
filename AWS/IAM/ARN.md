---
tags:
  - aws
  - certification
  - iam
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[IAM]] · [[Cognito]]

# ARN

In AWS (Amazon Web Services), an **ARN** (Amazon Resource Name) is a unique identifier used to specify a resource within AWS. ARNs are standardized strings that uniquely identify AWS resources, such as IAM users, groups, roles, S3 buckets, EC2 instances, or any other resource across AWS services.

### **What is an ARN?**
- **Definition**: An ARN is a string that follows a specific format to identify a resource in AWS, allowing precise referencing for permissions, policies, or API calls.
- **Purpose**:
  - Uniquely identifies resources across AWS accounts, regions, and services.
  - Used in IAM policies to specify which resources permissions apply to.
  - Enables cross-service and cross-account interactions (e.g., granting access to an S3 bucket in another account).

### **ARN Format**
The general format of an ARN is:

```
arn:partition:service:region:account-id:resource-type/resource-id
```

**Components**:
1. **arn**: Fixed string indicating an Amazon Resource Name.
2. **partition**: The AWS partition (e.g., `aws` for standard AWS, `aws-cn` for China, `aws-us-gov` for GovCloud).
3. **service**: The AWS service namespace (e.g., `s3`, `iam`, `ec2`, `lambda`).
4. **region**: The AWS region (e.g., `us-east-1`, `eu-west-2`). Some services (e.g., IAM, S3) are global and omit the region.
5. **account-id**: The 12-digit AWS account ID (e.g., `123456789012`). Can be omitted for some global resources.
6. **resource-type/resource-id**: The type and identifier of the resource (e.g., `user/username`, `bucket/bucket-name`, `role/role-name`).

### **Examples of ARNs**
1. **IAM User**:
   ```
   arn:aws:iam::123456789012:user/johndoe
   ```
   - Identifies a user named `johndoe` in account `123456789012`.

2. **S3 Bucket**:
   ```
   arn:aws:s3:::my-example-bucket
   ```
   - Identifies an S3 bucket named `my-example-bucket`. Note: S3 is global, so region and account ID are omitted.

3. **IAM Role**:
   ```
   arn:aws:iam::123456789012:role/MyEC2Role
   ```
   - Identifies an IAM role named `MyEC2Role`.

4. **EC2 Instance**:
   ```
   arn:aws:ec2:us-east-1:123456789012:instance/i-0abcd1234efgh5678
   ```
   - Identifies an EC2 instance in the `us-east-1` region.

5. **Lambda Function**:
   ```
   arn:aws:lambda:us-west-2:123456789012:function:my-function
   ```
   - Identifies a Lambda function named `my-function` in `us-west-2`.

### **ARN in IAM**
- **Role in IAM**: ARNs are critical in IAM policies to specify which resources a user, group, or role can access.
- **Use in Policies**:
  - In the `Resource` field of an IAM policy, ARNs define the specific resources the policy applies to.
  - Example:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3:::my-example-bucket/*"
        }
      ]
    }
    ```
    This policy allows `GetObject` actions on all objects in the `my-example-bucket` S3 bucket.
- **Trust Policies**: ARNs are used in IAM role trust policies to specify who (e.g., an AWS service or another account) can assume the role.
  - Example:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": { "AWS": "arn:aws:iam::123456789012:root" },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```
    This trust policy allows the account `123456789012` to assume the role.

### **Key Points**
- **Uniqueness**: ARNs ensure resources are uniquely identified, avoiding conflicts across accounts or regions.
- **Wildcards**: ARNs in policies can use wildcards (e.g., `*` or `?`) to apply to multiple resources (e.g., `arn:aws:s3:::my-bucket/*` for all objects in a bucket).
- **Global vs. Regional**: Some services (e.g., IAM, S3) have global ARNs (no region), while others (e.g., EC2, Lambda) include a region.
- **Cross-Account Access**: ARNs enable granting access to resources in different AWS accounts by specifying the full ARN, including the account ID.

### **Best Practices**
- Use specific ARNs in policies to follow the principle of least privilege (avoid overly broad permissions like `*`).
- Verify ARNs in policies to ensure they point to the correct resource.
- Use the AWS Policy Simulator to test how ARNs affect permissions.

If you need help crafting an ARN for a specific resource, writing an IAM policy with ARNs, or understanding ARN usage in a particular AWS service, let me know!
