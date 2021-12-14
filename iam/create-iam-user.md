# Create a new IAM User

This example assumes you have created an <a href="verify-administrator-user.md">Administrator IAM user</a> for your AWS Account.


    # Required Input parameters
    IAM_USER="rdsdemo"
    IAM_GROUP="rds-group-access"

    # Identify appropriate policy by `aws iam list-policies --scope AWS`
    # In this example we are granting access manage RDS resources
    RDS_POLICY_ARN="arn:aws:iam::aws:policy/AmazonRDSFullAccess"


## IAM User and IAM Group Creation

    export AWS_PROFILE=administrator
    aws sts get-caller-identity

    # Create new IAM group and attach applicable AWS Policies to perform desired activities
    aws iam create-group --group-name ${IAM_GROUP}
    # The following command has no output
    aws iam attach-group-policy --group-name ${IAM_GROUP} --policy-arn ${RDS_POLICY_ARN}
    aws iam list-attached-group-policies --group-name ${IAM_GROUP}

    # Create new IAM user belonging to the new IAM group
    aws iam list-users
    aws iam create-user --user-name ${IAM_USER}
    aws iam get-user --user-name ${IAM_USER}
    # The following command has no output
    aws iam add-user-to-group --user-name ${IAM_USER} --group-name ${IAM_GROUP}
    aws iam list-groups-for-user --user-name ${IAM_USER}
    aws iam create-access-key --user-name ${IAM_USER} | tee ~/${IAM_USER}.access.json


## Verification of new IAM user

You should be able to validate the new user and access credentials in a different terminal window with

    IAM_USER="rdsdemo"
    export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId ~/${IAM_USER}.access.json)
    export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey ~/${IAM_USER}.access.json)
    aws sts get-caller-identity
    aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]'
    aws rds describe-db-clusters

Access credentials should be added accordingly to necessary configuration for later usage. See <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html">Configuration and credential file settings</a>. For adding this to <code>~/.aws/credentials</code>. This is one way to configure.

    echo "
    [${IAM_USER}]
    aws_access_key_id=${AWS_ACCESS_KEY_ID}
    aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}" >> ~/.aws/credentials

    unset AWS_ACCESS_KEY_ID
    unset AWS_SECRET_ACCESS_KEY
    rm ~/${IAM_USER}.access.json
    env | grep AW
    aws sts get-caller-identity

## Example awscli configuration setup

The following configuration is used in this tutorial when creating RDS resources.

    $ cat ~/.aws/credentials

    [rdsdemo]
    aws_access_key_id=XXXXXXXXXXXXXXXXX
    aws_secret_access_key=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    region=us-east-2
    output=json
    cli_pager=cat


## Teardown

    aws iam remove-user-from-group --user-name ${IAM_USER} --group-name ${IAM_GROUP}
    # NOTE: You may have to redefine AWS_ACCESS_KEY_ID
    aws iam delete-access-key --user-name ${IAM_USER} --access-key-id ${AWS_ACCESS_KEY_ID}
    aws iam delete-user --user-name ${IAM_USER}
    aws iam detach-group-policy --group-name ${IAM_GROUP} --policy-arn ${RDS_POLICY_ARN}
    aws iam delete-group --group-name ${IAM_GROUP}


# References
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/attach-group-policy.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/list-attached-group-policies.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/list-users.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-user.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/add-user-to-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/list-groups-for-user.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-access-key.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/delete-access-key.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/remove-user-from-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/delete-user.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/delete-group.html
