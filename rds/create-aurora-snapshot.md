# Create an Aurora Snapshot

    SNAPSHOT_ID="first-snapshot"
    aws rds create-db-cluster-snapshot --db-cluster-identifier ${CLUSTER_ID} --db-cluster-snapshot-identifier ${SNAPSHOT_ID}
    aws rds wait db-cluster-snapshot-available --db-cluster-snapshot-identifier ${SNAPSHOT_ID}

    # Alternative interactive feedback
    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-cluster-snapshots --db-cluster-snapshot-identifier ${SNAPSHOT_ID} --query '*[].Status' --output text; exit $?)
      [ $? -ne 0 ] && break
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done

    aws rds describe-db-cluster-snapshots --db-cluster-snapshot-identifier ${SNAPSHOT_ID}

# Example Output

## rds create-db-cluster-snapshot

    {
        "DBClusterSnapshot": {
            "AvailabilityZones": [
                "us-east-2a",
                "us-east-2b",
                "us-east-2c"
            ],
            "DBClusterSnapshotIdentifier": "first-snapshot",
            "DBClusterIdentifier": "rds-aurora-mysql-demo",
            "SnapshotCreateTime": "2021-12-17T22:08:31.699000+00:00",
            "Engine": "aurora-mysql",
            "EngineMode": "provisioned",
            "AllocatedStorage": 1,
            "Status": "creating",
            "Port": 0,
            "VpcId": "vpc-c860b5a1",
            "ClusterCreateTime": "2021-12-14T20:12:34.867000+00:00",
            "MasterUsername": "dba",
            "EngineVersion": "5.7.mysql_aurora.2.10.1",
            "LicenseModel": "aurora-mysql",
            "SnapshotType": "manual",
            "PercentProgress": 0,
            "StorageEncrypted": true,
            "KmsKeyId": "arn:aws:kms:us-east-2:999999999999:key/707a692c-3fbc-4861-bb90-cc0342a76287",
            "DBClusterSnapshotArn": "arn:aws:rds:us-east-2:999999999999:cluster-snapshot:first-snapshot",
            "IAMDatabaseAuthenticationEnabled": false,
            "TagList": []
        }

## rds describe-db-cluster-snapshot

    {
        "DBClusterSnapshot": {
            "AvailabilityZones": [
                "us-east-2a",
                "us-east-2b",
                "us-east-2c"
            ],
            "DBClusterSnapshotIdentifier": "first-snapshot",
            "DBClusterIdentifier": "rds-aurora-mysql-demo",
            "SnapshotCreateTime": "2021-12-17T22:08:31.699000+00:00",
            "Engine": "aurora-mysql",
            "EngineMode": "provisioned",
            "AllocatedStorage": 0,
            "Status": "available",
            "Port": 0,
            "VpcId": "vpc-c860b5a1",
            "ClusterCreateTime": "2021-12-14T20:12:34.867000+00:00",
            "MasterUsername": "dba",
            "EngineVersion": "5.7.mysql_aurora.2.10.1",
            "LicenseModel": "aurora-mysql",
            "SnapshotType": "manual",
            "PercentProgress": 100,
            "StorageEncrypted": true,
            "KmsKeyId": "arn:aws:kms:us-east-2:414340713341:key/707a692c-3fbc-4861-bb90-cc0342a76287",
            "DBClusterSnapshotArn": "arn:aws:rds:us-east-2:414340713341:cluster-snapshot:first-snapshot",
            "IAMDatabaseAuthenticationEnabled": false,
            "TagList": []
        }
    }

# rds describe-events

    $ aws rds describe-events

    {
        "SourceIdentifier": "first-snapshot",
        "SourceType": "db-cluster-snapshot",
        "Message": "Creating manual cluster snapshot",
        "EventCategories": [
            "backup"
        ],
        "Date": "2021-12-17T22:09:00.071000+00:00",
        "SourceArn": "arn:aws:rds:us-east-2:414340713341:cluster-snapshot:first-snapshot"
    },
    {
        "SourceIdentifier": "first-snapshot",
        "SourceType": "db-cluster-snapshot",
        "Message": "Manual cluster snapshot created",
        "EventCategories": [
            "backup"
        ],
        "Date": "2021-12-17T22:12:33.695000+00:00",
        "SourceArn": "arn:aws:rds:us-east-2:414340713341:cluster-snapshot:first-snapshot"
    },
    {
        "SourceIdentifier": "first-snapshot",
        "SourceType": "db-cluster-snapshot",
        "Message": "Deleted manual snapshot",
        "EventCategories": [
            "deletion"
        ],
        "Date": "2021-12-17T22:19:35.957000+00:00",
        "SourceArn": "arn:aws:rds:us-east-2:414340713341:cluster-snapshot:first-snapshot"
    }


# Teardown

RDS Snaphots have a cost for storage. It is unclear if the AWS Free tier offers any snapshots at no cost.

    aws rds delete-db-cluster-snapshot --db-cluster-snapshot-identifier ${SNAPSHOT_ID}


#References

## awscli
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/create-db-cluster-snapshot.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-cluster-snapshots.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/delete-db-cluster-snapshot.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/wait/index.html

## User Guide
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_CreateSnapshotCluster.html

## Errors
- https://the.error.expert/amazon-web-services/awscli/rds/create-db-cluster-snapshot/
- https://the.error.expert/amazon-web-services/awscli/rds/delete-db-cluster-snapshot/
- https://the.error.expert/amazon-web-services/awscli/rds/describe-db-cluster-snapshots/
