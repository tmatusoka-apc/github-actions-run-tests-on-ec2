# Run a command on a private EC2 instance from GitHub Actions

This repository demonstrates how to run a shell command on a private Amazon EC2 instance from GitHub Actions using AWS Systems Manager Run Command. The instance has no inbound rules and does not require SSH access. A NAT Gateway provides outbound internet access for the instance.

## Repository layout

```text
.
|-- .github/
|   `-- workflows/
|       `-- deploy.yml
|-- cloudformation/
|   `-- EC2.yaml
`-- README.md
```

## Requirements

* An AWS account and AWS CLI configured for the target account
* Permission to create the CloudFormation resources in `cloudformation/EC2.yaml`
* A GitHub repository with Actions enabled

The examples use the `ap-northeast-1` region. The template creates a VPC, public and private subnets, a NAT Gateway, an Elastic IP, and a `t3.micro` EC2 instance. NAT Gateway and Elastic IP charges apply while the stack exists.

## Deploy the AWS resources

From the repository root, create the stack:

```bash
aws cloudformation create-stack \
  --stack-name ec2-private-stack \
  --template-body file://cloudformation/EC2.yaml \
  --capabilities CAPABILITY_IAM \
  --region ap-northeast-1

aws cloudformation wait stack-create-complete \
  --stack-name ec2-private-stack \
  --region ap-northeast-1
```

Retrieve the instance ID:

```bash
aws cloudformation describe-stacks \
  --stack-name ec2-private-stack \
  --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" \
  --output text \
  --region ap-northeast-1
```

## Configure GitHub Actions

In **Settings > Secrets and variables > Actions**, add these repository settings:

* Repository variable `EC2_INSTANCE_ID`: the instance ID returned by CloudFormation
* Repository secret `AWS_ACCESS_KEY_ID`: an AWS access key for a dedicated automation identity
* Repository secret `AWS_SECRET_ACCESS_KEY`: the corresponding secret access key

The AWS identity needs permission to send the `AWS-RunShellScript` command to the target instance and retrieve the command invocation result. Limit permissions to the required resources where possible. For production use, prefer GitHub OIDC and an assumable IAM role over long-lived access keys.

The workflow runs on pushes to `main`, sends `curl -sS https://example.com` through Systems Manager, waits for completion, and prints the standard output in the Actions log. Do not commit AWS credentials or other secrets to this repository.

## Clean up

Delete the stack after testing to avoid ongoing NAT Gateway and Elastic IP charges:

```bash
aws cloudformation delete-stack \
  --stack-name ec2-private-stack \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name ec2-private-stack \
  --region ap-northeast-1
```