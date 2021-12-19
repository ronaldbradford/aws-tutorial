# Aurora MySQL Major upgrade - Aurora 2.x to Aurora 3.x

With the most recent release of Aurora 3 (MySQL 8.0 compatibility) on <a href="https://aws.amazon.com/blogs/database/amazon-aurora-mysql-3-with-mysql-8-0-compatibility-is-now-generally-available/">18 NOV 2021</a> this major upgrade has been significantly streamlined.  In a subsequent tutorial I will demonstrate the upgrade process for Aurora PostgreSQL.


- Aurora 3.01.x is the publically available version that is compatible with MySQL 8.0.23. See <a href="https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-23.html">MySQL Community 8.0.23 - 2021-01-1 Release Notes</a>.
- Aurora 2.07.x to 2.10.x is the version that is compatible with MySQL 5.7.12. <a href="https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-12.html">See MySQL Community 5.7.12 - 2016-04-11 Release Notes</a>.

In this tutorial we will demonstrate the easiest path for a simple environment that is using <a href="mysql-sample-data.md">mysql.com example data</a>.  NOTE: While any software upgrade process requires detailed planning, evaluation, documented processes and rollback plans, this tutorial is an introduction of the steps to perform a upgrade on a demo cluster.

In this first approach, there are two steps to the upgrade process.

- Create a snapshot of your Aurora 2.x
- Restore the snapshot into an Aurora 3.x cluster

NOTE: This will not be the only steps used in a migration of an operational Aurora Cluster. This is for demonstration puroses only.

# Create a demo Aurora MySQL cluster

See the tutorial <a href="create-mysql-aurora-cluster">Create an RDS Aurora MySQL Cluster</a> to get started. For this upgrade we will use the <a href="mysql-sample-data.md">mysql.com example databases</a> as sample data.

## Required parameters

     # Variables from create tutorial
     CLUSTER_ID="rds-aurora-mysql-demo"
     ENGINE="aurora-mysql"
     SG_NAME="rds-aurora-sg"
     SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=${SG_NAME} --query '*[].GroupId' --output text)

     # New variables for this upgrade
     SNAPSHOT_ID="before-major-upgrade"
     NEW_CLUSTER_ID="${CLUSTER_ID}-upgraded"
     NEW_INSTANCE_ID=$"INSTANCE_ID}-upgraded"
     NEW_INSTANCE_TYPE="db.t4g.medium"
     NEW_MYSQL_VERSION="8.0.mysql_aurora.3.01.0"



# Create a snapshot

As demonstrated in <a href="mysql-aurora-snapshot.md">Creating an Aurora snapshot</a> tutorial.

    aws rds create-db-cluster-snapshot --db-cluster-identifier ${CLUSTER_ID} --db-cluster-snapshot-identifier ${SNAPSHOT_ID}
    aws rds wait db-cluster-snapshot-available --db-cluster-snapshot-identifier ${SNAPSHOT_ID}
    aws rds describe-db-cluster-snapshots --db-cluster-snapshot-identifier ${SNAPSHOT_ID}

```
# Alternative interactive feedback to `wait`
EXPECTED_STATUS="available"
while : ; do
  STATUS=$(aws rds describe-db-cluster-snapshots --db-cluster-snapshot-identifier ${SNAPSHOT_ID} --query '*[].Status' --output text; exit $?)
  [ $? -ne 0 ] && break
  echo $(date) ${STATUS}
  [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
  sleep 3
done
```


# Restore snapshot to a new cluster

In this upgrade approach we will not change the existing cluster, rather create a new cluster with the intended version.

      aws rds restore-db-cluster-from-snapshot \
          --db-cluster-identifier ${NEW_CLUSTER_ID} \
          --snapshot-identifier ${SNAPSHOT_ID} \
          --vpc-security-group-ids ${SG_ID} \    # Important Requirement
          --engine ${ENGINE} \
          --engine-version ${NEW_MYSQL_VERSION}

      # There is no applicable `wait` option to hold until completion.
      EXPECTED_STATUS="available"
      while : ; do
        STATUS=$(aws rds describe-db-clusters --db-cluster-identifier ${NEW_CLUSTER_ID} --query '*[].Status' --output text)
        echo $(date) ${STATUS}
        [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
        sleep 3
      done

## Example logging output

You can see in this example it took approximately 4 minutes to create the cluster.

      Tue Dec 14 21:34:14 UTC 2021 creating
      ...
      Tue Dec 14 21:38:27 UTC 2021 creating
      Tue Dec 14 21:38:31 UTC 2021 available

# Create new instances for upgraded cluster

An Aurora cluster has no use case without having one or more cluster instances. This will create an instance for the new cluster.

    # An RDS Cluster requries one or more instances to be able to access the cluster.
    aws rds create-db-instance \
      --db-cluster-identifier ${NEW_CLUSTER_ID} \
      --db-instance-identifier ${NEW_INSTANCE_ID} \
      --db-instance-class ${NEW_INSTANCE_TYPE} \
      --engine ${ENGINE} \
      --engine-version ${NEW_MYSQL_VERSION}

    time aws rds wait db-instance-available --db-instance-identifier ${NEW_INSTANCE_ID}


## Example Logging Output

    Tue Dec 14 21:50:19 UTC 2021 creating
    ..
    Tue Dec 14 21:58:33 UTC 2021 creating
    Tue Dec 14 21:58:37 UTC 2021 available

## WARNING

The smallest instance type `db.t2.small` used for Aurora MySQL 5.7 Cluster is not valid with an Aurora MySQL 8.0 Cluster.

    An error occurred (InvalidParameterCombination) when calling the CreateDBInstance operation: RDS does not support creating a DB instance with the following combination: DBInstanceClass=db.t2.small, Engine=aurora-mysql, EngineVersion=8.0.mysql_aurora.3.01.0, LicenseModel=general-public-license. For supported combinations of instance class and database engine version, see the documentation.

## Supported bustable instance types for Aurora MySQL 8.0

You can find the list of supported instance classes in your region with:

    aws rds describe-orderable-db-instance-options --engine ${ENGINE} --engine-version ${NEW_MYSQL_VERSION} --query '*[].DBInstanceClass' | grep db.t
        "db.t3.large",
        "db.t3.medium",
        "db.t4g.large",
        "db.t4g.medium",


# Sanity validation of new version following upgrade


    NEW_CLUSTER_ENDPOINT=$(aws rds describe-db-clusters --db-cluster-identifier ${NEW_CLUSTER_ID} --query '*[].Endpoint' --output text)
    nc -vz ${NEW_CLUSTER_ENDPOINT} 3306
    docker run -it --rm mysql mysql -h${NEW_CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} -e "SELECT @@aurora_server_id,  @@aurora_version, VERSION(), USER(), @@innodb_read_only;"


### Example output
    +----------------------------------+------------------+-----------+-------------------+--------------------+
    | @@aurora_server_id               | @@aurora_version | VERSION() | USER()            | @@innodb_read_only |
    +----------------------------------+------------------+-----------+-------------------+--------------------+
    | rds-aurora-mysql-demo-0-upgraded | 3.01.0           | 8.0.23    | dba@172.31.39.139 |                  0 |
    +----------------------------------+------------------+-----------+-------------------+--------------------+

As you can see the Aurora Version is 3.x, and the MySQL version is 8.0.x.

# Simple data evaluation

Having added  <a href="mysql-sample-data.md">mysql.com example databases</a> as sample data to your original cluster, some simple checks confirm the database is accessible.

    mysql> SELECT Name, Population FROM world.city ORDER BY Population DESC LIMIT 10;
    +------------------+------------+
    | Name             | Population |
    +------------------+------------+
    | Mumbai (Bombay)  |   10500000 |
    | Seoul            |    9981619 |
    | S�o Paulo        |    9968485 |
    | Shanghai         |    9696300 |
    | Jakarta          |    9604900 |
    | Karachi          |    9269265 |
    | Istanbul         |    8787958 |
    | Ciudad de M�xico |    8591309 |
    | Moscow           |    8389200 |
    | New York         |    8008278 |
    +------------------+------------+
    10 rows in set (0.03 sec)

    mysql> SELECT District, SUM(Population) FROM world.ity WHERE CountryCode = 'USA' GROUP BY District HAVING SUM(Population) > 3000000;
    +------------+-----------------+
    | District   | SUM(Population) |
    +------------+-----------------+
    | New York   |         8958085 |
    | California |        16716706 |
    | Illinois   |         3737498 |
    | Texas      |         9208281 |
    | Arizona    |         3178903 |
    | Florida    |         3151408 |
    +------------+-----------------+
    6 rows in set (0.01 sec)


# Additional logs

If your upgrade fails, commonly observed when your instance is left with a status of `incompatible-parameters` you will find some detailed information in the additional log file `upgrade-prechecks.log`.

    aws rds describe-db-log-files --db-instance-identifier ${NEW_INSTANCE_ID}

```
    ....
    {
       "DescribeDBLogFiles": [
           ...
           {
               "LastWritten": 1639518825000,
               "LogFileName": "upgrade-prechecks.log",
               "Size": 13145
           }
       ]
    }
```
Even in the simplest of data examples use the <a href="mysql-aurora-sample-data.md">mysql.com sample databases</a>, you can find a number of warning messages that are applicable to show how this log is important.

    aws rds download-db-log-file-portion --db-instance-identifier ${NEW_INSTANCE_ID} --log-file-name upgrade-prechecks.log --output text

```    
    {
        "serverAddress": "/tmp%2Fmysql.sock",
        "serverVersion": "5.7.12",
        "targetVersion": "8.0.23",
        "auroraServerVersion": "2.10.1",
        "auroraTargetVersion": "3.01.0",
        "outfilePath": "/rdsdbdata/tmp/PreChecker.log",
        "checksPerformed": [
            {
                "id": "oldTemporalCheck",
                "title": "Usage of old temporal type",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "reservedKeywordsCheck",
                "title": "Usage of db objects with names conflicting with new reserved keywords",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "utf8mb3Check",
                "title": "Usage of utf8mb3 charset",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "mysqlSchemaCheck",
                "title": "Table names in the mysql schema conflicting with new tables in 8.0",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "nonNativePartitioningCheck",
                "title": "Partitioned tables using engines with non native partitioning",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "foreignKeyLengthCheck",
                "title": "Foreign key constraint names longer than 64 characters",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "maxdbFlagCheck",
                "title": "Usage of obsolete MAXDB sql_mode flag",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "sqlModeFlagCheck",
                "title": "Usage of obsolete sql_mode flags",
                "status": "OK",
                "description": "Notice: The following DB objects have obsolete options persisted for sql_mode, which will be cleared during upgrade to 8.0.",
                "documentationLink": "https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html#mysql-nutshell-removals",
                "detectedProblems": [
                    {
                        "level": "Notice",
                        "dbObject": "sakila.film_in_stock",
                        "description": "PROCEDURE uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.film_not_in_stock",
                        "description": "PROCEDURE uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.get_customer_balance",
                        "description": "FUNCTION uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.inventory_held_by_customer",
                        "description": "FUNCTION uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.inventory_in_stock",
                        "description": "FUNCTION uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.rewards_report",
                        "description": "PROCEDURE uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.customer_create_date",
                        "description": "TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.ins_film",
                        "description": "TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.upd_film",
                        "description": "TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.del_film",
                        "description": "TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.payment_date",
                        "description": "TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    },
                    {
                        "level": "Notice",
                        "dbObject": "sakila.rental_date",
                        "description": "TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode"
                    }
                ]
            },
            {
                "id": "enumSetElementLenghtCheck",
                "title": "ENUM/SET column definitions containing elements longer than 255 characters",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "partitionedTablesInSharedTablespaceCheck",
                "title": "Usage of partitioned tables in shared tablespaces",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "circularDirectoryCheck",
                "title": "Circular directory references in tablespace data file paths",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "removedFunctionsCheck",
                "title": "Usage of removed functions",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "groupByAscCheck",
                "title": "Usage of removed GROUP BY ASC/DESC syntax",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "removedSysLogVars",
                "title": "Removed system variables for error logging to the system log configuration",
                "status": "CONFIGURATION_ERROR",
                "description": "To run this check requires full path to MySQL server configuration file to be specified at 'configPath' key of options dictionary",
                "documentationLink": "https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-13.html#mysqld-8-0-13-logging"
            },
            {
                "id": "removedSysVars",
                "title": "Removed system variables",
                "status": "CONFIGURATION_ERROR",
                "description": "To run this check requires full path to MySQL server configuration file to be specified at 'configPath' key of options dictionary",
                "documentationLink": "https://dev.mysql.com/doc/refman/8.0/en/added-deprecated-removed.html#optvars-removed"
            },
            {
                "id": "sysVarsNewDefaults",
                "title": "System variables with new default values",
                "status": "CONFIGURATION_ERROR",
                "description": "To run this check requires full path to MySQL server configuration file to be specified at 'configPath' key of options dictionary",
                "documentationLink": "https://mysqlserverteam.com/new-defaults-in-mysql-8-0/"
            },
            {
                "id": "zeroDatesCheck",
                "title": "Zero Date, Datetime, and Timestamp values",
                "status": "OK",
                "description": "Warning: By default zero date/datetime/timestamp values are no longer allowed in MySQL, as of 5.7.8 NO_ZERO_IN_DATE and NO_ZERO_DATE are included in SQL_MODE by default. These modes should be used with strict mode as they will be merged with strict mode in a future release. If you do not include these modes in your SQL_MODE setting, you are able to insert date/datetime/timestamp values that contain zeros. It is strongly advised to replace zero values with valid ones, as they may not work correctly in the future.",
                "documentationLink": "https://lefred.be/content/mysql-8-0-and-wrong-dates/",
                "detectedProblems": [
                    {
                        "level": "Warning",
                        "dbObject": "global.sql_mode",
                        "description": "does not contain either NO_ZERO_DATE or NO_ZERO_IN_DATE which allows insertion of zero dates"
                    }
                ]
            },
            {
                "id": "schemaInconsistencyCheck",
                "title": "Schema inconsistencies resulting from file removal or corruption",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "engineMixupCheck",
                "title": "Tables recognized by InnoDB that belong to a different engine",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "getDanglingFulltextIndex",
                "title": "Tables with dangling FULLTEXT index reference",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "getMismatchedMetadata",
                "title": "Column definition mismatch between InnoDB Data Dictionary and actual table definition.",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "getEventsWithNullDefiner",
                "title": "The definer column for mysql.events cannot be null or blank.",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "getRoutinesWithDepreciatedKeywords",
                "title": "Routines with deprecated keywords in definition",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "getValueOfVariablelower_case_table_names",
                "title": "MySQL pre-checks that all database, table, and trigger names are lowercase when the lower_case_table_names parameter is set to 1.",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "checkTableOutput",
                "title": "Issues reported by 'check table x for upgrade' command",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "auroraCheckDDLRecovery",
                "title": "Check for artifacts related to Aurora DDL recovery feature",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "auroraFODUpgradeCheck",
                "title": "Check for artifacts related to Aurora Fast Online DDL feature",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "auroraUpgradeCheckIndexLengthLimit",
                "title": "The tables with redundant/compact row format can't have an index larger than 767 bytes.",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "auroraUpgradeCheckForIncompleteXATransactions",
                "title": "Pre-checks for XA Transactions in prepared state.",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "auroraUpgradeCheckForRollbackSegmentHistoryLength",
                "title": "Checks if the rollback segment history length for the cluster is high",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "auroraUpgradeCheckForUncommittedRowModifications",
                "title": "Checks if there are many uncommitted modifications to rows",
                "status": "OK",
                "detectedProblems": []
            },
            {
                "id": "defaultAuthenticationPlugin",
                "title": "New default authentication plugin considerations",
                "description": "Warning: The new default authentication plugin 'caching_sha2_password' offers more secure password hashing than previously used 'mysql_native_password' (and consequent improved client connection authentication). However, it also has compatibility implications that may affect existing MySQL installations.  If your MySQL installation must serve pre-8.0 clients and you encounter compatibility issues after upgrading, the simplest way to address those issues is to reconfigure the server to revert to the previous default authentication plugin (mysql_native_password). For example, use these lines in the server option file:\n\n[mysqld]\ndefault_authentication_plugin=mysql_native_password\n\nHowever, the setting should be viewed as temporary, not as a long term or permanent solution, because it causes new accounts created with the setting in effect to forego the improved authentication security.\nIf you are using replication please take time to understand how the authentication plugin changes may impact you.",
                "documentationLink": "https://dev.mysql.com/doc/refman/8.0/en/upgrading-from-previous-series.html#upgrade-caching-sha2-password-compatibility-issues\nhttps://dev.mysql.com/doc/refman/8.0/en/upgrading-from-previous-series.html#upgrade-caching-sha2-password-replication"
            }
        ],
        "errorCount": 0,
        "warningCount": 2,
        "noticeCount": 12,
        "Summary": "No fatal errors were found that would prevent an upgrade, but some potential issues were detected. Please ensure that the reported issues are not significant before upgrading."
    }
```

# More detailed upgrade preparation

This tutorial jumped into the minimum required steps to show a suitable workflow for a major upgrade to an Aurora MySQL cluster. It is recommended that you also use these two compatibility checks available with MySQL community version before upgrading.

## mysqlcheck

```
docker run -it --rm mysql mysqlcheck -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} --all-databases --check-upgrade
```

    mysql.aurora_s3_load_history                       OK
    mysql.bin_log_md_table                             OK
    mysql.bin_log_table                                OK
    mysql.columns_priv                                 OK
    mysql.db                                           OK
    mysql.ddl_log_md_table                             OK
    mysql.ddl_log_table                                OK
    mysql.engine_cost                                  OK
    mysql.event                                        OK
    mysql.func                                         OK
    mysql.general_log                                  OK
    mysql.general_log_backup                           OK
    mysql.gtid_executed                                OK
    mysql.help_category                                OK
    mysql.help_keyword                                 OK
    mysql.help_relation                                OK
    mysql.help_topic                                   OK
    mysql.host                                         OK
    mysql.innodb_index_stats                           OK
    mysql.innodb_table_stats                           OK
    mysql.metadata_md_table                            OK
    mysql.metadata_table                               OK
    mysql.ndb_binlog_index                             OK
    mysql.plugin                                       OK
    mysql.proc                                         OK
    mysql.procs_priv                                   OK
    mysql.proxies_priv                                 OK
    mysql.rds_configuration                            OK
    mysql.rds_global_status_history                    OK
    mysql.rds_global_status_history_old                OK
    mysql.rds_history                                  OK
    mysql.rds_replication_status                       OK
    mysql.rds_sysinfo                                  OK
    mysql.relay_log_md_table                           OK
    mysql.relay_log_table                              OK
    mysql.ro_replica_status                            OK
    mysql.server_cost                                  OK
    mysql.servers                                      OK
    mysql.slave_master_info                            OK
    mysql.slave_relay_log_info                         OK
    mysql.slave_worker_info                            OK
    mysql.slow_log                                     OK
    mysql.slow_log_backup                              OK
    mysql.tables_priv                                  OK
    mysql.time_zone                                    OK
    mysql.time_zone_leap_second                        OK
    mysql.time_zone_name                               OK
    mysql.time_zone_transition                         OK
    mysql.time_zone_transition_type                    OK
    mysql.user                                         OK
    sakila.actor                                       OK
    sakila.address                                     OK
    sakila.category                                    OK
    sakila.city                                        OK
    sakila.country                                     OK
    sakila.customer                                    OK
    sakila.film                                        OK
    sakila.film_actor                                  OK
    sakila.film_category                               OK
    sakila.film_text                                   OK
    sakila.inventory                                   OK
    sakila.language                                    OK
    sakila.payment                                     OK
    sakila.rental                                      OK
    sakila.staff                                       OK
    sakila.store                                       OK
    sys.sys_config                                     OK
    world.city                                         OK
    world.country                                      OK
    world.countrylanguage                              OK


## mysqlsh - util.checkForServerUpgrade()

With this more detailed evaluation with the MySQL shell you can see were information found in the prechecks log is being sourced.

```
    # -e "CLUSTER_ENDPOINT=${CLUSTER_ENDPOINT}" "MYSQL_USER=${MYSQL_USER}"  

    docker run --name mysql8 -e "MYSQL_ALLOW_EMPTY_PASSWORD=yes" -d mysql/mysql-server
    docker exec -it mysql8  /bin/bash

    mysqlsh --js -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p -e "util.checkForServerUpgrade()"

    docker stop mysql8 && docker rm mysql8
```

    Cannot set LC_ALL to locale en_US.UTF-8: No such file or directory
    MySQL Shell 8.0.27

    Copyright (c) 2016, 2021, Oracle and/or its affiliates.
    Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
    Other names may be trademarks of their respective owners.

    Type '\help' or '\?' for help; '\quit' to exit.
    WARNING: Using a password on the command line interface can be insecure.
    Creating a session to 'dba@rds-aurora-mysql-demo.cluster-c2qmmyq8rslr.us-east-2.rds.amazonaws.com'
    Fetching schema names for autocompletion... Press ^C to stop.
    Your MySQL connection id is 27
    Server version: 5.7.12 MySQL Community Server (GPL)
    No default schema selected; type \use <schema> to set one.
     MySQL  rds-aurora-mysql-demo.cluster-c2qmmyq8rslr.us-east-2.rds.amazonaws.com:3306 ssl  JS > util.checkForServerUpgrade()
    The MySQL server at
    rds-aurora-mysql-demo.cluster-c2qmmyq8rslr.us-east-2.rds.amazonaws.com:3306,
    version 5.7.12 - MySQL Community Server (GPL), will now be checked for
    compatibility issues for upgrade to MySQL 8.0.27...

    1) Usage of old temporal type
      No issues found

    2) Usage of db objects with names conflicting with new reserved keywords
      No issues found

    3) Usage of utf8mb3 charset
      No issues found

    4) Table names in the mysql schema conflicting with new tables in 8.0
      No issues found

    5) Partitioned tables using engines with non native partitioning
      No issues found

    6) Foreign key constraint names longer than 64 characters
      No issues found

    7) Usage of obsolete MAXDB sql_mode flag
      No issues found

    8) Usage of obsolete sql_mode flags
      Notice: The following DB objects have obsolete options persisted for
        sql_mode, which will be cleared during upgrade to 8.0.
      More information:
        https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html#mysql-nutshell-removals

      sakila.film_in_stock - PROCEDURE uses obsolete NO_AUTO_CREATE_USER sql_mode
      sakila.film_not_in_stock - PROCEDURE uses obsolete NO_AUTO_CREATE_USER
        sql_mode
      sakila.get_customer_balance - FUNCTION uses obsolete NO_AUTO_CREATE_USER
        sql_mode
      sakila.inventory_held_by_customer - FUNCTION uses obsolete
        NO_AUTO_CREATE_USER sql_mode
      sakila.inventory_in_stock - FUNCTION uses obsolete NO_AUTO_CREATE_USER
        sql_mode
      sakila.rewards_report - PROCEDURE uses obsolete NO_AUTO_CREATE_USER sql_mode
      sakila.customer_create_date - TRIGGER uses obsolete NO_AUTO_CREATE_USER
        sql_mode
      sakila.ins_film - TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode
      sakila.upd_film - TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode
      sakila.del_film - TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode
      sakila.payment_date - TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode
      sakila.rental_date - TRIGGER uses obsolete NO_AUTO_CREATE_USER sql_mode

    9) ENUM/SET column definitions containing elements longer than 255 characters
      No issues found

    10) Usage of partitioned tables in shared tablespaces
      No issues found

    11) Circular directory references in tablespace data file paths
      No issues found

    12) Usage of removed functions
      No issues found

    13) Usage of removed GROUP BY ASC/DESC syntax
      No issues found

    14) Removed system variables for error logging to the system log configuration
      To run this check requires full path to MySQL server configuration file to be specified at 'configPath' key of options dictionary
      More information:
        https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-13.html#mysqld-8-0-13-logging

    15) Removed system variables
      To run this check requires full path to MySQL server configuration file to be specified at 'configPath' key of options dictionary
      More information:
        https://dev.mysql.com/doc/refman/8.0/en/added-deprecated-removed.html#optvars-removed

    16) System variables with new default values
      To run this check requires full path to MySQL server configuration file to be specified at 'configPath' key of options dictionary
      More information:
        https://mysqlserverteam.com/new-defaults-in-mysql-8-0/

    17) Zero Date, Datetime, and Timestamp values
      Warning: By default zero date/datetime/timestamp values are no longer allowed
        in MySQL, as of 5.7.8 NO_ZERO_IN_DATE and NO_ZERO_DATE are included in
        SQL_MODE by default. These modes should be used with strict mode as they will
        be merged with strict mode in a future release. If you do not include these
        modes in your SQL_MODE setting, you are able to insert
        date/datetime/timestamp values that contain zeros. It is strongly advised to
        replace zero values with valid ones, as they may not work correctly in the
        future.
      More information:
        https://lefred.be/content/mysql-8-0-and-wrong-dates/

      global.sql_mode - does not contain either NO_ZERO_DATE or NO_ZERO_IN_DATE
        which allows insertion of zero dates

    18) Schema inconsistencies resulting from file removal or corruption
      Error: Following tables show signs that either table datadir directory or frm
        file was removed/corrupted. Please check server logs, examine datadir to
        detect the issue and fix it before upgrade

      mysql.rds_heartbeat2 - present in INFORMATION_SCHEMA's INNODB_SYS_TABLES
        table but missing from TABLES table

    19) Tables recognized by InnoDB that belong to a different engine
      No issues found

    20) Issues reported by 'check table x for upgrade' command
      No issues found

    21) New default authentication plugin considerations
      Warning: The new default authentication plugin 'caching_sha2_password' offers
        more secure password hashing than previously used 'mysql_native_password'
        (and consequent improved client connection authentication). However, it also
        has compatibility implications that may affect existing MySQL installations.
        If your MySQL installation must serve pre-8.0 clients and you encounter
        compatibility issues after upgrading, the simplest way to address those
        issues is to reconfigure the server to revert to the previous default
        authentication plugin (mysql_native_password). For example, use these lines
        in the server option file:

        [mysqld]
        default_authentication_plugin=mysql_native_password

        However, the setting should be viewed as temporary, not as a long term or
        permanent solution, because it causes new accounts created with the setting
        in effect to forego the improved authentication security.
        If you are using replication please take time to understand how the
        authentication plugin changes may impact you.
      More information:
        https://dev.mysql.com/doc/refman/8.0/en/upgrading-from-previous-series.html#upgrade-caching-sha2-password-compatibility-issues
        https://dev.mysql.com/doc/refman/8.0/en/upgrading-from-previous-series.html#upgrade-caching-sha2-password-replication

    Errors:   1
    Warnings: 2
    Notices:  12

    1 errors were found. Please correct these issues before upgrading to avoid compatibility issues.

## Benchmarking

As with any upgrade, you should have a known set of test cases that perform query analysis, timing and benchmarking to ensure any new version does not introduce a regression.  This can be particularly relevant when SQL hints are used extensively to hone the query optimizer into preferred indexex in the Query Execution Plans (QEP).

# References
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL80.html
- https://dev.mysql.com/doc/refman/8.0/en/upgrading.html
