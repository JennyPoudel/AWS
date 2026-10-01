# STS example: a user with no permissions borrows a role to reach S3

This folder is a hands-on exercise in AWS STS (Security Token Service). The goal:

1. Create an IAM user (`machine-u`) that is allowed to do **nothing**.
2. Create a role that **is** allowed to use S3.
3. Let `machine-u` "assume" (borrow) that role and get temporary credentials.
4. Use those temporary credentials to reach S3.

Everything below is what I actually ran, in order, including the errors I hit and how each one was fixed.


## The values used in this exercise

| Thing | Value |
|---|---|
| AWS account ID | `380301679044` |
| Region | `us-east-1` |
| IAM user with no permissions | `machine-u` |
| CloudFormation stack name | `my-sts-stack` |
| Bucket created by the stack | `jp-sts-example-380301679044` |
| Role created by the stack | `my-sts-stack-StsRole-GCcBvurfiUQ7` |
| Role ARN | `arn:aws:iam::380301679044:role/my-sts-stack-StsRole-GCcBvurfiUQ7` |

The random suffix on the role name (`GCcBvurfiUQ7`) is added by CloudFormation. If the stack is deleted and created again, the suffix changes, and the ARN must be updated in `policy.json` and in the `assume-role` command.

## Words I needed to learn first

- **IAM user**: a permanent identity with a long-lived access key. `machine-u` is one.
- **Root user**: the account owner (the email used to open the AWS account). It can do everything. Its ARN ends in `:root`.
- **Role**: an identity with permissions but no password or permanent keys. Nobody "is" a role; you temporarily assume it.
- **STS**: the service that hands out temporary credentials when you assume a role.
- **ARN**: the full unique name of an AWS resource, for example `arn:aws:iam::380301679044:user/machine-u`.
- **Policy**: a JSON document that says which actions are allowed on which resources.
- **Trust policy**: the policy on a role that says **who is allowed to assume it**.
- **Profile**: a named set of credentials on my laptop. I pick one with `--profile <name>`. With no `--profile`, the CLI uses the one called `default`.
- **CloudFormation stack**: a group of AWS resources created together from a template file.

## The files in this folder

| File | What it is |
|---|---|
| `template.yaml` | CloudFormation template. Creates the S3 bucket and the role. |
| `bin/deploy` | Small script that deploys `template.yaml` as the stack `my-sts-stack`. |
| `policy.json` | Policy attached to `machine-u` that allows it to assume the role. |
| `README.md` | This file. |

## The three profiles on my laptop

The AWS CLI keeps its settings in two files in my home folder:

- `~/.aws/config` holds settings such as the region.
- `~/.aws/credentials` holds access keys.

By the end of the exercise I had three profiles:

| Profile | Who it is | How it was set up | Used for |
|---|---|---|---|
| `default` | root (`arn:aws:iam::380301679044:root`) | `aws login` in the browser | Admin work: creating the user, deploying the stack, attaching the policy |
| `sts` | `machine-u` | Access key pasted into `~/.aws/credentials` | Calling `assume-role` |
| `assumed` | The role, temporarily | Temporary credentials pasted into `~/.aws/credentials` | Reaching S3 |

To check who a profile is at any time:

```bash
aws sts get-caller-identity                    # default profile
aws sts get-caller-identity --profile sts      # machine-u
aws sts get-caller-identity --profile assumed  # the role
```

---

## Step 1: Create a user with no permissions

Run as the admin (default) profile.

```bash
aws iam create-user --user-name machine-u --output table
```

This creates the user. A new user has no permissions at all.

## Step 2: Create an access key for the user

```bash
aws iam create-access-key --user-name machine-u --output table
```

The output shows an `AccessKeyId` (starts with `AKIA`) and a `SecretAccessKey`. The secret is shown **only this once**. Copy both straight into the credentials file in the next step. Do not save them anywhere else.

## Step 3: Save the key as a profile called `sts`

Open the credentials file:

```bash
open ~/.aws/credentials
```

Add this block, using the two values from Step 2:

```ini
[sts]
aws_access_key_id = <YOUR_ACCESS_KEY_ID>
aws_secret_access_key = <YOUR_SECRET_ACCESS_KEY>
```

What I originally did was run `aws configure`, which saves the key under `[default]`, and then edit the file to rename `[default]` to `[sts]`. A shorter way that gives the same result is:

```bash
aws configure --profile sts
```

## Step 4: Check who I am, and that the user really can do nothing

```bash
aws sts get-caller-identity --profile sts
```

Expected output:

```json
{
    "UserId": "***",
    "Account": "***",
    "Arn": "***"
}
```

Now try S3:

```bash
aws s3 ls --profile sts
```

Expected output is an error, and that is the point of the exercise:

```
An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::380301679044:user/machine-u is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
```

## Step 5: Give the default profile a region and admin credentials

The deploy script uses the `default` profile, which started out empty. Two things were missing.

**Region.** Without it the deploy failed with `NoRegion`. Fix:

```bash
aws configure set region us-east-1
```

**Credentials.** Without them the deploy failed with `NoCredentials`. Fix:

```bash
aws login
```

`aws login` opens the browser. Sign in, approve, then go back to the terminal and wait until it prints:

```
Updated profile default to use arn:aws:iam::380301679044:root credentials.
```

Do not press Ctrl+C while it is waiting.

This is why I "became root": `aws login` saves whoever signed in to the browser as the `default` profile, and I signed in as the root account. My `machine-u` key was not lost; it was always in the `sts` profile.

## Step 6: Write the CloudFormation template

`template.yaml` creates two resources:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: create a role for us to assume and create a resource that we will have access to
Parameters:
  BucketName:
    Type: String
    Description: The name of the bucket to create
    Default: jp-sts-example-380301679044
Resources:
  S3Bucket:
   Type: 'AWS::S3::Bucket'
   Properties:
    BucketName: !Ref BucketName
  StsRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS: arn:aws:iam::380301679044:user/machine-u
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: S3access
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: 's3:*'
                Resource: [!Sub 'arn:aws:s3:::*',!Sub 'arn:aws:s3:::${BucketName}/*', !Sub 'arn:aws:s3:::${BucketName}']
```

What each part does:

- **`Parameters` / `BucketName`**: a value I can change without editing the rest of the template. `Default` is the value used when the stack is first created.
- **`S3Bucket`**: creates the bucket. `!Ref BucketName` means "use the value of the `BucketName` parameter".
- **`StsRole`**: creates the role.
- **`AssumeRolePolicyDocument`**: the trust policy. `Principal: AWS: ...user/machine-u` means only `machine-u` may assume this role.
- **`Policies`**: what the role is allowed to do once assumed. `!Sub` fills in `${BucketName}` with the parameter value.
- **`arn:aws:s3:::${BucketName}`** is the bucket itself (needed for listing it). **`arn:aws:s3:::${BucketName}/*`** is the objects inside it (needed for upload and download). Both are needed.

> **Note on `arn:aws:s3:::*`**: I added this so that plain `aws s3 ls` works for the role. It works, but it gives the role every S3 action on **every bucket in the account**, not just the example bucket. The narrower way to get the same result is to remove `arn:aws:s3:::*` from that list and add a second statement instead:
>
> ```yaml
>               - Effect: Allow
>                 Action: 's3:ListAllMyBuckets'
>                 Resource: '*'
> ```

Check a template for syntax errors before deploying:

```bash
aws cloudformation validate-template --template-body file://template.yaml
```

This only checks the format. It does not catch problems like an illegal bucket name.

## Step 7: Write the deploy script and make it runnable

`bin/deploy`:

```bash
#! /bin/bash

aws cloudformation deploy \
--template-file template.yaml \
 --stack-name my-sts-stack \
 --capabilities CAPABILITY_IAM \
```

- **`aws cloudformation deploy`** creates the stack if it does not exist, or updates it if it does.
- **`--template-file template.yaml`** is the template to use. The path is relative, so the script must be run from this folder.
- **`--stack-name my-sts-stack`** is the name of the stack.
- **`--capabilities CAPABILITY_IAM`** is required because the template creates an IAM role. Without it CloudFormation refuses.
- **`\`** at the end of a line means "the command continues on the next line". The last line does not need one.
- There is no `--profile`, so the script runs as the `default` profile (root).

A new script file is not runnable until it is given permission. Do this once:

```bash
chmod u+x bin/deploy
```

## Step 8: Deploy the stack

From this folder:

```bash
./bin/deploy
```

Expected output:

```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - my-sts-stack
```

Check what was created:

```bash
aws cloudformation describe-stack-resources --stack-name my-sts-stack \
  --query 'StackResources[].[LogicalResourceId,PhysicalResourceId,ResourceStatus]' --output table
```

There should be two rows: `S3Bucket` and `StsRole`. The `PhysicalResourceId` of `StsRole` is the role name, which goes into the role ARN used in the next steps.

## Step 9: Allow machine-u to assume the role

`policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::380301679044:role/my-sts-stack-StsRole-GCcBvurfiUQ7"
    }
  ]
}
```

Attach it to the user. This runs as the default (admin) profile, because `machine-u` cannot give permissions to itself:

```bash
aws iam put-user-policy \
  --user-name machine-u \
  --policy-name StsAssumePolicy \
  --policy-document file://policy.json
```

There is no output when it succeeds.

**Assuming a role needs permission on both sides:**

1. The **user** must be allowed to call `sts:AssumeRole` on that role. That is `policy.json`.
2. The **role** must trust that user. That is the `Principal` in `template.yaml`.

If either side is missing, `assume-role` fails with `AccessDenied`.

## Step 10: Assume the role

Run as `machine-u`:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::380301679044:role/my-sts-stack-StsRole-GCcBvurfiUQ7 \
  --role-session-name s3-sts \
  --profile sts
```

- **`--role-arn`** is the role to assume.
- **`--role-session-name`** is any label I choose. It shows up in the identity afterwards.
- **`--profile sts`** makes the call as `machine-u`.

The output looks like this (secrets replaced):

```json
{
    "Credentials": {
        "AccessKeyId": "ASIA<...>",
        "SecretAccessKey": "<TEMPORARY_SECRET>",
        "SessionToken": "<LONG_TOKEN>",
        "Expiration": "2026-10-01T06:59:23+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAVRC57DHCLAN73A4SJ:s3-sts",
        "Arn": "arn:aws:sts::380301679044:assumed-role/my-sts-stack-StsRole-GCcBvurfiUQ7/s3-sts"
    }
}
```

These are temporary credentials:

- The access key starts with `ASIA` (temporary). Permanent keys start with `AKIA`.
- There are **three** values, not two. The `SessionToken` is required.
- They stop working at the `Expiration` time, which is one hour after the command was run. After that, run `assume-role` again.

## Step 11: Save the temporary credentials as a profile called `assumed`

```bash
open ~/.aws/credentials
```

Add:

```ini
[assumed]
aws_access_key_id = <AccessKeyId from Step 10>
aws_secret_access_key = <SecretAccessKey from Step 10>
aws_session_token = <SessionToken from Step 10>
```

The credentials file is **not JSON**. Copy only the value: no quotes and no comma at the end of the line.

## Step 12: Use the role

Check who I am now:

```bash
aws sts get-caller-identity --profile assumed
```

```json
{
    "UserId": "AROAVRC57DHCLAN73A4SJ:s3-sts",
    "Account": "380301679044",
    "Arn": "arn:aws:sts::380301679044:assumed-role/my-sts-stack-StsRole-GCcBvurfiUQ7/s3-sts"
}
```

The ARN now says `assumed-role`, followed by the role name and my session name.

List buckets, which `machine-u` could not do in Step 4:

```bash
aws s3 ls --profile assumed
```

```
2026-09-26 21:34:19 jenishap-bucket
2026-10-01 11:13:10 jp-sts-example-380301679044
```

List the example bucket (prints nothing while it is empty):

```bash
aws s3 ls s3://jp-sts-example-380301679044 --profile assumed
```

Upload a file and list again:

```bash
echo "hello" > hello.txt
aws s3 cp hello.txt s3://jp-sts-example-380301679044/ --profile assumed
aws s3 ls s3://jp-sts-example-380301679044 --profile assumed
```

**What this proves:** the same person at the same laptop is denied as `machine-u` (`--profile sts`) and allowed as the role (`--profile assumed`). The permissions belong to the role, and they are only on loan for an hour.

---

## Errors I hit and what each one meant

| Error message | Cause | Fix |
|---|---|---|
| `NoRegion: You must specify a region` | The profile had no region. | `aws configure set region us-east-1` |
| `NoCredentials: Unable to locate credentials` | The `default` profile had no credentials; my key was in the `sts` profile. | `aws login`, or add `--profile sts` |
| `ls: la: No such file or directory` | Typed `ls la` without the dash. | `ls -la` |
| `Failed to create/update the stack` | This message never says why. The real reason is in the stack events. | Run the "find the real reason" command below |
| `The specified value for policyName is invalid` | `PolicyName: S3 access` had a space. | Renamed to `S3access` |
| Stack stuck in `ROLLBACK_COMPLETE` | The very first create failed. A stack in this state cannot be updated. | Delete the stack, then deploy again |
| `Template format error: YAML not well-formed` | `Type:'AWS::S3::Bucket'` had no space after the colon. | YAML needs `key: value` with a space |
| `Bucket name should not contain uppercase characters` | The bucket was named `MyBucket`. | Bucket names must be lowercase and unique across all of AWS |
| Same uppercase error after changing `Default` | On an update, `deploy` reuses the parameter value stored on the stack and ignores a new `Default`. | Delete and recreate the stack, or add `--parameter-overrides BucketName=<name>` to the deploy command |
| `At least one Resources member must be defined` | Ran the deploy while the template was half-edited and `Resources:` was empty. | Finish and save the file first |
| `AccessDenied ... sts:AssumeRole` | The role's trust policy named `s3.amazonaws.com` instead of `machine-u`, and I had edited the file but not redeployed. | Set `Principal` to the user's ARN, then run `./bin/deploy` |
| `assume-role` command not accepted | Wrote `--role arn` (should be `--role-arn`), left a `/` at the end of the ARN, and left out the `\` at line ends. | Use the command in Step 10 |
| `policy.json` rejected | It was not valid JSON: no outer `{ }`, single quotes, and `Resourece` misspelled. | Use the file in Step 9 |
| `IncompleteSignature ... Invalid key=value pair` | Trailing commas copied from the JSON output into `~/.aws/credentials`. | Remove the commas |
| `AccessDenied ... s3:ListAllMyBuckets` (as the role) | Plain `aws s3 ls` lists every bucket in the account, and the role was only allowed on one bucket. | Name the bucket: `aws s3 ls s3://<bucket>`, or allow `s3:ListAllMyBuckets` |

### Find the real reason a deploy failed

```bash
aws cloudformation describe-stack-events --stack-name my-sts-stack \
  --query 'StackEvents[?contains(ResourceStatus,`FAILED`)].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

### Delete a stack that is stuck

```bash
aws cloudformation delete-stack --stack-name my-sts-stack
aws cloudformation wait stack-delete-complete --stack-name my-sts-stack
```

## Things to remember

- No `--profile` means the `default` profile. Check with `aws sts get-caller-identity` before running anything important.
- Editing `template.yaml` changes nothing in AWS until `./bin/deploy` is run again.
- A parameter's `Default` only applies when the stack is first created.
- Assuming a role needs a policy on the user **and** a trust policy on the role.
- Temporary credentials have three parts and expire after one hour.
- `aws s3 ls` (all buckets) and `aws s3 ls s3://bucket` (one bucket) are different permissions.
- Secrets go in `~/.aws/credentials`, never in a file inside the project.
- Using the root account for everyday CLI work is risky. An IAM user with admin rights is the safer choice for the admin steps.

## Clean up when finished

Run as the default (admin) profile, in this order.

1. Empty the bucket. CloudFormation cannot delete a bucket that still has files in it:

   ```bash
   aws s3 rm s3://jp-sts-example-380301679044 --recursive
   ```

2. Delete the stack. This removes the bucket and the role:

   ```bash
   aws cloudformation delete-stack --stack-name my-sts-stack
   aws cloudformation wait stack-delete-complete --stack-name my-sts-stack
   ```

3. Remove the policy from the user:

   ```bash
   aws iam delete-user-policy --user-name machine-u --policy-name StsAssumePolicy
   ```

4. Delete the user's access key, then the user:

   ```bash
   aws iam list-access-keys --user-name machine-u
   aws iam delete-access-key --user-name machine-u --access-key-id <ACCESS_KEY_ID>
   aws iam delete-user --user-name machine-u
   ```

5. Remove the `[sts]` and `[assumed]` blocks from `~/.aws/credentials`.
