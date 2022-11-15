# Create an RDS Aurora MySQL Cluster

This tutorial will demonstrate a way to create a simple MySQL RDS cluster. A subsequent tutorial will provide details for more robust cluster setup.

## RDS Cluster Requirements

An RDS cluster has a number of required AWS resources in order to be successfully created and accessed. Some of these resources require additional IAM policy permissions in addition to the AWS Managed policy by `AmazonRDSFullAccess` that is demonstrated in <a href="../iam/create-iam-user.md">this tutorial</a>.

<ul>
<li>A VPC that has correctly configured subnets. <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html">Get started with Amazon VPC</a> provides an example setup.</li>
<li>An IAM user with applicable privileges to create RDS resources. See the tutorial <a href="../iam/create-iam-user.md">Create a new IAM User</a></li>
<li>An EC2 security group that enables Aurora cluster ingress (MySQL - 3306, PostgreSQL - 5432) within your VPC. See the tutorial <a href="../ec2/create-rds-security-group.md">Create RDS Security Group</a> for how to create this in your AWS account.</li>
<li>A DB subnet group based on the applicable VPC subnets. Described in this tutorial.</li>
<li>A cluster parameter group. AWS does provide a default that is used in this tutorial.</li>
<li>An instance parameter group. AWS does provide a default that is used in this tutorial.</li>
<li>A KMS Custom Managed Key (CMK). AWS does provide an RDS default KMS key that is used in this tutorial.</li>
<li>One or more RDS instances to access the cluster from a suitable client. Described in this tutorial. </li>
<li>An EC2 instance within the VPC to access the cluster. See the tutorial <a href="../ec2/create-an-assessible-instance.md">Create an Internet Accessible EC2 instance in your VPC</a> for the appropriate setup.
</ul>

## Minimal better practices

While this example is designed to create an RDS cluster in the simplest way, some additional best practice decisions are made.

- All data should always be encrypted. By default a new RDS Aurora cluster is not using encrypted storage which is does not match the AWS Well-Architected Framework best practices for security.
- While not required to specify an engine version, the default version chosen is the oldest version for the engine type, not the most recent version. This is also not a best practice for security.

A subsequent tutorial will provide a number of better practices for creating a more robust and extensible RDS cluster.

## IAM user validation
It is possible to execute the setup of the RDS cluster from your local machine. You will not be able to perform the validation of and use the cluster. For simplicity all installation is performed on an instance that can complete the tutorial.

    # Connect to EC2 Instance inside of VPC
    ssh ${IP}

    # Validate IAM User
    export AWS_PROFILE="rdsdemo"
    aws sts get-caller-identity
    aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]'
    aws rds describe-db-clusters

## Required configurable parameters

    # Necessary configuration to create an example RDS cluster
    ENGINE="aurora-mysql"

    # Look at possible versions for engine
    aws rds describe-db-engine-versions --engine ${ENGINE} --query '*[].EngineVersion' --output text
    ENGINE_VERSION=$(aws rds describe-db-engine-versions --engine ${ENGINE} --query '*[].EngineVersion' --output text | awk '{print $NF}') # Not required, but needed for Well-Architected Framework
    INSTANCE_TYPE="db.t3.medium"  # THIS IS NOT part of the AWS Free Tier
    CLUSTER_ID="rds-${ENGINE}-demo"
    INSTANCE_ID="${CLUSTER_ID}-0"

    SUBNET_GROUP="${CLUSTER_ID}-sng"
    SG_NAME="rds-aurora-sg"
    KMS_KEY_ID="alias/aws/rds"  # Not required, but needed for Well-Architected Framework

    DBA_USER="dba"
    DBA_PASSWD=$(date |md5sum - | cut -c1-20)
    echo "${DBA_PASSWD}"

    # This simple example assumes there is one VPC
    VPC_ID=$(aws ec2 describe-vpcs --query '*[0].VpcId' --output text)

    # This simple example assumes you have only one subnet per AZ for this VPC
    # A more advanced --filter would include for example: Name=tag:tier,Values=db
    SUBNET_IDS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=${VPC_ID} | jq '.Subnets[].SubnetId' | tr '\n' ',' | sed -e "s/,\$"//)
    echo ${VPC_ID},${SUBNET_IDS}

## Setup
    # Create an RDS DB Subnet Group
    aws rds create-db-subnet-group \
        --db-subnet-group-name ${SUBNET_GROUP} \
        --db-subnet-group-description "Subnets for ${CLUSTER_ID}" \
        --subnet-ids "[${SUBNET_IDS}]"
    aws rds describe-db-subnet-groups  --db-subnet-group-name ${SUBNET_GROUP}

    # Obtain the Security Group Id for the RDS created security group name
    SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=${SG_NAME} --query '*[].GroupId' --output text)

    # While the default is to create an unencrypted RDS Aurora cluster, this breaks the AWS Well-Architected Framework Security best practices.
    # --storage-encrypted and --kms-key-id are used to improve data security at rest.
    aws rds create-db-cluster \
      --db-cluster-identifier ${CLUSTER_ID} \
      --vpc-security-group-ids ${SG_ID} \
      --db-subnet-group-name ${SUBNET_GROUP}  \
      --engine ${ENGINE} \
      --engine-version ${ENGINE_VERSION} \
      --master-username ${DBA_USER} \
      --master-user-password ${DBA_PASSWD} \
      --storage-encrypted \
      --kms-key-id ${KMS_KEY_ID}

    # It is not possible to use the `aws rds wait` command for a cluster creation to proceed with subsequent statements.
    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].Status' --output text)
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done

    # An RDS Cluster requires one or more instances to be able to access the cluster.
    aws rds create-db-instance \
      --db-cluster-identifier ${CLUSTER_ID} \
      --db-instance-identifier ${INSTANCE_ID} \
      --db-instance-class ${INSTANCE_TYPE} \
      --engine ${ENGINE}

    time aws rds wait db-instance-available --db-instance-identifier ${INSTANCE_ID}

    # Alternative method to `rds wait` that is similar to above example
    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].DBInstanceStatus' --output text)
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done

## Warning
While the AWS Free tier provides for a t2.micro instance for RDS, this instance class is not supported for RDS Aurora. You can see the list of valid <a href="https://aws.amazon.com/ec2/instance-types/t3/">burstable t3 instance types</a> for the Aurora MySQL version with:

    aws rds describe-orderable-db-instance-options --engine ${ENGINE} --engine-version ${MYSQL_VERSION} --query '*[].DBInstanceClass' | grep db.t

For each MySQL version you will need to verify which instance types are supported. See <a href="mysql-aurora-major-upgrade.md">Aurora MySQL Major Upgrade</a> for example difference with this tutorial.

## Tip
New instance types are regularly added, for example the t4g instance type was just announced when this tutorial was created.


# Validation of your new RDS Cluster

Your RDS cluster will not be accessible from your local machine if you performed the setup. The tutorial <a href="../ec2/create-an-assessible-instance.md">Create an Internet Accessible EC2 instance in your VPC</a> provides an example of how to be ableo to connect to with a client program.

    # Each RDS Clusters has at least two endpoints, one endpoint that connects to the Writer instance
    # and one reader endpoint that connects in a round-robin to all Reader instances.  
    # When there are no reader instances, this points to the Writer.
    CLUSTER_ENDPOINT=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].Endpoint' --output text)
    CLUSTER_READER_ENDPOINT=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].ReaderEndpoint' --output text)
    echo ${CLUSTER_ENDPOINT},${CLUSTER_READER_ENDPOINT}

    # This will ensure the security group(s) for the cluster is appropriately configured
    nc -vz ${CLUSTER_ENDPOINT} 3306

    # This is an insecure means of connecting to MySQL by passing the password on the command line. Used only for cut/paste repeatable demonstration purposes.
    docker run -it --rm mysql mysql -h${CLUSTER_ENDPOINT} -u${DBA_USER} -p${DBA_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"
    docker run -it --rm mysql mysql -h${CLUSTER_READER_ENDPOINT} -u${DBA_USER} -p${DBA_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"

    # Each Instance within a cluster has an endpoint. While your application should use cluster endpoints, this endpoint is useful for monitoring
    INSTANCE_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].Endpoint.Address' --output text)
    docker run -it --rm mysql mysql -h${INSTANCE_ENDPOINT} -u${DBA_USER} -p${DBA_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"

## Example validation output

    +-------------------------+------------------+-----------+-------------------+--------------------+
    | @@aurora_server_id      | @@aurora_version | VERSION() | USER()            | @@innodb_read_only |
    +-------------------------+------------------+-----------+-------------------+--------------------+
    | rds-aurora-mysql-demo-0 | 2.10.0           | 5.7.12    | dba@172.31.39.139 |                  0 |
    +-------------------------+------------------+-----------+-------------------+--------------------+

# Next Steps

The tutorial <a href="mysql-aurora-cluster-common-commands.md">MySQL aurora cluster common commands</a> provides a number of common tasks you may use with this cluster.

# Example Output

## aws rds describe-db-engine-versions --engine aurora-mysql

    {
        "DBEngineVersions": [
            {
                "Engine": "aurora-mysql",
                "EngineVersion": "5.7.mysql_aurora.2.04.0",
                "DBParameterGroupFamily": "aurora-mysql5.7",
                "DBEngineDescription": "Aurora MySQL",
                "DBEngineVersionDescription": "Aurora (MySQL 5.7) 2.04.0",
                "ValidUpgradeTarget": [
                    {
                        "Engine": "aurora-mysql",
                        "EngineVersion": "5.7.mysql_aurora.2.04.1",
                        "Description": "RDS Aurora (MySQL)",
                        "AutoUpgrade": false,
                        "IsMajorVersionUpgrade": false,
                        "SupportedEngineModes": [
                            "provisioned"
                        ],
                        "SupportsParallelQuery": false,
                        "SupportsGlobalDatabases": false,
                        "SupportsBabelfish": false
                    },
                    {
                        "Engine": "aurora-mysql",
                        "EngineVersion": "5.7.mysql_aurora.2.04.2",
                        "Description": "RDS Aurora (MySQL)",
                        "AutoUpgrade": false,
                        "IsMajorVersionUpgrade": false,
                        "SupportedEngineModes": [
                            "provisioned"
                        ],
                        "SupportsParallelQuery": false,
                        "SupportsGlobalDatabases": false,
                        "SupportsBabelfish": false
                    },

## rds describe-db-clusters

      $ aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID}
      {
          "DBClusters": [
              {
                  "MasterUsername": "dba",
                  "ReaderEndpoint": "rds-aurora-mysql-demo.cluster-ro-c2qmmyq8rslr.us-east-2.rds.amazonaws.com",
                  "HttpEndpointEnabled": false,
                  "ReadReplicaIdentifiers": [],
                  "VpcSecurityGroups": [
                      {
                          "Status": "active",
                          "VpcSecurityGroupId": "sg-78b81711"
                      }
                  ],
                  "CopyTagsToSnapshot": false,
                  "HostedZoneId": "Z2XHWR1WZ565X2",
                  "EngineMode": "provisioned",
                  "Status": "available",
                  "MultiAZ": false,
                  "LatestRestorableTime": "2021-12-12T22:03:36.962Z",
                  "DomainMemberships": [],
                  "PreferredBackupWindow": "05:09-05:39",
                  "DBSubnetGroup": "rds-aurora-mysql-demo-sng",
                  "AllocatedStorage": 1,
                  "ActivityStreamStatus": "stopped",
                  "BackupRetentionPeriod": 1,
                  "PreferredMaintenanceWindow": "sun:06:44-sun:07:14",
                  "Engine": "aurora-mysql",
                  "Endpoint": "rds-aurora-mysql-demo.cluster-c2qmmyq8rslr.us-east-2.rds.amazonaws.com",
                  "EarliestRestorableTime": "2021-12-12T22:03:36.962Z",
                  "CrossAccountClone": false,
                  "IAMDatabaseAuthenticationEnabled": false,
                  "ClusterCreateTime": "2021-12-12T22:02:35.218Z",
                  "EngineVersion": "5.7.mysql_aurora.2.10.1",
                  "DeletionProtection": false,
                  "DBClusterIdentifier": "rds-aurora-mysql-demo",
                  "DbClusterResourceId": "cluster-SRLISEITRIAVBYKCEUQD5B2XK4",
                  "DBClusterMembers": [
                      {
                          "IsClusterWriter": true,
                          "DBClusterParameterGroupStatus": "in-sync",
                          "PromotionTier": 1,
                          "DBInstanceIdentifier": "rds-aurora-mysql-demo-0"
                      }
                  ],
                  "DBClusterArn": "arn:aws:rds:us-east-2:999999999999:cluster:rds-aurora-mysql-demo",
                  "KmsKeyId": "arn:aws:kms:us-east-2:999999999999:key/707a692c-3fbc-4861-bb90-cc0342a76287",
                  "StorageEncrypted": true,
                  "AssociatedRoles": [],
                  "DBClusterParameterGroup": "default.aurora-mysql5.7",
                  "AvailabilityZones": [
                      "us-east-2b",
                      "us-east-2c",
                      "us-east-2a"
                  ],
                  "Port": 3306
              }
          ]
      }

## rds describe-instances
    {
        "DBInstances": [
            {
                "PubliclyAccessible": false,
                "MasterUsername": "dba",
                "MonitoringInterval": 0,
                "LicenseModel": "general-public-license",
                "VpcSecurityGroups": [
                    {
                        "Status": "active",
                        "VpcSecurityGroupId": "sg-78b81711"
                    }
                ],
                "InstanceCreateTime": "2021-12-12T22:09:56.015Z",
                "CopyTagsToSnapshot": false,
                "OptionGroupMemberships": [
                    {
                        "Status": "in-sync",
                        "OptionGroupName": "default:aurora-mysql-5-7"
                    }
                ],
                "PendingModifiedValues": {},
                "Engine": "aurora-mysql",
                "MultiAZ": false,
                "DBSecurityGroups": [],
                "DBParameterGroups": [
                    {
                        "DBParameterGroupName": "default.aurora-mysql5.7",
                        "ParameterApplyStatus": "in-sync"
                    }
                ],
                "PerformanceInsightsEnabled": false,
                "AutoMinorVersionUpgrade": true,
                "PreferredBackupWindow": "05:09-05:39",
                "PromotionTier": 1,
                "DBSubnetGroup": {
                    "Subnets": [
                        {
                            "SubnetStatus": "Active",
                            "SubnetIdentifier": "subnet-f3b5139a",
                            "SubnetOutpost": {},
                            "SubnetAvailabilityZone": {
                                "Name": "us-east-2a"
                            }
                        },
                        {
                            "SubnetStatus": "Active",
                            "SubnetIdentifier": "subnet-0bb54170",
                            "SubnetOutpost": {},
                            "SubnetAvailabilityZone": {
                                "Name": "us-east-2b"
                            }
                        },
                        {
                            "SubnetStatus": "Active",
                            "SubnetIdentifier": "subnet-7aa0a830",
                            "SubnetOutpost": {},
                            "SubnetAvailabilityZone": {
                                "Name": "us-east-2c"
                            }
                        }
                    ],
                    "DBSubnetGroupName": "rds-aurora-mysql-demo-sng",
                    "VpcId": "vpc-c860b5a1",
                    "DBSubnetGroupDescription": "Subnets for rds-aurora-mysql-demo",
                    "SubnetGroupStatus": "Complete"
                },
                "ReadReplicaDBInstanceIdentifiers": [],
                "AllocatedStorage": 1,
                "DBInstanceArn": "arn:aws:rds:us-east-2:999999999999:db:rds-aurora-mysql-demo-0",
                "BackupRetentionPeriod": 1,
                "PreferredMaintenanceWindow": "fri:06:34-fri:07:04",
                "Endpoint": {
                    "HostedZoneId": "Z2XHWR1WZ565X2",
                    "Port": 3306,
                    "Address": "rds-aurora-mysql-demo-0.c2qmmyq8rslr.us-east-2.rds.amazonaws.com"
                },
                "DBInstanceStatus": "available",
                "IAMDatabaseAuthenticationEnabled": false,
                "EngineVersion": "5.7.mysql_aurora.2.10.1",
                "DeletionProtection": false,
                "AvailabilityZone": "us-east-2b",
                "DomainMemberships": [],
                "DBClusterIdentifier": "rds-aurora-mysql-demo",
                "StorageType": "aurora",
                "DbiResourceId": "db-NNDATD2DFOLMGODAEZLTHWRZ4Y",
                "CACertificateIdentifier": "rds-ca-2019",
                "KmsKeyId": "arn:aws:kms:us-east-2:999999999999:key/707a692c-3fbc-4861-bb90-cc0342a76287",
                "StorageEncrypted": true,
                "AssociatedRoles": [],
                "DBInstanceClass": "db.t2.small",
                "DbInstancePort": 0,
                "DBInstanceIdentifier": "rds-aurora-mysql-demo-0"
            }
        ]
    }

## rds describe-db-subnet-groups

    {
        "DBSubnetGroups": [
            {
                "DBSubnetGroupName": "rds-aurora-mysql-demo-sng",
                "DBSubnetGroupDescription": "Subnets for rds-aurora-mysql-demo",
                "VpcId": "vpc-c860b5a1",
                "SubnetGroupStatus": "Complete",
                "Subnets": [
                    {
                        "SubnetIdentifier": "subnet-f3b5139a",
                        "SubnetAvailabilityZone": {
                            "Name": "us-east-2a"
                        },
                        "SubnetOutpost": {},
                        "SubnetStatus": "Active"
                    },
                    {
                        "SubnetIdentifier": "subnet-0bb54170",
                        "SubnetAvailabilityZone": {
                            "Name": "us-east-2b"
                        },
                        "SubnetOutpost": {},
                        "SubnetStatus": "Active"
                    },
                    {
                        "SubnetIdentifier": "subnet-7aa0a830",
                        "SubnetAvailabilityZone": {
                            "Name": "us-east-2c"
                        },
                        "SubnetOutpost": {},
                        "SubnetStatus": "Active"
                    }
                ],
                "DBSubnetGroupArn": "arn:aws:rds:us-east-2:999999999999:subgrp:rds-aurora-mysql-demo-sng"
            }
        ]
    }

##  aws rds describe-orderable-db-instance-options

    "db.t2.medium",
    "db.t2.small",
    "db.t3.large",
    "db.t3.medium",
    "db.t3.small",
    "db.t4g.large",
    "db.t4g.medium",

## kms describe-key

      aws kms describe-key --key-id  alias/aws/rds
      {
          "KeyMetadata": {
              "AWSAccountId": "999999999999",
              "KeyId": "707a692c-3fbc-4861-bb90-cc0342a76287",
              "Arn": "arn:aws:kms:us-east-2:999999999999:key/707a692c-3fbc-4861-bb90-cc0342a76287",
              "CreationDate": "2021-12-11T17:52:26.933000-05:00",
              "Enabled": true,
              "Description": "Default key that protects my RDS database volumes when no other key is defined",
              "KeyUsage": "ENCRYPT_DECRYPT",
              "KeyState": "Enabled",
              "Origin": "AWS_KMS",
              "KeyManager": "AWS",
              "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
              "EncryptionAlgorithms": [
                  "SYMMETRIC_DEFAULT"
              ]
          }
      }

# Troublshooting

The most common issue is the inability to connect to the cluster with the mysql client.  If `nc` fails, verify that the cluster has the correct security group and that security group has ingress from your current EC2 instances IP address.

    CLUSTER_SG_ID=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].VpcSecurityGroups[].VpcSecurityGroupId' --output text)
    echo ${CLUSTER_SG_ID}
    aws ec2 describe-security-groups --group-id ${CLUSTER_SG_ID} --query '*[].IpPermissions'
    ifconfig eth0 | grep "inet " | awk '{print $2}'

# Teardown

RDS Cluster resources incur an operating cost, An RDS Aurora cluster instance is not covered by the free tier. It is always wise to remove resources when only used for testing purposes.

## Validate existing resources
    aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID}
    aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID}
    aws ec2 describe-security-groups --filters Name=group-name,Values=${SG_NAME}
    aws rds describe-db-subnet-groups  --db-subnet-group-name ${SUBNET_GROUP}


## Remove Instance
    aws rds delete-db-instance --db-instance-identifier ${INSTANCE_ID}
    time aws rds wait db-instance-deleted --db-instance-identifier ${INSTANCE_ID}

## Remove Cluster
    # While not advisable to skip the final snapshot when deleting a cluster, this has an incurred cost and for testing purposes we do not need this
    aws rds delete-db-cluster --db-cluster-identifier ${CLUSTER_ID} --skip-final-snapshot

    # This is no `aws rds wait` option for removing a cluster
    while : ; do
      STATUS=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].Status' --output text; exit $?)
      [ $? -ne 0 ] && break
      echo $(date) ${STATUS}
      sleep 3
    done

## Remove unused related resources
    aws rds delete-db-subnet-group --db-subnet-group-name ${SUBNET_GROUP}

For the purposes of this example we do not remove the security group. This also does not incur an ongoing cost.

# References

## awscli

- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-engine-versions.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-subnet-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-subnet-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-cluster-parameter-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-parameter-group.html

- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-cluster.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-instance.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-clusters.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-instances.html

- https://docs.aws.amazon.com/cli/latest/reference/rds/wait/index.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-orderable-db-instance-options.html


## Remove RDS Aurora Cluster resources
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/delete-db-instance.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/delete-db-cluster.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/delete-db-subnet-group.html

## KMS
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/list-keys.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/list-aliases.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/describe-key.html


## User Guide
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.DBInstanceClass.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances.html
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.CreateVPC.html

## Iaas
- https://the.error.expert/amazon-web-services/awscli/rds/
