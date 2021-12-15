# MySQL sample data sets for Aurora


# mysql.com
MySQL provides a number of smaller <a href="https://dev.mysql.com/doc/index-other.html">sample data sets</a>.

# Download examples

    mkdir ${HOME}/data
    cd ${HOME}/data
    wget https://downloads.mysql.com/docs/world-db.tar.gz
    wget https://downloads.mysql.com/docs/sakila-db.tar.gz
    wget https://downloads.mysql.com/docs/airport-db.tar.gz
    wget https://downloads.mysql.com/docs/menagerie-db.tar.gz
    ls -lh
    for f in *.tar.gz; do tar -I pigz -xvf $f; done
    du -sh *.db

## Example Output

    ls -lh

    -rw-rw-r-- 1 ec2-user ec2-user 626M Dec 14 18:25 airport-db.tar.gz
    -rw-rw-r-- 1 ec2-user ec2-user 2.0K Nov 30 23:05 menagerie-db.tar.gz
    -rw-rw-r-- 1 ec2-user ec2-user 715K Nov 30 23:05 sakila-db.tar.gz
    -rw-rw-r-- 1 ec2-user ec2-user  91K Nov 30 23:05 world-db.tar.gz

    du -sh *.db

    627M	airport-db
    28K	menagerie-db
    3.3M	sakila-db
    392K	world-db

## World Database
To install the world database.

    docker run -i --rm mysql mysql -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} < data/world-db/world.sql
    docker run -i --rm mysql mysql -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} world-db -e "SELECT COUNT(*) FROM city; SELECT COUNT(*) FROM country;"

## Sakila Database

To install the Sakila database.

    docker run -i --rm mysql mysql -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} -f < data/sakila-db/sakila-schema.sql
    docker run -i --rm mysql mysql -h${CLUSTER_ENDPOINT} -u${MYSQL_USER} -p${MYSQL_PASSWD} < data/sakila-db/sakila-data.sql

The creation of the schema must use the --force (-f) option to complete installation do to how the syntax is defined to create a table supporting a FULLTEXT index.

    # ERROR 1238 (HY000) at line 196: Variable 'default_storage_engine' is a read only variable


    194 -- Use InnoDB for film_text as of 5.6.10, MyISAM prior to 5.6.10.
    195 SET @old_default_storage_engine = @@default_storage_engine;
    196 SET @@default_storage_engine = 'MyISAM';
    197 /*!50610 SET @@default_storage_engine = 'InnoDB'*/;

This database is also a good example <a href="mysql-aurora-major-upgade.md">Aurora MySQL Major Upgrade</a> tutorial where it provides some example of deprecated functionality.

# References
  - https://dev.mysql.com/doc/index-other.html
