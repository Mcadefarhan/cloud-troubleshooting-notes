# S3 Access Denied for IAM user

## Symptom
An IAM user gets "Access Denied" when trying to list objects from an S3 bucket:

```
$  aws s3 ls s3://demozz

aws: [ERROR]: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: User: arn:aws:iam::random:user/example-user is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::demoz" because no identity-based policy allows the s3:ListBucket action

```

![AccessDenied error in terminal]
<img width="903" height="117" alt="Screenshot 2026-10-30 184306" src="https://github.com/user-attachments/assets/720d3fec-86f8-41bb-854e-b0b101d332fb" />

*AccessDenied error for the IAM user with no policies attached*

## Environment
- AWS S3 bucket (Block Public Access enabled)
- IAM user `test-user` with programmatic access (AWS CLI, run from local machine)
- Region: ap-southeast-2 (Sydney)

## How I reproduced it
Created an S3 bucket with a test file using my admin account. Then created a new IAM user with no policies attached, configured its access key in the AWS CLI, and tried to list the bucket contents.

## Troubleshooting steps
1. Read the error message carefully: it names the action (`s3:ListBucket`), the resource, and says "no identity-based policy allows"
2. Confirmed which identity I was calling as with `aws sts get-caller-identity`: the ARN showed `user/test-user`
3. Checked the IAM user's attached policies: none
4. Checked the bucket policy: none, so nothing allowing access there either
5. Checked for an explicit Deny (bucket policy, SCP, permission boundary): none found

## Root cause
The IAM user had no policy allowing S3 access. IAM denies everything by default unless there is an explicit Allow.

## Fix
Attached a least-privilege inline policy allowing `s3:ListBucket` on the bucket and `s3:GetObject` on its objects:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::<bucket-name>"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::<bucket-name>/*"
    }
  ]
}
```

![Policy attached to the IAM user]
<img width="1460" height="480" alt="Screenshot 2026-10-30 184849" src="https://github.com/user-attachments/assets/cc0a7cfe-471e-454e-8404-aff56b8550de" />

*Permissions tab of the IAM user after attaching the policy*

## Verification
```
farha@LAPTOP-46R52sa MINGW64 ~/downloads
$ aws s3 ls s3://demozz
2026-10-30 18:21:18        152 normal.txt


$ aws s3 cp s3://<bucket-name>/test.txt .
download: s3://<bucket-name>/test.txt to ./test.txt
```

![Successful S3 listing]
<img width="471" height="77" alt="Screenshot 2026-10-30 184913" src="https://github.com/user-attachments/assets/cbece0b5-278c-48c7-99ef-8211fc72e4ee" />

*Successful listing after the policy was attached*

## Key learning
- IAM is **deny by default**. An explicit Allow is needed
- An explicit **Deny** always overrides any Allow
- `s3:ListBucket` applies to the bucket ARN, `s3:GetObject` applies to the object ARN (`/*`)
- The error message tells you the exact action and resource that is missing. Read it first
- Access can come from an identity policy, a bucket policy, or both. Also check SCPs and permission boundaries in organizations

## Commands used
```
aws sts get-caller-identity
aws s3 ls s3://<bucket-name>
aws s3 cp s3://<bucket-name>/test.txt .
aws iam list-attached-user-policies --user-name test-user
```
