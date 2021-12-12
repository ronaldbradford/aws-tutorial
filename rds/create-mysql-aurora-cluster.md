# Create an RDS Aurora MySQL Cluster

This example will show the easiest way to create a MySQL RDS cluster. See [xxx] for a more robust setup.

## RDS Cluster Requirements

An RDS cluster has a number of required AWS resources in order to be created. Some resources require additional IAM policy permissions than provided by `AmazonRDSFullAccess`.

<ul>
<li>A VPC that has subnets.</li>
<li>An EC2 security group that enables ingress (MySQL -3306, PostgreSQL - 5432) within your VPC. See the example <a href="../ec2/create-rds-security-group.md">Create RDS Security Group</a>.</li>
<li>A DB Subnet Group based on applicable VPC subnets.</li>
<li>A Cluster parameter group, AWS does provide a default.</li>
<li>An instance parameter group, AWS does provide a default.</li>
<li>A KMS Custom Managed Key (CMK), AWS does provide an RDS default KMS key.</li>
<li>One or more RDS instances to access the cluster.</li>
</ul>



While this example is designed to create an RDS cluster in the simplest way, some best practice decisions are made.

- All data should be encrypted, by default a new RDS Aurora Cluster is not encrypted breaking the AWS Well-Architected Framework best practices for Security
- While not required to specify an engine version, the default chooses the oldest version for the engine type, not the newest version, also not a best practice.

PARAMETER_GROUP_FAMILY="${ENGINE}5.7"


    # Connect to EC2 Instance inside of VPC
    ssh ${IP}


    # Validate IAM User
    export AWS_PROFILE=rdsdemo
    aws sts get-caller-identity
    aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]'
    aws rds describe-db-clusters


    # Necessary configuration to create an example RDS cluster
    ENGINE="aurora-mysql"
    MYSQL_VERSION="5.7.mysql_aurora.2.10.1" # Not required, but needed for Well-Architected Framework
    INSTANCE_TYPE="db.t2.small"  # NOT Part of the AWS Free Tier
    CLUSTER_ID="rds-aurora-mysql-demo"
    INSTANCE_ID="${CLUSTER_ID}-0"

    SUBNET_GROUP="${CLUSTER_ID}-sng"
    SG_NAME="rds-aurora-sg"
    KMS_KEY_ID="alias/aws/rds"  # Not required, but needed for Well-Architected Framework

    MYSQL_USER="dba"
    MYSQL_PASSWD=$(date |md5sum - | cut -c1-20)

    # This simple example assumes one VPC
    VPC_ID=$(aws ec2 describe-vpcs --query '*[0].VpcId' --output text)

    # This simple example assumes you have only one subnet per AZ for this VPC
    # A more advanced --filter would include for example: Name=tag:tier,Values=db
    SUBNET_IDS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=${VPC_ID} | jq '.Subnets[].SubnetId' | tr '\n' ',' | sed -e "s/,\$"//)


    aws rds create-db-subnet-group \
        --db-subnet-group-name ${SUBNET_GROUP} \
        --db-subnet-group-description "Subnets for ${CLUSTER_ID}" \
        --subnet-ids "[${SUBNET_IDS}]"
    aws rds describe-db-subnet-groups  --db-subnet-group-name ${SUBNET_GROUP}


    SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=${SG_NAME} --query '*[].GroupId' --output text)

    # While the default is to create an unencrypted RDS Aurora cluster, this breaks the AWS Well-Architected Framework Security best practices.
    aws rds create-db-cluster \
      --db-cluster-identifier ${CLUSTER_ID} \
      --vpc-security-group-ids ${SG_ID} \
      --db-subnet-group-name ${SUBNET_GROUP}  \
      --engine ${ENGINE} \
      --engine-version ${MYSQL_VERSION} \
      --master-username ${MYSQL_USER} \
      --master-user-password ${MYSQL_PASSWD} \
      --storage-encrypted \
      --kms-key-id ${KMS_KEY_ID}


    # It is not possible to use the `aws rds wait` command for a cluster creation to proceed with subsequent statements
    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].Status' --output text)
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done


    aws rds create-db-instance \
      --db-cluster-identifier ${CLUSTER_ID} \
      --db-instance-identifier ${INSTANCE_ID} \
      --db-instance-class ${INSTANCE_TYPE} \
      --engine ${ENGINE}

    time aws rds wait db-instance-available --db-instance-identifier ${INSTANCE_ID}

    # Alternative method similar to above example
    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].DBInstanceStatus' --output text)
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done

    # While the AWS Free tier provides for a t2.micro instance for RDS, this instance class is not supported for RDS Aurora
    aws rds describe-orderable-db-instance-options --engine ${ENGINE} --engine-version ${MYSQL_VERSION} --query '*[].DBInstanceClass' | grep db.t

# Validation


    CLUSTER_ENDPOINT=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].Endpoint' --output text)
    CLUSTER_READER_ENDPOINT=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].ReaderEndpoint' --output text)
    nc -vz ${CLUSTER_ENDPOINT} 3306
    # This is an insecure means of connecting to mysql, passing the password on the command line
    docker run -it --rm mysql mysql -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"
    docker run -it --rm mysql mysql -h${CLUSTER_READER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"

    INSTANCE_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].Endpoint.Address' --output text)
    docker run -it --rm mysql mysql -h${INSTANCE_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"

# Next Steps

<a href="mysql-aurora-cluster-common-commands.md">MySQL aurora cluster common commands</a> provides a number of common tasks you may use with a cluster.

# Example Output

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
# rds describe-instances
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

# kms describe-key
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

THe most common issue is the inability to connect to the cluster with the mysql client.  If `nc` fails, verify that the cluster has the correct security group and that security group has ingres from your current EC2 instances IP address.

    CLUSTER_SG_ID=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].VpcSecurityGroups[].VpcSecurityGroupId' --output text)
    echo ${CLUSTER_SG_ID}
    aws ec2 describe-security-groups --group-id ${CLUSTER_SG_ID} --query '*[].IpPermissions'
    ifconfig eth0 | grep "inet " | awk '{print $2}'

# Teardown

RDS Cluster resources incur an operating cost, An RDS Aurora cluster instance is not covered by the free tier. It is always wise to remove resources when only used for testing purposes.

    aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID}
    aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID}
    aws ec2 describe-security-groups --filters Name=group-name,Values=${SG_NAME}
    aws rds describe-db-subnet-groups  --db-subnet-group-name ${SUBNET_GROUP}



    aws rds delete-db-instance --db-instance-identifier ${INSTANCE_ID}

    aws rds wait db-instance-deleted --db-instance-identifier ${INSTANCE_ID}

    # While not advisable to skip the final snapshot when deleting a cluster, this has an incurred cost and for testing purposes we do not need this
    aws rds delete-db-cluster --db-cluster-identifier ${CLUSTER_ID} --skip-final-snapshot

    # This is no `aws rds wait` option for removing a cluster
    while : ; do
      STATUS=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].Status' --output text; exit $?)
      [ $? -ne 0 ] && break
      echo $(date) ${STATUS}
      sleep 3
    done

    aws rds delete-db-subnet-group --db-subnet-group-name ${SUBNET_GROUP}


# References

## awscli

- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-subnet-groups.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-subnet-groups.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-cluster-parameter-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-parameter-group.html

- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-cluster.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-instance.html

- https://docs.aws.amazon.com/cli/latest/reference/rds/wait/index.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-orderable-db-instance-options.html


## Remove RDS Aurora Cluster resources
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/delete-db-cluster.html

## KMS
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/list-keys.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/list-aliases.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/describe-key.html


## User Guide
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.DBInstanceClass.html

## Iaas
- https://the.error.expert/amazon-web-services/awscli/rds/
