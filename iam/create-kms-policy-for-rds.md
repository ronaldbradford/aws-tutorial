# Create an IAM Policy to manage KMS resources

In this tutorial we will create an IAM policy an attach this to the created `rds-group-access` as described in <a href="create-iam-user.md">Create a new IAM User</a>. This will enable an IAM user that has permissions to manage RDS resources also be able to


## New IAM Policy JSON data
    # rds-kms-policy.json
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
    aws iam attach-group-policy --group-name ${IAM_GROUP} --policy-arn arn:aws:iam::999999999999:policy/${IAM_KMS_POLICY}
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
