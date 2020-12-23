# AWS Nitro Enclaves Certificate Manager Sample

## Introduction

This sample solution demonstrates a use case for
[AWS Nitro Enclaves](https://aws.amazon.com/ec2/nitro/nitro-enclaves/). Nitro
Enclaves helps customers reduce the attack surface area for their most sensitive
data processing applications. Enclaves offers an isolated, hardened, and highly
constrained environment to host security-critical applications. Nitro Enclaves
includes cryptographic attestation for your software, so that you can be sure
that only authorized code is running, as well as integration with the
[AWS Key Management Service (KMS)](https://aws.amazon.com/kms/), so that only
your enclaves can access sensitive material. Nitro Enclaves are fully isolated
virtual machines, hardened, and highly constrained. They have no persistent
storage, no interactive access, and no external networking. Communication
between your instance and your enclave is done using a secure local channel.
Even a root user or an admin user on the instance will not be able to access or
SSH into the enclave.

This sample solution automates the deployment of a highly available sample web
page behind an AWS Network Load Balancer (NLB). The NLB uses TCP passthrough to
terminate HTTPS/TLS Traffic directly on the EC2 Instance Enclaves. Traffic
between clients and the sample application is encrypted end-to-end in transit.

This sample solution uses
[AWS Certificate Manager (ACM) for Nitro Enclaves](https://aws.amazon.com/ec2/nitro/nitro-enclaves/features/)
integration to automatically issue and provision an x509 certificate from
[Amazon Certificate Manager](https://aws.amazon.com/certificate-manager/) within
the enclaves of an autoscaling group of EC2 instances where it is used to
perform cryptographic operations for the web server processes via the PKCS #11
SSL engine of nginx. At no time is the certificate private key generated for
this solution made available in plaintext to the primary EC2 instance. The
private key is decrypted and utilized only in the Nitro Enclave of each
instance.

Optionally the solution can help configure the EC2 instances for management and
remote access via
[AWS Systems Manager (SSM)](https://aws.amazon.com/systems-manager/).

## Deploying the solution

This solution deploys the following components:

- An Amazon VPC with two public subnets and two private subnets
- An Internet Gateway
- A NAT Gateway
- A Route53 A record
- A Network Access Control List
- A Security Group
- An ACM certificate for the provided domain
- A DNS validation record in Route53 for the certificate
- A sample EC2 Auto Scaling group with two instances using a Nitro Enclaves
  enabled sample Amazon Linux 2 AMI
- A Network Load Balancer that forwards tcp traffic to the Auto Scaling Group
- An IAM role for the EC2 instances to use to communicate S3, ACM, and KMS
- An AWS Lambda function that associates the EC2 instance profile role with the
  ACM certificate.
- An IAM role for Lambda function

### Requirements

**Note:** For easiest deployment you can create a
[Cloud9 environment](https://docs.aws.amazon.com/cloud9/latest/user-guide/create-environment.html),
it already has the below requirements installed. Note, however, that you'll need
a volume with additional free space to accomodate the docker build. 10-20GB of
additional volume space should be sufficient.

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Docker installed](https://www.docker.com/community-edition)
- [SAM CLI installed](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
- [Git installed](https://git-scm.com/downloads)

Additionally you will require:

- Access to an AWS account with permissions to create the above resources
- A Route53 Public Hosted Zone ID
- The name of an available domain within that zone to use for deploying the
  solution and issuing a certificate with Amazon Certificate Manager.

### Deployment Steps

Once you've installed the requirements listed above, open a terminal session as
you'll need to run through a few commands to deploy the solution.

First, we need an S3 bucket where we can upload the Lambda function packaged
as ZIP before we deploy anything - If you don't have a S3 bucket to store code
artifacts then, this is a good time to create one:

```bash
aws s3 --region REPLACE_WITH_YOUR_REGION mb s3://REPLACE_WITH_YOUR_BUCKET_NAME
```

Next, clone the aws-nitro-enclaves-certificate-manager-sample repository to your
local workstation or to your Cloud9 environment.

```bash
git clone https://github.com/aws-samples/aws-nitro-enclaves-certificate-manager-sample.git
```

Next, change directories to the root directory for this example solution.

```bash
cd aws-nitro-enclaves-certificate-manager-sample
```

Next, run the following command to build the Lambda function:

```bash
sam build --use-container
```

Next, run the following command to package the Lambda function to S3:

```bash
sam package \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME
```

Next, the following command will create a Cloudformation Stack and deploy your
SAM resources.

```bash
sam deploy \
  --template-file packaged.yaml\
  --region REPLACE_WITH_YOUR_REGION \
  --stack-name nitro-enclave-acm \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
  HostedZone=REPLACE_WITH_YOUR_ROUTE_53_HOSTED_ZONE_ID \
  DomainName="REPLACE_WITH_YOUR_DOMAIN_NAME"
```

If you wish to enable SSM Agent and configure your instance role management
through the
[AWS Systems Manager Quick Setup](https://console.aws.amazon.com/systems-manager/quick-setup)
add the following option to your command's parameter overrides:

```bash
  SSMConfig="true"
```

You will find all the resources created on the
[AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/home?#/stacks/).

## Testing the solution

Once deployed the solution will output a URL for the sample. After a few
minutes, try visiting that URL. Once the solution has finished its initial
configuration you will be able to view the sample webpage over HTTPS secured by
an ACM issued certificate stored in the enclave of each EC2 instance in the
autoscaling group.

If you wish to manage the solution instances using AWS Systems Manager pass the
`SSMConfig="true"` parameter override when deploying. You can use
[AWS Systems Manager Quick Setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-quick-setup.html)
to quickly configure commonly used Systems Manager capabilities on your Amazon
EC2 instances including:

- A scheduled, bi-weekly update of SSM Agent.
- A scheduled collection of Inventory metadata every 30 minutes.
- A daily scan of your instances to identify missing patches.
- A one-time installation and configuration of the Amazon CloudWatch agent.
- A scheduled, monthly update of the CloudWatch agent.

Systems Manager can also be used to establish a console session on the EC2
instances in the solution's private subnets. To use SSM Quick Setup, specify the
EC2 instance role created by this solution as the
[SSM Quick Setup instance profile role](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-quick-setup.html#quick-setup-instance-profile).
For your SSM Quick Setup target, specify instance tags and enter the
`aws:autoscaling:groupName` key/value pair in the solution stack output.

## Clean up

Once you're done, you can delete the solution going to the
[AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/home#/stacks)
and deleting the `nitro-enclave-acm`. Don't forget to delete the following
artifacts too:

- Delete the
  [CloudWatch log group](https://console.aws.amazon.com/cloudwatch/home#logsV2:log-groups)
  for the Lambda function.
- Consider deleting the Amazon S3 bucket used to store the packaged Lambda
  artifact if you created it on purpose to deploy this solution

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more
information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
