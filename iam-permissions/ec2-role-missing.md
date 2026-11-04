# EC2 cannot access S3 (IAM role missing)

## Symptom
Running an AWS CLI command from inside an EC2 instance fails:

```
$ aws s3 ls s3://pracgtizee
aws: [ERROR]: An error occurred (NoCredentials): Unable to locate credentials. You can configure credentials by running "aws login".
```

<img width="1261" height="35" alt="Unable to locate credentials error" src="https://github.com/user-attachments/assets/d1547285-32df-4c62-82f6-74fe830096fb" />

*Error from inside the EC2 instance with no IAM role attached*

## Environment
- AWS EC2 (t3.micro, Amazon Linux 2023, AWS CLI preinstalled)
- S3 bucket `pracgtizee` with one test object
- Region: ap-southeast-2 (Sydney)
- No IAM role attached to the instance

## How I reproduced it
Launched an EC2 instance without an IAM instance profile, connected over SSH, and ran an S3 command from inside it.

## Troubleshooting steps
1. Read the error: "Unable to locate credentials" means the CLI found no credentials at all. This is different from AccessDenied
2. Ran `aws sts get-caller-identity`: same error, so the instance has no identity
3. Checked the instance in the console (Security tab): **IAM Role** was empty
4. Checked `aws configure list`: no access key or profile configured
5. Decided to attach an IAM role instead of storing access keys on the instance

## Root cause
The instance had no IAM role attached, so the AWS CLI had no credentials to sign requests with.

## Fix
1. Created an IAM role (trusted entity: EC2) with an inline policy allowing `s3:ListBucket` on `pracgtizee` and `s3:GetObject` on `pracgtizee/*`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::pracgtizee"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::pracgtizee/*"
    }
  ]
}
```

2. Attached the role via EC2 → Actions → Security → Modify IAM role

<img width="1507" height="627" alt="Screenshot 2026-11-04 230120" src="https://github.com/user-attachments/assets/d39e9892-da2a-45cb-8e42-3fe6bde161b5" />


*Security tab of the instance showing the attached IAM role*

## Verification
```
$ aws sts get-caller-identity
{
    "UserId": "AROA...:i-...",
    "Account": "<account-id>",
    "Arn": "arn:aws:sts::<account-id>:assumed-role/s3-read-pragtizz/i-..."
}

$ aws s3 ls s3://pracgtizee
2026-11-04 17:12:06        152 normal.txt
```
```

<img width="530" height="50" alt="Successful S3 listing from EC2" src="https://github.com/user-attachments/assets/63565648-0a45-4718-b217-bdf30b3a7eba" />


*Successful listing from the instance after attaching the role*

## Key learning
- "Unable to locate credentials" = no identity found. "AccessDenied" = identity found, but not allowed. The fixes are different
- Use IAM roles for EC2. Never store access keys on an instance
- The role reaches the instance through an **instance profile**, and the credentials come from the instance metadata service (temporary and auto-rotated)
- After attaching a role, `get-caller-identity` shows an `assumed-role` ARN
- If a role is attached but access is still denied, check the role's policy, the bucket policy, and any explicit Deny

## Commands used
```
aws s3 ls s3://pracgtizee
aws sts get-caller-identity
aws configure list
```
