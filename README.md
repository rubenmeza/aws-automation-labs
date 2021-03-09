# Automation AWS

## Configure AWS CLI

``` bash
# run and set your keys
$ aws --profile admin configure
```

## Creating Users, Groups and Policies with CloudFormation

``` bash
# validate the template to ensure no syntax errors
$ aws --profile admin cloudformation validate-template --template-body file://cloudformation/access/users-developers.yml

# first create the stack specifying the template file and the necessary IAM capabilities
$ aws --profile admin cloudformation create-stack --stack-name users-group-developers --template-body file://cloudformation/access/users-developers.yml --capabilities "CAPABILITY_IAM" "CAPABILITY_NAMED_IAM"

# now wait for completion
$ aws --profile admin cloudformation wait stack-create-complete --stack-name users-group-developers
$ aws --profile admin cloudformation describe-stacks --stack-name users-group-developers
$ aws --profile admin cloudformation describe-stacks --stack-name users-group-developers --query 'Stacks[].[StackName,StackStatus]' --output text

# describe cloudformation stack resources to see details
$ aws --profile admin cloudformation describe-stack-resources --stack-name users-group-developers

# list IAM users to
$ aws --profile admin iam list-users

# list customer managed policies (not AWS Policies)
$ aws --profile admin iam list-policies --scope Local

# see details of the PolicyDocument
$ aws --profile admin iam get-policy-version --policy-arn <policy arn> --version-id v1
```

## Enabling an MFA Device

```bash
# create the virtual MFA device
$ aws --profile admin iam create-virtual-mfa-device --virtual-mfa-device-name ScooperMFADevice --outfile ./QRCode.png --bootstrap-method QRCodePNG

# enable by attaching to user
$ aws --profile admin iam enable-mfa-device --user-name s.cooper --serial-number <device arn> --authentication-code1 <code> --authentication-code2 <code>

# configure AWS CLI to use this device when assuming roles
$ aws --profile dev configure set role_arn <role arn from dev account>
$ aws --profile dev configure set source_profile admin
$ aws --profile dev configure set mfa_serial <mfa device serial>
$ aws --profile dev configure set region us-east-2

# make a call to the DEV ACCOUNT
$ aws --profile dev ec2 describe-instances
```

## Password Reset

```bash
# for users without a login profile, create one
$ aws --profile admin iam create-login-profile --user-name s.cooper --password Ch4n63M3N0w! --password-reset-required

# change YOUR password
$ aws --profile admin iam change-password --old-password Ch4n63M3N0w! --new-password Iloveprograming!

# change the password for SOMEONE ELSE
$ aws --profile admin iam update-login-profile --user-name a.fowler --password Ch4n63M3N0w! --password-reset-required
```

## Credential Rotation

```bash
# capture old key to delete later
$ aws --profile admin configure get aws_access_key_id

# create new credentials
$ aws --profile admin iam create-access-key --user-name <username>

# set new keys
$ aws --profile admin configure set aws_access_key_id <new access key>
$ aws --profile admin configure set aws_secret_access_key <new secret key>

# delete the old ones (beware eventual consistency)
$ aws --profile admin iam delete-access-key --user-name <username> --access-key-id <old key>
```