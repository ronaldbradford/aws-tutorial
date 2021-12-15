# Verify AWS IAM Administrator User Setup

When first creating a new AWS account, you will initially create a new user. This example validate you have an accessible IAM User that has `AdministratorAccess` level access for the various examples in this tutorial.

    # Verify Administrator User Setup
    export AWS_PROFILE=administrator  
    export AWS_PAGER=cat

    aws sts get-caller-identity
    aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]'
    aws iam list-attached-user-policies --user-name $(aws sts get-caller-identity --query 'Arn' --output text | cut -d'/' -f2)


# Triage

These commands work provided you have:
- Setup an IAM user with `AdministratorAccess`. See <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html">Creating an IAM user in your AWS account</a>
- The access credentials for this IAM user. See <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html">Configuration and credential file settings</a>


      $ cat ~/.aws/credentials
      [administrator]
      aws_access_key_id=XXXXXXXXXXXXXXXXX
      aws_secret_access_key=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      region=us-east-2
      output=json
      cli_pager=cat

# Example Output

## sts get-caller-identity

      {
          "UserId": "AIDAWA6FBRN6XP5AWG4HU",
          "Account": "999999999999",
          "Arn": "arn:aws:iam::999999999999:user/administrator"
      }

## ec2 describe-availability-zones

      us-east-2


## ec2 list-attached-user-policies
      {
          "AttachedPolicies": [
              {
                  "PolicyName": "AdministratorAccess",
                  "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
              }
          ]
      }
