# Create a new IAM User

This example assumes you have created an <a href="verify-administrator-user.md">Administrator IAM user</a> for your AWS Account.


## Required configurable parameters

    # Required Input parameters
    IAM_USER="rdsdemo"
    IAM_GROUP="rds-group-access"

    # Identify an appropriate RDS policy
    aws iam list-policies --scope AWS --query 'Policies[*].[PolicyName,Arn]' --output text | grep RDS

    # In this example we are granting access manage RDS resources
    RDS_POLICY_ARN="arn:aws:iam::aws:policy/AmazonRDSFullAccess"

## IAM User and IAM Group Creation

    # Ensure you are using an IAM user with the privilege to create and maintain IAM resources
    export AWS_PROFILE=administrator
    aws sts get-caller-identity

    # Create a new IAM group and attach applicable AWS Policies to perform desired activities
    # This group is used in later tutorials to extend capabilities for this user
    aws iam create-group --group-name ${IAM_GROUP}
    # The following command has no output
    aws iam attach-group-policy --group-name ${IAM_GROUP} --policy-arn ${RDS_POLICY_ARN}
    aws iam list-attached-group-policies --group-name ${IAM_GROUP}

    # Create a new IAM user belonging to the new IAM group
    aws iam list-users
    aws iam create-user --user-name ${IAM_USER}
    aws iam get-user --user-name ${IAM_USER}
    # The following command has no output
    aws iam add-user-to-group --user-name ${IAM_USER} --group-name ${IAM_GROUP}
    aws iam list-groups-for-user --user-name ${IAM_USER}
    aws iam create-access-key --user-name ${IAM_USER} | tee ~/${IAM_USER}.access.json


## Verification of new IAM user

You should be able to validate the new IAM user and the access credentials in a different terminal window with

    IAM_USER="rdsdemo"
    export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId ~/${IAM_USER}.access.json)
    export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey ~/${IAM_USER}.access.json)
    export AWS_DEFAULT_REGION=$(aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]')
    aws sts get-caller-identity
    aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]'
    aws rds describe-db-clusters

The new IAM user access credentials should be added to a necessary configuration for later usage. <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html">Configuration and credential file settings</a> provides information for adding this to <code>~/.aws/credentials</code>.

This is one approach to configuring these settings

    echo "
    [${IAM_USER}]
    region=${AWS_DEFAULT_REGION}
    aws_access_key_id=${AWS_ACCESS_KEY_ID}
    aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}" >> ~/.aws/credentials

    # Reset any current session settings
    unset AWS_ACCESS_KEY_ID
    unset AWS_SECRET_ACCESS_KEY
    unset AWS_DEFAULT_REGION
    unset AWS_PROFILE
    rm -f ~/${IAM_USER}.access.json
    env | grep AWS

    # Test the new awscli profile
    export AWS_PROFILE=${IAM_USER}
    aws sts get-caller-identity

## Example awscli configuration setup

The following configuration is used in these tutorials when creating RDS resources.

    $ cat ~/.aws/credentials

    [rdsdemo]
    region=us-east-2
    aws_access_key_id=XXXXXXXXXXXXXXXXX
    aws_secret_access_key=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx


## Teardown

The following commands will remove the IAM resources created in this tutorial, allowing you to repeat and refine your setup steps.

    aws iam remove-user-from-group --user-name ${IAM_USER} --group-name ${IAM_GROUP}
    # NOTE: You may have to redefine AWS_ACCESS_KEY_ID first
    aws iam delete-access-key --user-name ${IAM_USER} --access-key-id ${AWS_ACCESS_KEY_ID}
    aws iam delete-user --user-name ${IAM_USER}
    aws iam detach-group-policy --group-name ${IAM_GROUP} --policy-arn ${RDS_POLICY_ARN}
    aws iam delete-group --group-name ${IAM_GROUP}


# Example Output

## aws iam list-policies
    AmazonRDSFullAccess	arn:aws:iam::aws:policy/AmazonRDSFullAccess
    AmazonRDSDirectoryServiceAccess	arn:aws:iam::aws:policy/service-role/AmazonRDSDirectoryServiceAccess
    AmazonRDSServiceRolePolicy	arn:aws:iam::aws:policy/aws-service-role/AmazonRDSServiceRolePolicy
    RDSCloudHsmAuthorizationRole	ar77n:aws:iam::aws:policy/service-role/RDSCloudHsmAuthorizationRole
    AmazonRDSPreviewServiceRolePolicy	arn:aws:iam::aws:policy/aws-service-role/AmazonRDSPreviewServiceRolePolicy
    AmazonRDSBetaServiceRolePolicy	arn:aws:iam::aws:policy/aws-service-role/AmazonRDSBetaServiceRolePolicy
    AmazonRDSDataFullAccess	arn:aws:iam::aws:policy/AmazonRDSDataFullAccess
    AWSApplicationAutoscalingRDSClusterPolicy	arn:aws:iam::aws:policy/aws-service-role/AWSApplicationAutoscalingRDSClusterPolicy
    AmazonRDSReadOnlyAccess	arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess
    AWSQuickSightDescribeRDS	arn:aws:iam::aws:policy/service-role/AWSQuickSightDescribeRDS
    AmazonRDSEnhancedMonitoringRole	arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole
    AmazonRDSCustomPreviewServiceRolePolicy	arn:aws:iam::aws:policy/aws-service-role/AmazonRDSCustomPreviewServiceRolePolicy
    AmazonRDSCustomServiceRolePolicy	arn:aws:iam::aws:policy/aws-service-role/AmazonRDSCustomServiceRolePolicy
    AWSElasticBeanstalkRoleRDS	arn:aws:iam::aws:policy/service-role/AWSElasticBeanstalkRoleRDS
    AWSFaultInjectionSimulatorRDSAccess	arn:aws:iam::aws:policy/service-role/AWSFaultInjectionSimulatorRDSAccess
    AmazonRDSPerformanceInsightsReadOnly	arn:aws:iam::aws:policy/AmazonRDSPerformanceInsightsReadOnly


## aws iam create-group

    {
        "Group": {
            "Path": "/",
            "GroupName": "rds-group-access",
            "GroupId": "AGPAVFZSAHXLNRUDAWBO7",
            "Arn": "arn:aws:iam::999999999999:group/rds-group-access",
            "CreateDate": "2022-11-15T15:21:07+00:00"
        }
    }

## aws iam attach-group-policy

    Does not offer output

## aws iam list-attached-group-policies

    {
        "AttachedPolicies": [
            {
                "PolicyName": "AmazonRDSFullAccess",
                "PolicyArn": "arn:aws:iam::aws:policy/AmazonRDSFullAccess"
            }
        ]
    }

## aws iam create-user

    {
        "User": {
            "Path": "/",
            "UserName": "rdsdemo",
            "UserId": "AIDAVFZSAHXLGVPZQ3GMX",
            "Arn": "arn:aws:iam::999999999999:user/rdsdemo",
            "CreateDate": "2022-11-15T15:23:49+00:00"
        }
    }
## aws iam list-groups-for-user

    {
        "Groups": [
            {
                "Path": "/",
                "GroupName": "rds-group-access",
                "GroupId": "AGPAVFZSAHXLNRUDAWBO7",
                "Arn": "arn:aws:iam::999999999999:group/rds-group-access",
                "CreateDate": "2022-11-15T15:21:07+00:00"
            }
        ]
    }

## aws rds describe-db-clusters

    {
        "DBClusters": []
    }


# References

## awscli
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

## User Guide
- https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_rds_region.html
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAM.html
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.IAMPolicy.html
