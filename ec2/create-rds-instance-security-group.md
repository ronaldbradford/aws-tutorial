# Create Security Group for RDS Instance

This tutorial will create a Security Group that will allow ingress/egress of all RDS traffic (MySQL/MariaDB - 3306, PostgreSQL - 5432, SQL Server - 1433, Oracle -???) within the given VPC for the region you are using. It will also create a separate Security Group that will allow SSH ingress from your present IP.

This example assumes you have created an <a href="verify-administrator-user.md">Administrator IAM user</a> or similar user with EC2 privilege capability for your AWS Account.

## Select AWS user with appropriate EC2 privileges

    export AWS_PROFILE=administrator
    aws sts get-caller-identity


## RDS Instance Security Group

This example will create a security group for use with RDS Instances (all flavors)

    # Input Parameters
    SERVICE="rds"  # What we want to create SGs for

    SG_NAME="${SERVICE}-instance-sg"
    SG_CIDR=""   # If you do not specify a CIDR, the VPC CIDR block is used

    # This example assumes you have only 1 VPC for this account/region  
    VPC_ID=$(aws ec2 describe-vpcs --query '*[0].VpcId' --output text)
    VPC_CIDR=$(aws ec2 describe-vpcs --query '*[0].CidrBlock' --output text)
    SG_CIDR=${SG_CIDR:-$VPC_CIDR}

    OUTPUT=$(aws ec2 create-security-group --group-name ${SG_NAME} --description "${SERVICE} DB instance security group" --vpc-id ${VPC_ID})
    SG_ID=$(jq -r .GroupId <<< ${OUTPUT})
    aws ec2 describe-security-groups --group-ids ${SG_ID}

    for port in 3306 5432 1433 1521; do
      aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port ${port} --cidr ${VPC_CIDR}
    done

    aws ec2 describe-security-groups --group-ids ${SG_ID}

### Optional egress

It is not required to create egress rules as by default an egress rule with full access is created. See the example output below

    aws ec2 revoke-security-group-egress --group-id "${SG_ID}" --ip-permissions '[{"IpProtocol": "-1", "IpRanges" : [{"CidrIp" : "0.0.0.0/0"}] }]'

    for port in 3306 5432 1433 1521; do
      aws ec2 authorize-security-group-egress --group-id ${SG_ID} --protocol tcp --port ${port} --cidr ${SG_CIDR}
    done
    aws ec2 describe-security-groups --group-ids ${SG_ID}


## EC2 Instance for RDS Instance access

An RDS instance is created within a VPC for best security practices. While it is technically possible to enable public access, this is not Well-Architected Framework Security best practice. This is a security group within your VPC that enables SSH ingress from your current IPV4 address (for example a bastian instance).


    VPC_ID=$(aws ec2 describe-vpcs --query '*[0].VpcId' --output text)
    SG_NAME="ec2-${SERVICE}-sg"
    SG_CIDR="$(curl -s http://icanhazip.com)/32"

    OUTPUT=$(aws ec2 create-security-group --group-name ${SG_NAME} --description "EC2 instance for ${SERVICE} access" --vpc-id ${VPC_ID})
    EC2_SG_ID=$(jq -r .GroupId <<< ${OUTPUT})
    aws ec2 describe-security-groups --group-ids ${EC2_SG_ID}
    aws ec2 authorize-security-group-ingress --group-id ${EC2_SG_ID} --protocol tcp --port 22 --cidr ${SG_CIDR}
    aws ec2 describe-security-groups --group-ids ${EC2_SG_ID}


# Example Output

## ec2 describe-security-groups

New Security Group after creation.

    {
       "SecurityGroups": [
           {
               "Description": "RDS DB security group",
               "GroupName": "rds-sg",
               "IpPermissions": [],
               "OwnerId": "999999999999",
               "GroupId": "sg-0f4638496a5176064",
               "IpPermissionsEgress": [
                   {
                       "IpProtocol": "-1",
                       "IpRanges": [
                           {
                               "CidrIp": "0.0.0.0/0"
                           }
                       ],
                       "Ipv6Ranges": [],
                       "PrefixListIds": [],
                       "UserIdGroupPairs": []
                   }
               ],
               "VpcId": "vpc-c860b5a1"
           }
       ]
    }

Security Group after adding Ingres for AWS RDS Instance

    {
        "SecurityGroups": [
            {
                "IpPermissionsEgress": [
                    {
                        "PrefixListIds": [],
                        "FromPort": 1433,
                        "IpRanges": [
                            {
                                "CidrIp": "172.31.0.0/16"
                            }
                        ],
                        "ToPort": 1433,
                        "IpProtocol": "tcp",
                        "UserIdGroupPairs": [],
                        "Ipv6Ranges": []
                    },
                    {
                        "PrefixListIds": [],
                        "FromPort": 5432,
                        "IpRanges": [
                            {
                                "CidrIp": "172.31.0.0/16"
                            }
                        ],
                        "ToPort": 5432,
                        "IpProtocol": "tcp",
                        "UserIdGroupPairs": [],
                        "Ipv6Ranges": []
                    },
                    {
                        "PrefixListIds": [],
                        "FromPort": 3306,
                        "IpRanges": [
                            {
                                "CidrIp": "172.31.0.0/16"
                            }
                        ],
                        "ToPort": 3306,
                        "IpProtocol": "tcp",
                        "UserIdGroupPairs": [],
                        "Ipv6Ranges": []
                    }
                ],
                "Description": "RDS Aurora DB security group",
                "IpPermissions": [
                    {
                        "PrefixListIds": [],
                        "FromPort": 1433,
                        "IpRanges": [
                            {
                                "CidrIp": "172.31.0.0/16"
                            }
                        ],
                        "ToPort": 1433,
                        "IpProtocol": "tcp",
                        "UserIdGroupPairs": [],
                        "Ipv6Ranges": []
                    },
                    {
                        "PrefixListIds": [],
                        "FromPort": 5432,
                        "IpRanges": [
                            {
                                "CidrIp": "172.31.0.0/16"
                            }
                        ],
                        "ToPort": 5432,
                        "IpProtocol": "tcp",
                        "UserIdGroupPairs": [],
                        "Ipv6Ranges": []
                    },
                    {
                        "PrefixListIds": [],
                        "FromPort": 3306,
                        "IpRanges": [
                            {
                                "CidrIp": "172.31.0.0/16"
                            }
                        ],
                        "ToPort": 3306,
                        "IpProtocol": "tcp",
                        "UserIdGroupPairs": [],
                        "Ipv6Ranges": []
                    }
                ],
                "GroupName": "rds-instance-sg",
                "VpcId": "vpc-0a03225585506c531",
                "OwnerId": "356050288086",
                "GroupId": "sg-032dc3d69fb1b2daf"
            }
        ]
    }

# Teardown

A Security Group does not incur an ongoing cost and the initial <a href="https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html">quotas</a> defaults are very adequate. However it is always a best practice to remove unused resources and maintain a clean infrastructure.

    aws ec2 delete-security-group --group-id ${SG_ID}
    aws ec2 delete-security-group --group-id ${EC2_SG_ID}

# References

## awscli
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-vpcs.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/create-security-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-security-groups.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/authorize-security-group-ingress.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/authorize-security-group-egress.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/revoke-security-group-egress.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/delete-security-group.html

## AWS user Guide
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-security-groups.html
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.RDSSecurityGroups.html
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Tutorials.WebServerDB.CreateVPC.html

## IaaS
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_security_group
