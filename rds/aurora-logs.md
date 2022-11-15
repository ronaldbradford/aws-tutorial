# How to review your RDS Aurora Cluster logs


There is no ability to look at log files for the cluster. They are only available per cluster instance.

    # List Log Files
    aws rds describe-db-log-files --db-instance-identifier ${INSTANCE_ID}

    # Note: if you do no specify --output text you will get the log in a JSON format
    ERROR_LOG="error/mysql-error-running.log" # For MySQL variant
    ERROR_LOG="error/postgres.log" # PostgreSQL
    aws rds download-db-log-file-portion --db-instance-identifier ${INSTANCE_ID} --log-file-name ${ERROR_LOG} --output text


# Example Output

## rds describe-db-log-files (MySQL)

    $ aws rds describe-db-log-files --db-instance-identifier ${INSTANCE_ID}
    {
        "DescribeDBLogFiles": [
            {
                "LastWritten": 1639516200000,
                "LogFileName": "error/mysql-error-running.log",
                "Size": 105234
            },
            {
                "LastWritten": 1639513801000,
                "LogFileName": "error/mysql-error-running.log.2021-12-14.20",
                "Size": 1407077
            },
            {
                "LastWritten": 1639514400000,
                "LogFileName": "error/mysql-error-running.log.2021-12-14.21",
                "Size": 13452
            },
            {
                "LastWritten": 1639517082000,
                "LogFileName": "error/mysql-error.log",
                "Size": 19071
            },
            {
                "LastWritten": 1639516200000,
                "LogFileName": "external/mysql-external.log",
                "Size": 1633
            }
        ]
    }

## rds describe-db-log-files (PostgreSQL)

    {
        "DescribeDBLogFiles": [
            {
                "LastWritten": 1668544339598,
                "LogFileName": "error/postgres.log",
                "Size": 1635
            },
            {
                "LastWritten": 1668544339602,
                "LogFileName": "error/postgresql.log.2022-11-15-2024",
                "Size": 8756
            },
            {
                "LastWritten": 1668548605075,
                "LogFileName": "error/postgresql.log.2022-11-15-2100",
                "Size": 1958
            },
            {
                "LastWritten": 1668549600175,
                "LogFileName": "error/postgresql.log.2022-11-15-2200",
                "Size": 0
            }
        ]
    }


# References
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/describe-db-log-files.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/rds/download-db-log-file-portion.html
