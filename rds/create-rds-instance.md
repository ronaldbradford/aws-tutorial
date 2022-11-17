# Create an RDS Database Instance

This tutorial will demonstrate a way to create a simple RDS instance. A subsequent tutorial will provide details for more robust instacne setup.

## RDS Instance Requirements

An RDS instance has a number of required AWS resources in order to be successfully created and accessed. Some of these resources require additional IAM policy permissions in addition to the AWS Managed policy by `AmazonRDSFullAccess` that is demonstrated in <a href="../iam/create-iam-user.md">this tutorial</a>.

<ul>
<li>A VPC that has correctly configured subnets. <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html">Get started with Amazon VPC</a> provides an example setup.</li>
<li>An IAM user with applicable privileges to create RDS resources. See the tutorial <a href="../iam/create-iam-user.md">Create a new IAM User</a></li>
<li>An EC2 security group that enables RDS instance ingress (MySQL/MariaDB - 3306, PostgreSQL - 5432, SQL Server - 1433, Oracle - ???? ) within your VPC. See the tutorial <a href="../ec2/create-rds-instance-security-group.md">Create RDS Instance Security Group</a> for how to create this in your AWS account.</li>
<li>A DB subnet group based on the applicable VPC subnets. Described in this tutorial.</li>
<li>An instance parameter group. AWS does provide a default that is used in this tutorial.</li>
<li>A KMS Custom Managed Key (CMK). AWS does provide an RDS default KMS key that is used in this tutorial.</li>
<li>An EC2 instance within the VPC to access the RDS instance. See the tutorial <a href="../ec2/create-an-assessible-instance.md">Create an Internet Accessible EC2 instance in your VPC</a> for the appropriate setup.
</ul>

## Minimal better practices

While this example is designed to create an RDS instance in the simplest way, some additional best practice decisions are made.

- All data should always be encrypted. By default a new RDS instance is not using encrypted storage which is does not match the AWS Well-Architected Framework best practices for security.
- While not required to specify an engine version, the default version chosen is the oldest version for the engine type, not the most recent version. This is also not a best practice for security.

A subsequent tutorial will provide a number of better practices for creating a more robust and extensible RDS instance.

## IAM user validation

It is possible to execute the setup of the RDS instance from your local machine. You will not be able to perform the validation of and use the instance by default. For simplicity all installation is performed on an EC2 compute instance that can complete the tutorial and validate your RDS instance.

    # Connect to EC2 Instance inside of VPC
    ssh ${IP}

    # Validate IAM User
    export AWS_PROFILE="rdsdemo"
    aws sts get-caller-identity
    aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]'
    aws rds describe-db-instances

## RDS Engines

RDS currently supports six different RDBMS products, each with multiple variants and different versions. These include

- SQL Server
 - Enterprise Edition, Standard Edition, Web Edition, Express Edition
 - Versions 12 (2014), 13 (2016), 14 (2017), 15 (2019)
- PostgreSQL (Open Source)
 - Versions 10 (Deprecated),11,12,13,14
- MySQL (Open Source)
 - Version 5.6, 5.7, 8.0
- MariaDB (Open Source)
 - Versions 103, 10.4, 10.5, 10.6
- Oracle
 - Enterprise Edition, Standard Edition 2, CDB
 - Versions 19, 21 (CDB only)
- Aurora MySQL Cluster (See <a href="create -mysql-aurora-cluster.html">Create an RDS Aurora Cluster</a>)
 - Versions 1 (MySQL 5.6), 2 (MySQL 5.7), 3 (MySQL 8.0)
- Aurora PostgresSQL Cluster (See <a href="create -mysql-aurora-cluster.html">Create an RDS Aurora Cluster</a>)
 - Versions 10 (Deprecated), 11, 12, 13, 14  


## Customable configurable parameters
### SQL Server

    # Necessary configuration to create an example RDS instance
    ENGINE="sqlserver-web" # sqlserver-ex, sqlserver-se, sqlserver-ee

    PORT="1433"
    LICENSE_MODEL="license-included"
    EXTRA_OPTIONS="--license-model ${LICENSE_MODEL}"

### PostgreSQL

    # Necessary configuration to create an example RDS instance
    ENGINE="postgres"
    PORT="5432"
    EXTRA_OPTIONS=""

## MySQL
    # Necessary configuration to create an example RDS instance
    ENGINE="mysql"
    PORT="3306"
    EXTRA_OPTIONS=""

## MariaDB
    # Necessary configuration to create an example RDS instance
    ENGINE="mariadb"
    PORT="3306"
    EXTRA_OPTIONS=""

## Oracle
    ENGINE="oracle-ee"  # oracle-se2, oracle-ee-cdb, oracle-se2-cdb
    PORT="???"
    EXTRA_OPTIONS="???"


## Engine Version
    # List available engine versions
    aws rds describe-db-engine-versions --engine ${ENGINE} --query '*[].EngineVersion' --output text

    # Select the most recent version (override as necessary)
    ENGINE_VERSION=$(aws rds describe-db-engine-versions --engine ${ENGINE} --query '*[].EngineVersion' --output text | awk '{print $NF}') # Not required, but needed for Well-Architected Framework

## Required configurable parameters

    INSTANCE_ID="rds-${ENGINE}-demo-0"
    SUBNET_GROUP="${INSTANCE_ID}-sng"
    SG_NAME="rds-instance-sg"  # Created in different tutorial
    INSTANCE_TYPE="db.t3.medium"  # THIS IS NOT part of the AWS Free Tier
    ALLOCATED_STORAGE=20

    KMS_KEY_ID="alias/aws/rds"  # Not required by default, but needed for Well-Architected Framework

    # Obtain the Security Group Id for the security group name
    SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=${SG_NAME} --query '*[].GroupId' --output text)

    DBA_USER="dba"
    DBA_PASSWD=$(date |md5sum - | cut -c1-20)
    echo "${DBA_PASSWD}"

    # This simple example assumes there is one VPC for the region
    VPC_ID=$(aws ec2 describe-vpcs --query '*[0].VpcId' --output text)

    # This simple example assumes you have only one subnet per AZ for this VPC
    # A more advanced --filter would include for example: Name=tag:tier,Values=db
    SUBNET_IDS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=${VPC_ID} | jq '.Subnets[].SubnetId' | tr '\n' ',' | sed -e "s/,\$"//)
    echo ${VPC_ID},${SUBNET_IDS}

## RDS Objects Creation

    # Create an RDS DB Subnet Group
    aws rds create-db-subnet-group \
        --db-subnet-group-name ${SUBNET_GROUP} \
        --db-subnet-group-description "Subnets for ${CLUSTER_ID}" \
        --subnet-ids "[${SUBNET_IDS}]"
    aws rds describe-db-subnet-groups  --db-subnet-group-name ${SUBNET_GROUP}

    # Create the RDS instance
    aws rds create-db-instance \
      --db-instance-identifier ${INSTANCE_ID} \
      --db-instance-class ${INSTANCE_TYPE} \
      --vpc-security-group-ids ${SG_ID} \
      --db-subnet-group-name ${SUBNET_GROUP}  \
      --engine ${ENGINE} \
      --engine-version ${ENGINE_VERSION} \
      --master-username ${DBA_USER} \
      --master-user-password ${DBA_PASSWD} \
      --storage-encrypted \
      --kms-key-id ${KMS_KEY_ID} \
      --allocated-storage ${ALLOCATED_STORAGE} \
      ${EXTRA_OPTIONS}

    time aws rds wait db-instance-available --db-instance-identifier ${INSTANCE_ID}

    # Alternative method to `rds wait` that is similar to above example.
    # This shows an interactive response and gives the different states and instance may have
    # before it is available. For example: creating, backing-up

    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].DBInstanceStatus' --output text)
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done

# Verification of Instance

     # Connectivity test to the instance
     INSTANCE_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].Endpoint.Address' --output text)
     nc -vzw 2 ${INSTANCE_ENDPOINT} ${PORT} || echo "ERROR: Unable to communicate with '${INSTANCE_ID}'"

## Instance Connection Test

    if [[ "${ENGINE}" == "postgres" ]]; then
      # docker run -it --rm postgres psql -h${INSTANCE_ENDPOINT} -U${DBA_USER} -dpostgres
      docker run -it --rm postgres psql postgresql://${DBA_USER}:${DBA_PASSWD}@${INSTANCE_ENDPOINT}:${PORT}/postgres -c " SELECT version(), current_user, current_database();"
    elif [[ "${ENGINE}" ~= "sqlsserver"]; then
      docker run -it --rm mcr.microsoft.com/mssql-tools /opt/mssql-tools/bin/sqlcmd -S ${INSTANCE_ENDPOINT} -U ${DBA_USER} -P ${DBA_PASSWD} -Q "SELECT @@version, SERVERPROPERTY('productversion') AS productversion, SERVERPROPERTY ('productlevel') AS productlevel, SERVERPROPERTY ('edition') AS edition"
    fi

# Teardown

RDS Instance resources incur an operating cost, An RDS instance is not always covered by the free tier. It is always wise to remove resources when only used for testing purposes.

## Validate existing RDS resources
    aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID}
    aws rds describe-db-subnet-groups  --db-subnet-group-name ${SUBNET_GROUP}

## Remove RDS Instance Resources
While not advisable to skip the final snapshot when deleting an instance, this has an incurred cost and for testing purposes we do not need this

    aws rds delete-db-instance --db-instance-identifier ${INSTANCE_ID} --skip-final-snapshot
    time aws rds wait db-instance-deleted --db-instance-identifier ${INSTANCE_ID}
    aws rds delete-db-subnet-group --db-subnet-group-name ${SUBNET_GROUP}


# References

## awscli

- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-engine-versions.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-subnet-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-subnet-group.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-parameter-group.html

- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-instance.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-instances.html

- https://docs.aws.amazon.com/cli/latest/reference/rds/wait/index.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-orderable-db-instance-options.html


## User Guide
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateDBInstance.html

### SQL Server
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_SQLServer.html
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_SQLServer.html#SQLServer.Concepts.General.InstanceClasses
https://aws.amazon.com/rds/sqlserver/features/

## Release Notes
- Oracle https://docs.aws.amazon.com/AmazonRDS/latest/OracleReleaseNotes/Welcome.html
