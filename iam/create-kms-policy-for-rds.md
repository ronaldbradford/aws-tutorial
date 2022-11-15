# Create an IAM Policy to manage KMS resources

In this tutorial we will create an IAM policy an attach this to the created `rds-group-access` as described in <a href="create-iam-user.md">Create a new IAM User</a>. This will enable an IAM user that has permissions to manage RDS resources also be able to use KMS and CMK keys.

This example assumes you have created an <a href="verify-administrator-user.md">Administrator IAM user</a> or similar user with IAM create privilege capability for your AWS Account.

## Select AWS user with IAM create privileges

    export AWS_PROFILE=administrator
    aws sts get-caller-identity


## New IAM Policy JSON data
    wget https://gist.githubusercontent.com/ronaldbradford/72a56e24571079cbd0255fe34c0c29e8/raw/167d4b767e77ffb0c5ee561aff0ce74d886efcd3/rds-kms-policy.json
    jq . rds-kms-policy.json

## rds-kms-policy.json    
    {
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Action": [
          "kms:CreateKey",
          "kms:TagResource",
          "kms:CreateAlias"
          ],
        "Resource": "*"
      }
    }

# Create and attach the Policy
    IAM_GROUP="rds-group-access"
    IAM_KMS_POLICY="rds-kms-policy"

    aws iam create-policy --policy-name ${IAM_KMS_POLICY} --policy-document file://${IAM_KMS_POLICY}.json
    ACCOUNT=$(aws sts get-caller-identity | jq -r .Account)

    #This command produces no output
    aws iam attach-group-policy --group-name ${IAM_GROUP} --policy-arn arn:aws:iam::${ACCOUNT}:policy/${IAM_KMS_POLICY}
    aws iam list-attached-group-policies --group-name ${IAM_GROUP}


# Example Output

## iam create-policy

    {
        "Policy": {
            "PolicyName": "rds-kms-policy",
            "PolicyId": "ANPAWA6FBRN62RCNN6SNK",
            "Arn": "arn:aws:iam::999999999999:policy/rds-kms-policy",
            "Path": "/",
            "DefaultVersionId": "v1",
            "AttachmentCount": 0,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2021-12-13T00:59:36+00:00",
            "UpdateDate": "2021-12-13T00:59:36+00:00"
        }
    }




## iam list-attached-Policies

    {
      "AttachedPolicies": [
          {
              "PolicyName": "AmazonRDSFullAccess",
              "PolicyArn": "arn:aws:iam::aws:policy/AmazonRDSFullAccess"
          },
          {
              "PolicyName": "rds-kms-policy",
              "PolicyArn": "arn:aws:iam::999999999999:policy/rds-kms-policy"
          }
      ]
    }


# References
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/attach-group-policy.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/list-attached-group-policies.html
