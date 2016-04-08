# Cross Component Lineage with Apache Atlas, across Apache Sqoop, Storm and Hive

## Introduction
Hortonworks introduced [Apache Atlas](http://hortonworks.com/blog/apache-atlas-project-proposed-for-hadoop-governance/) as part of the [Data Governance Initiative](http://hortonworks.com/press-releases/hortonworks-establishes-data-governance-initiative/), and has continued to deliver on the vision for open source solution for centralized metadata store, data classification, data lifecycle management and centralized security.

Atlas is now offering, as a tech preview, cross component lineage functionality, delivering a complete view of data movement across a number of analytic engines such as Apache Storm, Kafka, Falcon and Hive.
This tutorial walks through the steps for creating data in Apache Hive through Apache Sqoop and Apache Storm

## Prerequisites

Atlas-Ranger preview VM. You can download it from [here](https://s3.amazonaws.com/demo-drops.hortonworks.com/HDP-Atlas-Ranger-TP.ova)
Ensure that VM is fully booted up, all services are up and running. Reference screenshot is below
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/5-atlas-ranger-vm-bootup.png)

## Steps

### Sqoop - Hive Lineage

The VM contains script for creating a mySQL tables, then importing the table using Sqoop into Hive. 

#### Step 1 - Log into the VM. 

The root password to login is 'hadoop'

 ```
ssh root@127.0.0.1 -p 2222 
  ```

Run the following command to get to the scripts for the tutorial.
```  
su - demo
cd sqoop-demo/
```

#### Step 2 - Create a mysql table

Run the below command in your terminal   
```
cat 001-setup-mysql.sql | mysql -u root -p
```
*Note: default password for mysql root user is blank. Press enter when prompted for password*

#### Step 3- Run the SQOOP Job

Run the below command in your terminal  
```
sh 002-run-sqoop-import.sh
```
*Note: default password for mysql root user is blank. Press enter when prompted for password*

Here is the screenshot of results you would see in the screen when you run the above script. 
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/7-sqoop-import-start.png)
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/8-sqoop-import-finish.png)

#### Step 4- Create CTAS sql command

CTAS stands for *create table as select*. We would create a table in Hive from the table imported by the sqoop job above.

Run the below command in your terminal
```
cat 003-ctas-hive.sql | beeline -u "jdbc:hive2://localhost:10000/default" -n hive -p hive -d org.apache.hive.jdbc.HiveDriver
```
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/10-ctas-start.png)


#### Step 4 -View ATLAS UI for the lineage

Click on http://127.0.0.1:21000
Search for *hive_table*, click on *default.cur_hive_table@erietp*
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/9-atlas-hive-search.png)
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/1-sqoop-lineage.png)


## STORM - based lineage 

The following steps will show the lineage of data between Kafka TOPIC (mytopic@erietp) to STORM topology (erie_demo_topology),
which stores the output in the HDFS folder (/user/storm/storm-hdfs-test)

The VM contains script for creating a mySQL tables, then importing the table using Sqoop into Hive. 

#### Step 1 - Log into the VM. 

The root password to login is 'hadoop'

 ```
ssh root@127.0.0.1 -p 2222 
  ```

Run the following command to get to the scripts for the tutorial.
```  
su - demo
cd storm-demo/
```

#### Step 2 -Create a Kafka topic to be used in the demo

Run the following command
```
sh 001-create_topic.sh
```

#### Step 3 - Create a HDFS folder for output

Run the following command
```
sh 002-create-hdfs-outdir.sh
```

#### Step 4 - Download STORM job jar file (optional)

Source is available at https://github.com/yhemanth/storm-samples 

Run the following command
```
sh 003-download-storm-sample.sh
```
As the jar files is already downloaded in the vm, you would see the below information
```
Storm Jar file is already download in /home/demo/storm-demo/lib folder
You can view the source for this at https://github.com/yhemanth/storm-samples 
```

#### Step 5 -Run the Storm JOB 

Run the following command
```
sh 004-run-storm-job.sh
```
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/11-storm-job-start.png)
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/12-storm-job-end.png)

#### Step 5 -View ATLAS UI for the lineage

Go to the Atlas UI http://localhost:21000/

Search for: kafka_topic
Click on: my-topic-01@erietp

![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/3-atlas-kafka-topic.png)
![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/4-kafka-lineage.png)

## Summary

Apache Atlas is the only governance solution for Hadoop that has native hooks within multiple Hadoop components and delivers lineage across these components. With the new preview release, Atlas now supports lineage across data movement in Apache Sqoop, Storm and in Hive. 







