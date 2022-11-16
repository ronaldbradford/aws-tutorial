# Create an Aurora Manual Snapshot

By default <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Backups.html">Aurora continiously backs up</a> your cluster.  In addition, Aurora will perform a daily automated snapshot retaining these for the specified backup retention period of your cluster (1 to 35 days, defaults to 1).

Finally, when you delete a cluster, you have the option to optionally specify a final snapshot.

## Required parameters

    AWS_PROFILE=rdsdemo
    ENGINE="aurora-mysql" # or aurora-postgresql
    CLUSTER_ID="rds-${ENGINE}-demo"
    SNAPSHOT_ID="first-snapshot"

## Perform a manual snapshot


    aws rds create-db-cluster-snapshot --db-cluster-identifier ${CLUSTER_ID} --db-cluster-snapshot-identifier ${SNAPSHOT_ID}
    aws rds wait db-cluster-snapshot-available --db-cluster-snapshot-identifier ${SNAPSHOT_ID}
    aws rds describe-db-cluster-snapshots --db-cluster-snapshot-identifier ${SNAPSHOT_ID}

    # Alternative interactive feedback to `wait`
    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-cluster-snapshots --db-cluster-snapshot-identifier ${SNAPSHOT_ID} --query '*[].Status' --output text; exit $?)
      [ $? -ne 0 ] && break
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done


# Example Output (manual snapshot)
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

## rds describe-events

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

# Example Output (automated snapshot)
## aws rds describe-db-cluster-snapshots

Note the primary difference is `SnapshotType`.

    {
        "DBClusterSnapshots": [
            {
                "Engine": "aurora-postgresql",
                "SnapshotCreateTime": "2022-11-16T05:41:29.497Z",
                "VpcId": "vpc-0a03225585506c531",
                "DBClusterIdentifier": "rds-aurora-postgresql-demo",
                "DBClusterSnapshotArn": "arn:aws:rds:us-east-2:356050288086:cluster-snapshot:rds:rds-aurora-postgresql-demo-2022-11-16-05-41",
                "MasterUsername": "dba",
                "LicenseModel": "postgresql-license",
                "Status": "available",
                "PercentProgress": 100,
                "DBClusterSnapshotIdentifier": "rds:rds-aurora-postgresql-demo-2022-11-16-05-41",
                "KmsKeyId": "arn:aws:kms:us-east-2:356050288086:key/bb8ec82d-9d37-40db-adbc-3dad3a7d7214",
                "ClusterCreateTime": "2022-11-15T20:00:02.999Z",
                "StorageEncrypted": true,
                "AllocatedStorage": 0,
                "EngineVersion": "14.5",
                "SnapshotType": "automated",
                "AvailabilityZones": [
                    "us-east-2a",
                    "us-east-2b",
                    "us-east-2c"
                ],
                "IAMDatabaseAuthenticationEnabled": false,
                "Port": 0
            }
        ]
    }

# Teardown

RDS Snaphots have a cost for the storage of your data. It is unclear if the AWS Free tier offers any snapshots at no cost. For these example tutorials it is a good practice to cleanup your tested snapshot.

    aws rds delete-db-cluster-snapshot --db-cluster-snapshot-identifier ${SNAPSHOT_ID}

# Errors

 An error occurred (SnapshotQuotaExceeded) when calling operation: Cannot create more than 100 manual snapshots

Errors never happen when you would like them to. For example, when trying to <a hef="https://the.error.expert/amazon-web-services/awscli/rds/delete-db-cluster/rds-snapquotaexceeded-deletedbcluster-manual-snapshots.html">Delete a Cluster</a>.

# References

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
