# How to monitor cluster events

RDS Cluster, Instance and Cluster Snapshot events can be used to identify key changes that have occurred. This can be particularly useful in a failover situation.  


The default window of recent events displayed is rather short. You can view up to 7 days of events.
An easy example is to look at all events since the creation of the resource, in this example the cluster.

    CREATED=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].ClusterCreateTime' --output text)
    aws rds describe-events --source-type db-cluster --source-identifier ${CLUSTER_ID} --start-time ${CREATED}
    {
        "Events": [
            {
                "EventCategories": [
                    "creation"
                ],
                "SourceType": "db-cluster",
                "SourceArn": "arn:aws:rds:us-east-2:999999999999:cluster:rds-aurora-mysql-demo",
                "Date": "2021-12-14T20:13:32.411Z",
                "Message": "DB cluster created",
                "SourceIdentifier": "rds-aurora-mysql-demo"
            }
        }
    }

## Upgrade events

In this example of a minor version upgrade, you can see a cluster and an instance event.

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
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-events.html
