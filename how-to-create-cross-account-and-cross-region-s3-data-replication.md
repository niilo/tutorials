# How to create cross account and cross region AWS S3 data replication

In this tutorial we're creating two S3 buckets both on different AWS accounts and different regions, buckets are named:

my-important-data-bucket (AWS account: master-aws-account)
my-important-data-bucket-replica (AWS account: backup-aws-account)

We're going to create replication from my-important-data-bucket to my-important-data-bucket-replica. So that these buckets are in sync. But in case someone deletes master-aws-account's my-important-data-bucket replica bucket will not be deleted. So in case of security breach with one account's keys you still have backup on another AWS account if those keys are not compromised.

It's highly recommended to have very strict access control to backup-aws-account. Delete all api access to backup-aws-account when it's not needed. And use MFA on all accounts that have AWS web access.


## Prerequites

In this tutorial we have two AWS accounts:

- master-aws-account (region: eu-west-1, account id: 111111111111)
- backup-aws-account (region: eu-central-1, account id: 222222222222)

Both account have api users named "api-users-username" and they have AdministratorAccess policy attached.

## S3 data replication configuration

### Create IAM API user and configure AWS command line tools with account profiles

create "api-users-username" IAM api user to both accounts and configure AWS cli for both profiles, remember to set correct default region for both configurations:

```bash
aws configure --profile master-aws-account
aws configure --profile backup-aws-account
```

Assign Administrator profile to both IAM users. ! Remember to drop Administrator role from both API users after completing this tutorial !


### Create S3 buckets

Create one bucket on each AWS account and to different regions:

```bash
aws s3 mb s3://my-important-data-bucket --profile master-aws-account --region eu-west-1
aws s3 mb s3://my-important-data-bucket-replica --profile backup-aws-account --region eu-central-1
```

Enable versioning on both buckets:

```bash
aws s3api put-bucket-versioning --bucket my-important-data-bucket \
--versioning-configuration MFADelete=Enabled,Status=Enabled \
--profile master-aws-account
```

```bash
aws s3api put-bucket-versioning --bucket my-important-data-bucket-replica \
--versioning-configuration MFADelete=Enabled,Status=Enabled \
--profile backup-aws-account
```

### Create IAM profiles and roles for cross account and cross region replicaation

Get Account id for master-aws-account profile

```bash
aws iam get-user --profile master-aws-account | grep "Arn"
```

Response returns Account ID in Arn value. number in between "iam::" and ":user/", like:

```json
"Arn": "arn:aws:iam::111111111111:user/api-users-username"
"Arn": "arn:aws:iam::222222222222:user/api-users-username"
```

In this example account id for master-aws-account is 111111111111 and backup-aws-account is 222222222222

Create S3-bucket-policy-my-important-data-bucket-replica.json:

```json
tee S3-bucket-policy-my-important-data-bucket-replica.json <<EOF
{
	"Version": "2008-10-17",
	"Statement": [
		{
			"Sid": "",
			"Effect": "Allow",
			"Principal": {
				"AWS": "arn:aws:iam::111111111111:root"
			},
			"Action": [
				"s3:ReplicateDelete",
				"s3:ReplicateObject"
			],
			"Resource": "arn:aws:s3:::my-important-data-bucket-replica/*"
		}
	]
}
EOF
```

assign bucket policy to backup-aws-account account's my-important-data-bucket-replica bucket:

```bash
aws s3api put-bucket-policy \
--bucket my-important-data-bucket-replica \
--policy file://S3-bucket-policy-my-important-data-bucket-replica.json \
--profile backup-aws-account
```

Create S3-role-trust-policy.json:

```json
tee S3-role-trust-policy.json <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Principal":{
            "Service":"s3.amazonaws.com"
         },
         "Action":"sts:AssumeRole"
      }
   ]
}
EOF
```

Create IAM role for S3 cross account cross region replication:

```bash
aws iam create-role \
--role-name RoleForS3CrossAccountCrossRegionReplication \
--assume-role-policy-document file://S3-role-trust-policy.json \
--profile master-aws-account
```


Create file S3-role-permissions-policy-my-important-data-bucket.json:

```json
tee S3-role-permissions-policy-my-important-data-bucket.json <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetObjectVersion",
            "s3:GetObjectVersionAcl"
         ],
         "Resource":[
            "arn:aws:s3:::my-important-data-bucket/*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "s3:ListBucket",
            "s3:GetReplicationConfiguration"
         ],
         "Resource":[
            "arn:aws:s3:::my-important-data-bucket"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "s3:ReplicateObject",
            "s3:ReplicateDelete"
         ],
         "Resource":"arn:aws:s3:::my-important-data-bucket-replica/*"
      }
   ]
}
EOF
```

Create IAM policy for S3 cross account cross region replication to my-important-data-bucket

```bash
aws iam create-policy \
--policy-name PolicyForS3CrossAccountCrossRegionReplication-my-important-data-bucket  \
--policy-document file://S3-role-permissions-policy-my-important-data-bucket.json \
--profile master-aws-account
```

This will return json with Arn value that we need for attaching role to policy, in this example returned value Arn is:

```json
        "Arn": "arn:aws:iam::333333333333:policy/PolicyForS3CrossAccountCrossRegionReplication-my-important-data-bucket",
```

Attach created role to created policy

```bash
aws iam attach-role-policy \
--role-name RoleForS3CrossAccountCrossRegionReplication \
--policy-arn arn:aws:iam::333333333333:policy/PolicyForS3CrossAccountCrossRegionReplication-my-important-data-bucket \
--profile master-aws-account
```

Create replication configuration S3-replication-my-important-data-bucket.json:

```json
tee S3-replication-my-important-data-bucket.json <<EOF
{
  "Role": "arn:aws:iam::333333333333:role/RoleForS3CrossAccountCrossRegionReplication",
  "Rules": [
    {
      "Prefix": "",
      "Status": "Enabled",
      "Destination": {
        "Bucket": "arn:aws:s3:::my-important-data-bucket"
      }
    }
  ]
}
EOF
```

Assign S3 cross account and cross region replication bucket policy to my-important-data-bucket

```bash
aws s3api put-bucket-replication \
--bucket my-important-data-bucket \
--replication-configuration file://S3-replication-my-important-data-bucket.json  \
--profile master-aws-account
```

Verify that replication is enabled to my-important-data-bucket:

```bash
aws s3api get-bucket-replication \
--bucket my-important-data-bucket \
--profile master-aws-account
```

## Verify replication

Create and upload test-replication.txt file:

```bash
date > test-replication.txt
aws s3 cp test-replication.txt s3://my-important-data-bucket/ --profile master-aws-account
```

List bucket contents:

```bash
aws s3 ls s3://my-important-data-bucket --profile master-aws-account
aws s3 ls s3://my-important-data-bucket --profile backup-aws-account
```

## Extras

## Sync already existing files

If my-important-data-bucket has been created earlier and there's huge amount of "files" you can replicate / sync these with aws s3 sync command.

Create bucket policy where you give backup-aws-account access to my-important-data-bucket S3-bucket-policy-my-important-data-bucket.json:

```json
tee S3-bucket-policy-my-important-data-bucket.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"AWS": "222222222222"},
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::my-important-data-bucket",
      "arn:aws:s3:::my-important-data-bucket/*"
    ]
  }
}
EOF
```

Assign bucket policy

```bash
aws s3api put-bucket-policy \
--bucket my-important-data-bucket \
--policy file://S3-bucket-policy-my-important-data-bucket.json \
--profile master-aws-account
```

Replicate from my-important-data-bucket to my-important-data-bucket-replica:

```bash
aws s3 sync s3://my-important-data-bucket s3://my-important-data-bucket-replica --profile backup-aws-account --source-region eu-west-1 --region eu-central-1
```

You might want to delete bucket policy after sync is done:

```bash
aws s3api delete-bucket-policy \
--bucket my-important-data-bucket \
--profile master-aws-account
```

## Remove attached IAM profile from api user

List user's profiles:

```bash
aws iam list-attached-user-policies --user-name api-users-username --profile backup-aws-account
```

Detach AdministratorAccess profile from api-users-username user:

```bash
aws iam detach-user-policy --user-name api-users-username \
--policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
--profile backup-aws-account
```

And then same to master-aws-account

## Next steps

How to get alert if someone deletes replication?
