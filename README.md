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

## VPC Creation with AWS CloudFormation

```bash
# validate template
$ aws --profile admin cloudformation validate-template --template-body file://cloudformation/network/microservices-network.yml

# create the stack for microservices network
$ aws --profile dev cloudformation create-stack --stack-name microservices-network --template-body file://cloudformation/network/microservices-network.yml --parameters ParameterKey=VpcCidrPrefix,ParameterValue=10.0

# wait for the stack to finish
$ aws --profile dev cloudformation wait stack-create-complete --stack-name microservices-network

# list the exports
$ aws --profile dev cloudformation list-exports

# list the exports as table
$ aws --profile dev cloudformation list-exports --query 'Exports[].[Name,Value]' --output table

# list the exports using jq
$ aws --profile dev cloudformation list-exports | jq -r '.Exports[] | "\(.Name): \(.Value)"'
```

## Getting Resource Details with CLI

```bash
# use this to list all VPCs
$ aws --profile dev ec2 describe-vpcs

# filter by tag name
$ aws --profile dev ec2 describe-vpcs --filters "Name=tag:Name,Values=microservices-network"

# capture VPC ID to env variable
$ VPC_ID=$(aws --profile dev ec2 describe-vpcs --filters "Name=tag:Name,Values=microservices-network" --query 'Vpcs[0].VpcId' --output text)

# verify VPC_ID
$ echo ${VPC_ID}

# find subnets using VPC_ID
$ aws --profile dev ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}"

# user --query for cleaner output
$ aws --profile dev ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --query 'Subnets[].[SubnetId, CidrBlock]' --output text

# add scope
$ aws --profile dev ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --query 'Subnets[].[SubnetId, CidrBlock, Tags[?Key==`Scope`].Value]' --output text

# add scope
$ aws --profile dev ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --query 'Subnets[].[SubnetId, CidrBlock, Tags[?Key==`Scope`]|[0].Value]' --output text

# add scope, AZ and available address on same line
$ aws --profile dev ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --query 'Subnets[].[Tags[?Key==`Name`]|[0].Value,SubnetId, CidrBlock,Tags[?Key==`Scope`]|[0].Value, AvailableIpAddressCount, AvailabilityZone]' --output table
```

## Creating Gateways and Routes with AWS Cloudformation

```bash
# validate the template
$ aws --profile dev cloudformation validate-template --template-body file://cloudformation/network/microservices-internet.yml

# create the stack for microservices network internet access
$ aws --profile dev cloudformation create-stack --stack-name microservices-internet --template-body file://cloudformation/network/microservices-internet.yml --parameters ParameterKey=NetworkStack,ParameterValue=microservices-network

# wait for the stack to finish
$ aws --profile dev cloudformation wait stack-create-complete --stack-name  microservices-internet

# describe stack events
$ aws --profile dev cloudformation describe-stack-events --stack-name microservices-internet --query 'StackEvents[].[{Resource:LogicalResourceId, Status:ResourceStatus, Reason:ResourceStatusReason}]' --output table

# capture VPC ID to env variable
VPC_ID=$(aws --profile dev ec2 describe-vpcs --filters "Name=tag:Name,Values=microservices-network" --query 'Vpcs[0].VpcId' --output text)

# show routes
$ aws --profile dev ec2 describe-route-tables --filters "Name=vpc-id,Values=${VPC_ID}"

# show routes with cleaner output
$ aws --profile dev ec2 describe-route-tables --filters "Name=vpc-id,Values=${VPC_ID}" --query 'RouteTables[].[Tags[?Key==`Name`].Value, Associations[].SubnetId]' --output text
```

## Managin Network ACLs with AWS CloudFormation

```bash
# validate the template
$ aws --profile dev cloudformation validate-template --template-body file://cloudformation/network/microservices-security.yml

# create the stack for microservices network internet access
$ aws --profile dev cloudformation create-stack --stack-name microservices-security --template-body file://cloudformation/network/microservices-security.yml --parameters ParameterKey=NetworkStack,ParameterValue=microservices-network

# wait for the stack to finish
$ aws --profile dev cloudformation wait stack-create-complete --stack-name microservices-security

# capture VPC ID  to env variable
$ VPC_ID=$(aws --profile dev ec2 describe-vpcs --filter "Name=tag:Name,Values=microservices-network" --query 'Vpcs[0].VpcId' --output text)

# list network ACLs
$ aws --profile dev ec2 describe-network-acls --filters "Name=vpc-id,Values=${VPC_ID}" "Name=tag:aws:cloudformation:stack-name,Values=microservices-security" --query 'NetworkAcls[].[NetworkAclId,Tags[?Key==`Name`]|[0].Value]' --output text

# list NACL entries
$ aws --profile dev ec2 describe-network-acls --filters "Name=vpc-id,Values=${VPC_ID}" "Name=tag:aws:cloudformation:stack-name,Values=microservices-security" --query 'NetworkAcls[].Entries[]'
```

## Applying Change Sets with AWS Cloudformation

```bash
# validate template
$ aws --profile dev cloudformation validate-template --template-body file://cloudformation/network/microservices-security-2.yml

# create the change set and capture its ID
$ CHANGE_SET=$(aws --profile dev cloudformation create-change-set --change-set-name allow-http-traffic --stack-name microservices-security --template-body file://cloudformation/network/microservices-security-2.yml --parameters ParameterKey=NetworkStack,UsePreviousValue=true --query 'Id' --output text)

# view details of change set
$ aws --profile dev cloudformation describe-change-set --change-set-name ${CHANGE_SET}

# cleaner output
$ aws --profile dev cloudformation describe-change-set --change-set-name ${CHANGE_SET} --query '[StackName,Changes[].ResourceChange]' --output text

# apply the changes and wait for stack update
$ aws --profile dev cloudformation execute-change-set --change-set-name ${CHANGE_SET} && aws --profile dev cloudformation wait stack-update-complete --stack-name microservices-sercurity

# capture VPC ID to env variable
$ VPC_ID=$(aws --profile dev ec2 describe-vpcs --filters "Name=tag:Name,Values=microservices-network" --query 'Vpcs[0].VpcId' --output text)

# describe NACLs
$ aws --profile dev ec2 describe-network-acls --filters "Name=vpc-id,Values=${VPC_ID}" "Name=tag:aws:cloudformation:stack-name,Values=microservices-security" --query 'NetworkAcls[].[NetworkAclId,Tags[?Key==`Name`]|[0].Value]' --output table

# describe the entries in table format
$ aws --profile dev ec2 describe-network-acls --filters "Name=vpc-id,Values=${VPC_ID}" "Name=tag:aws:cloudformation:stack-name,Values=microservices-security" --query 'NetworkAcls[].Entries[]' --output table

# describe entries with customized names & values
$ aws --profile dev ec2 describe-network-acls --filters "Name=vpc-id,Values=${VPC_ID}" "Name=tag:aws:cloudformation:stack-name,Values=microservices-security" --query 'NetworkAcls[].Entries[].{Num:RuleNumber, Rule:RuleAction, Range:CidrBlock, Protocol:Protocol, Egress:Egress, Ports:join(`-`,[to_string(PortRange.From), to_string(PortRange.To)])}' --output table
```