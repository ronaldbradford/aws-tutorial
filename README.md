# AWS Tutorials

This repository provides several cut/paste reproducible tutorials to demonstrate AWS functionality using the AWS CLI.

# Tutorial Pre-requisites

- An AWS account. The <a href="https://aws.amazon.com/free/">Free 1 year AWS account</a> has sufficient free-tier services for many of these tutorials unless otherwise stated.
- The awscli. See <a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html">Installing AWS CLI Version 2</a>.
- An IAM user that has the privilege to control the authentication and authoriztion to use AWS resources including creating new IAM users, groups and policies. <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html">Creating an IAM user in your AWS account</a> provides example instructions. In these tutorials use the username <i>administrator</i> for demonstration purposes.
- The access credentials for this IAM user. <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html">Configuration and credential file settings</a> provides details for adding this to the <code>~/.aws/credentials</code> configuration file. In these tutorials a section called <i>[administrator]</i> is defined for demonstration purposes.
- A correctly configured VPC. The <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html">get started with Amazon VPC</a> documentation provides instructions for setup.

# Tutorials

## Identity Access Management (IAM)
- <a href="iam/verify-administrator-user.md">Verify your AWS IAM Administrator user setup</a>
- <a href="iam/create-iam-user.md">Create a new IAM User with privileges for managing RDS resources</a>
- <a href="iam/create-kms-policy-for-rds.md">Create a KMS policy for managing your Custom Managed Keys (CMK)</a>

## Elastic Compute Cloud (EC2)
- <a href="ec2/create-rds-security-group.md">Create a VPC Security Group for accessing RDS Aurora resource</a>
- <a href="ec2/create-an-assessible-instance.md">Create an Internet accessible EC2 instance for accessing created AWS resources in your VPC</a>

## Key Management Service (KMS)
- <a href="kms/create-cmk-for-rds.md">Create a Custom Managed Key (CMK) for your RDS clusters</a>

## Relational Database Service (RDS)
- <a href="rds/create-mysql-aurora-cluster.md">Create a simple RDS Aurora MySQL cluster</a>
- <a href="rds/mysql-aurora-cluster-common-commands.md">Aurora cluster common operations</a>
  - <a href="rds/mysql-aurora-logs.md">How to review your database logs</a>
  - <a href="rds/mysql-aurora-events.md">Monitor your database events</a>
  - <a href="rds/create-aurora-snapshot.md">How to create and delete cluster snapshots</a>
  - <a href="rds/mysql-aurora-minor-upgrade.md">How to perform a minor MySQL upgrade</a>
  - <a href="rds/mysql-aurora-major-upgrade.md">How to perform a major MySQL upgrade</a>


# Further References

## AWS Documentation
- https://aws.amazon.com/getting-started/
- https://aws.amazon.com/cli/
- https://aws.amazon.com/free/

## More about the author
- https://github.com/ronaldbradford
- https://ronaldbradford.com
- https://the.error.expert
