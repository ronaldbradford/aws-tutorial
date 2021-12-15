# Aurora MySQL minor version upgrade

Aurora provides an ability to automatically perform minor version upgrades within the weekly maintenance period defined for your cluster.
For a non-critical system this may be a practical approach to keep up to date with any new releases or security fixes. For larger and more complex organizations that perform testing and validation of all versions, you may institute a manual process for your environment train.

Using the <a href="create-mysql-aurora-cluster.md">Create an RDS Aurora MySQL Cluster</a> tutorial, the cluster was intentionally created one point release behind (2.10.0) the current (2.10.1) at the time this tutorial was created, so that an upgrade example could be demonstrated.

You should always familiarize yourself with details of any release via the <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2101.html">Release Notes Version 2.10.1</a>. In particular if you have experienced an issue with a current release and are awaiting this correction. Some release also include key improvements that may change your process of monitoring and validation.  In a subsequent tutorial we will discuss the improved <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.html#AuroraMySQL.Replication.Availability">Zero-downtime patching (ZDP)</a> in 2.10.0. In this example we see improved messaging in the events for this feature.

Verify your current version

    aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].EngineVersion' --output text
    5.7.mysql_aurora.2.10.0

Perform an immediate upgrade

    aws rds modify-db-cluster --db-cluster-identifier ${CLUSTER_ID} --engine-version 5.7.mysql_aurora.2.10.1 --apply-immediately

Behind the scenes a number of different activities happen for your Aurora Cluster and it's one or more Instances. All clusters instances must be at the same version (they can however be at different instance types).  In a later tutorial I will demonstrate the lifecycle of the status of the cluster, the loss of connectivity in instances during their reboot.  In the most simpliest example, the status will move from upgrading to available.  

As with several other tutorials, an interactive display of state change (e.g. status) can be helpful in calculating and recording times for your environments and providing feedback to operators. You can see in this example the entire time of the process took approximately 8 minutes, while the version change of the cluster is a subset of this time, and the loss of any connectivity a much smaller subset of seconds.

    # Interactively monitor status
    Tue Dec 14 20:57:45 UTC 2021 upgrading
    ...
    Tue Dec 14 21:04:19 UTC 2021 upgrading
    Tue Dec 14 21:04:23 UTC 2021 available

Verify your new version

    aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].EngineVersion' --output text
    5.7.mysql_aurora.2.10.1


    docker run -it --rm mysql mysql -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"
    mysql: [Warning] Using a password on the command line interface can be insecure.
    +-------------------------+------------------+-----------+-------------------+--------------------+
    | @@aurora_server_id      | @@aurora_version | VERSION() | USER()            | @@innodb_read_only |
    +-------------------------+------------------+-----------+-------------------+--------------------+
    | rds-aurora-mysql-demo-0 | 2.10.1           | 5.7.12    | dba@172.31.39.139 |                  0 |
    +-------------------------+------------------+-----------+-------------------+--------------------+


You will also have a number of <a href="mysql-aurora-events.md">Events</a> during the process. Events are only kept for 7 days so it is valuable to record this information within you ticket tracking records.

# WARNING
You cannot downgrade an Aurora Cluster minor version. Any attempt to will result in an error similar to:

    aws rds modify-db-cluster --db-cluster-identifier ${CLUSTER_ID} --engine-version 5.7.mysql_aurora.2.09.2 --apply-immediately
    An error occurred (InvalidParameterCombination) when calling the ModifyDBCluster operation: Cannot upgrade aurora-mysql from 5.7.mysql_aurora.2.10.0 to 5.7.mysql_aurora.2.09.2


# Example Output

## rds decribe-events
    aws rds describe-events --source-type db-cluster --source-identifier ${CLUSTER_ID}
    {
        "Events": [
            {
                "EventCategories": [
                    "maintenance"
                ],
                "SourceType": "db-cluster",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:cluster:rds-aurora-mysql-demo",
                "Date": "2021-12-14T21:04:22.856Z",
                "Message": "Database cluster has been patched",
                "SourceIdentifier": "rds-aurora-mysql-demo"
            }
        ]
    }


    aws rds describe-events --source-type db-instance --source-identifier ${INSTANCE_ID}
    {
        "Events": [
            {
                "EventCategories": [],
                "SourceType": "db-instance",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:db:rds-aurora-mysql-demo-0",
                "Date": "2021-12-14T21:01:24.716Z",
                "Message": "Attempting to upgrade the database with zero downtime.",
                "SourceIdentifier": "rds-aurora-mysql-demo-0"
            },
            {
                "EventCategories": [],
                "SourceType": "db-instance",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:db:rds-aurora-mysql-demo-0",
                "Date": "2021-12-14T21:01:24.717Z",
                "Message": "Attempt to upgrade the database with zero downtime finished. The process took 0 ms, 4 connections preserved, 0 connections dropped. See the database error log for details.",
                "SourceIdentifier": "rds-aurora-mysql-demo-0"
            }
        ]
    }


# References
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/modify-db-cluster.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-events.html
