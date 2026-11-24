# AWS Identity and Access Management (IAM)

## 1. IAM fundamentals

**AWS Identity and Access Management (IAM)** is a global AWS service that controls access to AWS resources.

- **Authentication** verifies the identity making a request. A password and multi-factor authentication (MFA) can verify a person signing in.
- **Authorization** determines whether that identity can perform the requested action on the target resource. AWS evaluates the applicable policies for the request.

```text
User
  |
  v
Authentication
(Password + MFA)
  |
  v
Authorization
(IAM Policies)
  |
  v
Access to Permitted AWS Resources
```

### AWS account root user

Every AWS account has a **root user** with full access by default. Reserve it for tasks that require root credentials. Protect root sign-in with MFA where supported, keep its credentials private, and avoid creating root access keys. Give each person an individual identity. AWS recommends federation through IAM Identity Center for workforce access. [AWS IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

### IAM users and groups

An **IAM user** is an identity within an AWS account. You can attach permissions directly to a user or grant them through group membership. A user can belong to zero, one, or several groups.

An **IAM user group** is a collection of IAM users. Members receive the permissions defined by the group's policies. Groups contain only IAM users, so nesting groups is unsupported.

| Group | Members |
| --- | --- |
| Developers | Alice, Bob, Charles |
| Audit Team | Charles, David |
| Operations | David, Edward |

Charles and David each belong to two groups. Fred is an IAM user with no group membership. If ten developers need the same EC2 permissions, attach a policy to the `Developers` group and add them as members. One policy update then changes the shared permissions for all ten developers.

## 2. Permissions and policies

### Policy types and evaluation

AWS permissions policies are JSON documents that define allowed or denied actions.

- **Identity-based policies** attach to IAM users, groups, or roles. The attachment determines which identities receive the permissions.
- **Resource-based policies** attach to resources, such as an S3 bucket, and identify the principals to which the statement applies.

Several policies can apply to a request. For an IAM principal, AWS denies a request by default when there is no applicable allow. This is an **implicit deny**. An applicable **explicit deny** overrides an allow from any other policy. [AWS policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)

### Least privilege

**Least privilege** means granting only the permissions needed for a task. This reduces accidental changes and limits the impact of compromised credentials.

- A developer who only needs to view EC2 instances can use `ec2:DescribeInstances`. Omit `ec2:TerminateInstances` and the broad `ec2:*` wildcard.
- An application that only reads S3 objects needs `s3:GetObject` on the relevant objects. It can operate without `s3:DeleteBucket`.

### JSON policy structure

| Top-level element | Purpose |
| --- | --- |
| `Version` | Identifies the policy language version. Use `2012-10-17` for new policies. |
| `Id` | Optionally identifies a resource-based policy, where the service supports this element. |
| `Statement` | Contains one or more permission statements. This element is required. |

| Statement element | Purpose |
| --- | --- |
| `Sid` | Optionally identifies the statement. |
| `Effect` | Sets the result to `Allow` or `Deny`. |
| `Principal` | Identifies the accounts, identities, or services to which a resource-based statement applies. Omit it from identity-based policies. |
| `Action` | Identifies the AWS API operations the statement allows or denies. |
| `Resource` | Identifies the AWS resources in scope, typically by Amazon Resource Name (ARN). |
| `Condition` | Optionally limits when the statement applies based on the request context. |

[AWS JSON policy grammar](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_grammar.html)

### Example read policy

This identity-based policy allows selected read operations for Amazon EC2, Elastic Load Balancing, and Amazon CloudWatch.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "elasticloadbalancing:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

The `2012-10-17` value refers to the policy language. The wildcard in `ec2:Describe*` matches EC2 actions whose names begin with `Describe`. The value `"Resource": "*"` covers all resources applicable to the statement. Actions that lack resource-level permission support require `"*"`.

## 3. Console sign-in and authentication

### AWS account alias

An **account alias** is a memorable name you can use in the IAM user sign-in URL.

| URL type | Example |
| --- | --- |
| Account ID | `https://123456789012.signin.aws.amazon.com/console` |
| Account alias | `https://my-company.signin.aws.amazon.com/console` |

An account can have one alias, which must be unique within its AWS partition. IAM users can continue to sign in at the original account ID URL. The alias affects sign-in only, and IAM policies still control permissions. [AWS account alias documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/console-account-alias.html)

### AWS Management Console multi-session support

After you opt in for a browser, you can keep **up to five AWS console identities** signed in at once. Sessions can belong to the same account or different accounts, and each keeps its own identity and permissions. For example, a cloud engineer can switch among development, staging, and production tabs without signing out each time. [AWS console multi-session documentation](https://docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/multisession.html)

### IAM password policy

An **IAM password policy** governs IAM user console passwords. Root passwords and programmatic access keys are outside its scope. [AWS password policy documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_passwords_account-policy.html)

| Setting | What it controls |
| --- | --- |
| Minimum length | The minimum number of characters in a password. |
| Character requirements | Whether passwords must include uppercase letters, lowercase letters, numbers, or special characters such as `!`, `@`, `#`, and `$`. |
| Self-service password changes | Whether IAM users can change their own passwords. |
| Password expiration | How long passwords can be used before they must be changed. |
| Password reuse prevention | How many previous passwords users are prevented from reusing. |

Use long, unique passwords and consider a password manager. Keep passwords and access keys private. NIST guidance calls for changing passwords when there is evidence of compromise and advises against routine password expiration. [NIST password guidance](https://pages.nist.gov/800-63-4/sp800-63b.html#passwordver)

### Multi-factor authentication (MFA)

**Multi-factor authentication (MFA)** verifies identity using more than one factor. A password is something you know, and a security key or authenticator device is something you have.

```text
IAM User
   |
   v
Enter Password
   |
   v
Complete MFA Verification
   |
   v
Successful Login
```

MFA can block sign-in attempts made with a stolen password alone. Phishing, session hijacking, and stolen session tokens can still pose risks. Enable MFA for human identities, especially administrators, and prefer phishing-resistant methods where supported. Protect physical devices against loss or theft and plan for account recovery with a backup MFA device where supported.

### MFA device options

| Type | Example | How it works |
| --- | --- | --- |
| Virtual authenticator | Google Authenticator, Authy | A software app generates time-based one-time passwords (TOTP). One app can store tokens for multiple accounts and generate codes offline after setup. |
| Passkey or security key | Synced passkey, YubiKey | A FIDO authenticator signs a challenge tied to the legitimate sign-in service. A physical key verifies possession, often with a touch, and sends a cryptographic response automatically. |
| Hardware TOTP token | Thales, formerly sold under the Gemalto brand | A dedicated token or card displays a numeric code the user enters at sign-in. It works without a smartphone and must be provisioned for AWS. |

**TOTP** uses a shared secret established during registration and the current time to generate a code. The user enters the code and AWS checks it. Authenticator app codes typically change every 30 seconds. **FIDO2/WebAuthn** uses public-key cryptography and offers stronger phishing resistance than one-time codes. U2F is an earlier FIDO standard. [AWS MFA device options](https://aws.amazon.com/iam/features/mfa/)

**AWS GovCloud (US)** consists of isolated Regions for eligible government and regulated workloads. GovCloud IAM users can use compatible hardware TOTP tokens, virtual authenticators, or FIDO security keys. AWS currently lists Hypersecu hardware TOTP tokens for GovCloud. Check current compatibility if your study materials show SurePassID. The GovCloud account root user does not support MFA. [AWS GovCloud MFA guide](https://docs.aws.amazon.com/govcloud-us/latest/UserGuide/mfa-device.html), [AWS device compatibility](https://aws.amazon.com/iam/features/mfa/)

## 4. IAM roles and temporary credentials

An **IAM role** is an identity that trusted people, applications, or AWS services can assume. The resulting session uses temporary credentials from **AWS Security Token Service (AWS STS)**. A role has no permanent password or long-term access keys.

| Role policy | Question it answers |
| --- | --- |
| **Trust policy** | Who can assume the role? This resource-based policy identifies trusted principals. |
| **Permissions policy** | What actions can the role session perform, and on which resources? |

To configure a role for an AWS service, create the role, specify the trusted service in its trust policy, grant the permissions it needs, and associate the role with the service. The service or its workload then uses temporary credentials to access the permitted resources. Keep permissions scoped to the workload and keep credentials out of application source code. [AWS policy and role documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_access-management.html)

```text
EC2 Instance
     |
     v
IAM Role
     |
     v
Temporary Credentials
     |
     v
AWS Services
(S3, CloudWatch, etc.)
```

| Role example | What it enables |
| --- | --- |
| **EC2 instance role** | Applications on an EC2 instance can read S3 objects, send logs to CloudWatch Logs, or access other permitted services. EC2 uses an instance profile to associate the role with the instance. |
| **Lambda execution role** | A Lambda function can write to CloudWatch Logs or access an S3 bucket. |
| **CloudFormation service role** | CloudFormation can create, update, or delete stack resources, such as EC2 instances and S3 buckets, on the caller's behalf. |

## 5. Programmatic access with the AWS CLI and SDKs

The **AWS Command Line Interface (AWS CLI)** and **AWS software development kits (SDKs)** both call AWS service APIs. Requests to protected AWS resources require credentials and sufficient permissions.

| Tool | Interface | Typical use |
| --- | --- | --- |
| **AWS CLI** | Terminal commands | Manage resources interactively or automate operations with shell scripts and CI/CD pipelines. |
| **AWS SDK** | Language-specific libraries | Call AWS APIs from application code. The SDK handles request signing and service responses. |

### AWS CLI examples

Upload a local file to an S3 bucket.

```bash
aws s3 cp myfile.txt s3://mybucket/myfile.txt
```

List objects in an S3 bucket.

```bash
aws s3 ls s3://mybucket
```

Describe EC2 instances in the configured Region.

```bash
aws ec2 describe-instances
```

Check the installed CLI version.

```bash
aws --version
```

The AWS CLI is an open-source tool written in Python that uses Botocore. [AWS CLI source code](https://github.com/aws/aws-cli)

### AWS CLI credentials

- **Workforce access** typically uses IAM Identity Center to obtain temporary credentials. Set up a profile with `aws configure sso`, then sign in with `aws sso login --profile <profile-name>`. [AWS CLI IAM Identity Center setup](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
- **Workloads** can use temporary credentials from IAM roles.
- **IAM user access keys** are a long-term option when temporary credentials are unsuitable. To configure the default CLI profile with access keys, use:

```bash
aws configure
```

The command prompts for the access key ID, secret access key, default AWS Region, and output format.

### AWS CloudShell

**AWS CloudShell** is a browser-based terminal that opens from the AWS Management Console. It comes with AWS CLI v2 and tools such as Git and Python. With credential forwarding enabled, the CLI uses your console identity automatically, so you can run commands without entering access keys or running `aws configure`. [AWS CloudShell overview](https://docs.aws.amazon.com/cloudshell/latest/userguide/)

- **Permissions.** IAM controls both access to CloudShell and the AWS API actions you can run from it. For example, permission to launch CloudShell and forward credentials is separate from `s3:ListBucket`, which you need to list objects in a bucket. [CloudShell IAM permissions](https://docs.aws.amazon.com/cloudshell/latest/userguide/sec-auth-with-identities.html)
- **Storage.** The standard environment provides 1 GB of persistent home-directory storage per AWS Region. CloudShell VPC environments use temporary storage. [CloudShell storage quotas](https://docs.aws.amazon.com/cloudshell/latest/userguide/limits.html)

Before running commands that change resources, check which account and identity the CLI is using. The following command returns the account ID and caller ARN.

```bash
aws sts get-caller-identity
```

### AWS SDK example

AWS provides SDKs for JavaScript and TypeScript (including Node.js), Python, Java, .NET (C#), PHP, Ruby, Go, and C++. It also provides SDKs and tools for mobile and IoT development.

The following JavaScript v3 example lists a page of objects in an S3 bucket. In Node.js, the SDK uses its default credential provider chain when credentials are omitted from the client configuration.

```javascript
import { S3Client, ListObjectsV2Command } from "@aws-sdk/client-s3";

const client = new S3Client({ region: "us-east-1" });

const response = await client.send(
  new ListObjectsV2Command({
    Bucket: "mybucket"
  })
);

console.log(response.Contents);
```

## 6. Audit credentials and permissions

| Tool | Scope | What to look for |
| --- | --- | --- |
| **IAM credential report** | Account-wide report covering the root user and IAM users | Password status, last use, and rotation information where applicable. It also shows MFA status and access key status, creation or rotation, and last use. Use it to identify missing MFA and unused or outdated credentials. |
| **IAM Access Advisor** | Last-accessed information for IAM users, groups, roles, and policies | Services covered by applicable permissions and the last recorded access attempts, plus certain management actions where tracked. Use it to identify potentially unused permissions. |

Access Advisor can record denied attempts and covers only the services and actions AWS tracks. Review credentials and permissions periodically. Use Access Advisor alongside audit evidence such as AWS CloudTrail logs before removing access. [AWS credential reports](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_getting-report.html), [AWS last-accessed information](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_last-accessed.html)
