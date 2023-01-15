Good day, this is a summary of my knowledge on Sqoop (main focus on import).
If you came here for a "ready-made recipe", here it is:
sqoop import -D mapreduce.map.memory.mb=32768 -D mapreduce.map.java.opts=-Xmx28800m -D mapreduce.task.io.sort.mb=9600 -D oracle.row.fetch.size=400000 --num-mappers 20 --fetch-size 200000 --table owner.table_name --connect jdbc:oracle:thin:@address:port:sid --username user --password pwd --direct --verbose --hcatalog-database db_name --hcatalog-table table_name --create-hcatalog-table --hcatalog-storage-stanza "stored as orcfile"

-D oraoop.chunk.method=PARTITION
--map-column-java ROW_ID=String --map-column-hive ROW_ID=String
-D mapreduce.task.timeout=0

The order of the commands is important (the commands starting with -D should be after import and before all the others).

So, first the setup:

### JAVA

RUN yum install -y  java-1.8.0-openjdk-devel
ENV JAVA_HOME=/…… /…… /…… /java-1.8.0-openjdk/

### SQOOP

ENV SQOOP_VERSION=1.4.7
ENV SQOOP_HOME=/opt/sqoop-${SQOOP_VERSION}.bin__hadoop-2.6.0
ENV PATH=$PATH:${SQOOP_HOME}/bin

RUN yum install -y unzip
RUN yum install -y make
RUN curl -fSL http://archive.apache.org/dist/sqoop/1.4.7/sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz -o /tmp/sqoop.tar.gz \
    && tar -xvf /tmp/sqoop.tar.gz -C /opt/ \
    && rm /tmp/sqoop.tar.gz

### NOTES
Make sure that the folder: ~/sqoop-1.4.7.bin__hadoop-2.6.0/lib has the driver file!
By default it is not there
It is necessary to edit the file ~/conf/sqoop-env.sh (by default it is called sqoop-env-template.sh, it is necessary to rename it)
Also it is necessary to rename the file oraoop-site-template.xml -> oraoop-site.xml

Set Hadoop-specific environment variables here:

Set path to where bin/hadoop is available
export HADOOP_VERSION=3.1.1.5
export HADOOP_COMMON_HOME=/opt/hadoop/

Set path to where hadoop-*-core.jar is available
export HADOOP_MAPRED_HOME=/opt/hadoop/

Set the path to where bin/hive is available
export HIVE_HOME=/opt/hive
export HBASE_HOME=
export ZOOCFEDIR=
export SQOOP_USER_CLASSPATH=
or you can use such syntax: 
export HBASE_HOME-${HBASE_HOME:-/opt/…}


#### The whole process boils down to the following steps:
1.	Installing JDK
2.	Installing Sqoop
3.	Adding the driver
4.	Adding variables to the Sqoop configuration file
5.	Adding environment variables

#### Prepare for work:
Grant the following rights to the Oracle account:
grant select on v_$instance to user;
grant select on dba_tables to user;
grant select on dba_tab_columns to user;
grant select on dba_objects to user;
grant select on dba_extents to user;
grant select on dba_segments to user;
grant select on dba_constraints to user;
grant select on v_$database to user;
grant select on v_$parameter to user;
grant select on dba_tab_subpartitions to user;

Now you can start the import. Sqoop is very efficient when transferring #### tables ####. If you need to transfer a view, it uses a full scan.

The command:

sqoop import -D mapreduce.map.memory.mb=32768 -D mapreduce.map.java.opts=-Xmx28800m -D mapreduce.task.io.sort.mb=9600 -D oracle.row.fetch.size=400000 --num-mappers 20 --fetch-size 200000 --table owner.table_name --connect jdbc:oracle:thin:@address:port:sid --username user --password pwd --direct --verbose --hcatalog-database db_name --hcatalog-table table_name --create-hcatalog-table --hcatalog-storage-stanza "stored as orcfile"

-D oraoop.chunk.method=PARTITION
--map-column-java ROW_ID=String --map-column-hive ROW_ID=String
-D mapreduce.task.timeout=0

#### Let's break it down step by step:

1. sqoop import (calling sqoop import)
2. -D mapreduce.map.memory.mb=32768 -D mapreduce.map.java.opts=-Xmx28800m -D mapreduce.task.io.sort.mb=9600 (setting the container size if not enough memory)
3. -D oracle.row.fetch.size=400000 (number of rows the Oracle JDBC driver should fetch in each network round-trip to the database) combined with --fetch-size 200000 in a 1 to 2 ratio
4. --num-mappers 20 (number of threads)
5. --table owner.table_name --connect jdbc:oracle:thin:@address:port:sid --username user --password pwd (connection parameters)
6. --direct (uses Oraoop queries, increasing speed. Does not work with views, use: --split-by <column-name>)
7. --verbose (full log output, helps when problems arise)
8. --hcatalog-database db_name --hcatalog-table table_name (for more stable work I use hcatalog. You can also use: --hive-table <table-name>)
9. --create-hcatalog-table (creating the table. Can also be dropped at the same time)
10. --hcatalog-storage-stanza "stored as orcfile" (storage format)
11. -D oraoop.chunk.method=PARTITION (one partition - one file)
12. --map-column-java ROW_ID=String --map-column-hive ROW_ID=String (sqoop does not know how to handle ROW_ID, have to change the type)
13. -D mapreduce.task.timeout=0 (sometimes view takes a long time to assemble and shoots by timer)
