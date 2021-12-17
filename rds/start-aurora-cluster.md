# Starting a stopped Aurora Cluster

This example will show you how to start a stopped cluster.  In this state, the cluster and all instances are stopped.
Aurora will automatically start a cluster that has been stopped for 7 days

## Required configurable parameters

    CLUSTER_ID="rds-aurora-mysql-demo"
    INSTANCE_ID="${CLUSTER_ID}-0"

## Verify cluster and instance are stopped

    aws rds describe-db-clusters --db-cluster-id ${CLUSTER_ID} --query '*[].Status' --output text
    aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].DBInstanceStatus'  --output text

## Check Cluster Instances Example Output

    aws rds describe-db-clusters --db-cluster-id ${CLUSTER_ID} --query '*[].DBClusterMembers'
    [
        [
            {
                "DBInstanceIdentifier": "rds-aurora-mysql-demo-0",
                "IsClusterWriter": true,
                "DBClusterParameterGroupStatus": "in-sync",
                "PromotionTier": 1
            }
        ]
    ]


## Start the cluster

The cluster and all instances are started with this command. You cannot start individual instances in an Aurora Cluster.

    aws rds start-db-cluster --db-cluster-identifier ${CLUSTER_ID}

    EXPECTED_STATUS="available"
    while : ; do
      STATUS=$(aws rds describe-db-clusters --db-cluster-identifier ${CLUSTER_ID} --query '*[].Status' --output text)
      echo $(date) ${STATUS}
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      sleep 3
    done

    aws rds describe-db-clusters --db-cluster-id ${CLUSTER_ID} --query '*[].Status' --output text                     
    aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].DBInstanceStatus' --output text

# Example output

    Fri Dec 17 16:51:53 EST 2021 starting
    ...
    Fri Dec 17 17:01:06 EST 2021 starting
    Fri Dec 17 17:01:10 EST 2021 starting
    Fri Dec 17 17:01:13 EST 2021 available

    $ % aws rds describe-db-clusters --db-cluster-id ${CLUSTER_ID} --query '*[].Status' --output
    available

    $ aws rds describe-db-instances --db-instance-identifier ${INSTANCE_ID} --query '*[].DBInstanceStatus' --output text
    available

## rds describe-events

    {
        "Events": [
            {
                "SourceIdentifier": "rds-aurora-mysql-demo-0",
                "SourceType": "db-instance",
                "Message": "Recovery of the DB instance has started. Recovery time will vary with the amount of data to be recovered.",
                "EventCategories": [
                    "recovery"
                ],
                "Date": "2021-12-17T21:52:30.235000+00:00",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:db:rds-aurora-mysql-demo-0"
            },
            {
                "SourceIdentifier": "rds-aurora-mysql-demo-0",
                "SourceType": "db-instance",
                "Message": "Recovery of the DB instance has started. Recovery time will vary with the amount of data to be recovered.",
                "EventCategories": [
                    "recovery"
                ],
                "Date": "2021-12-17T21:53:00.451000+00:00",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:db:rds-aurora-mysql-demo-0"
            },
            {
                "SourceIdentifier": "rds-aurora-mysql-demo-0",
                "SourceType": "db-instance",
                "Message": "DB instance restarted",
                "EventCategories": [
                    "availability"
                ],
                "Date": "2021-12-17T21:57:07.052000+00:00",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:db:rds-aurora-mysql-demo-0"
            },
            {
                "SourceIdentifier": "rds-aurora-mysql-demo-0",
                "SourceType": "db-instance",
                "Message": "Recovery of the DB instance is complete.",
                "EventCategories": [
                    "recovery"
                ],
                "Date": "2021-12-17T21:57:30.515000+00:00",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:db:rds-aurora-mysql-demo-0"
            },
            {
                "SourceIdentifier": "rds-aurora-mysql-demo-0",
                "SourceType": "db-instance",
                "Message": "DB instance started",
                "EventCategories": [
                    "notification"
                ],
                "Date": "2021-12-17T21:58:24.663000+00:00",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:db:rds-aurora-mysql-demo-0"
            },
            {
                "SourceIdentifier": "rds-aurora-mysql-demo",
                "SourceType": "db-cluster",
                "Message": "DB cluster started",
                "EventCategories": [
                    "notification"
                ],
                "Date": "2021-12-17T22:01:12.355000+00:00",
                "SourceArn": "arn:aws:rds:us-east-2:414340713341:cluster:rds-aurora-mysql-demo"
            }
        }    

# References
## awscli
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/start-db-cluster.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-events.html

## User Guide
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-cluster-stop-start.html
