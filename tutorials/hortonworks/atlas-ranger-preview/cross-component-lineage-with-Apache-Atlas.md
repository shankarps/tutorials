# Cross Component Lineage with Apache Atlas, across Apache Sqoop, Storm and Hive

## Introduction
Hortonworks introduced [Apache Atlas](http://hortonworks.com/blog/apache-atlas-project-proposed-for-hadoop-governance/) as part of the [Data Governance Initiative](http://hortonworks.com/press-releases/hortonworks-establishes-data-governance-initiative/), and has continued to deliver on the vision for open source solution for centralized metadata store, data classification, data lifecycle management and centralized security.

Atlas is now offering, as a tech preview, cross component lineage functionality, delivering a complete view of data movement across a number of analytic engines such as Apache Storm, Kafka, Falcon and Hive.
This tutorial walks through the steps for creating data in Apache Hive through Apache Sqoop and Apache Storm

## Prerequisites

Atlas-Ranger preview VM. You can download it from here (link)

## Steps

### Sqoop - Hive Lineage

The VM contains script for creating a mySQL tables, then importing the table using Sqoop into Hive. 

#### Step 1 Create a mysql table
*Note: default password is blank*

SSH into your VM. The password to login is 'hadoop'
  HW10957:~ bganesan$ ssh root@127.0.0.1 -p 2222
  root@127.0.0.1's password: 
  Last login: Wed Apr  6 19:36:31 2016 from 10.0.2.2

*Note: It is recommend to change the default root password once you have installed the VM in your environment*

Run the below command in your terminal   
`cat 001-setup-mysql.sql | mysql -u root -p`


#### Step 2 Run the SQOOP Job
*Note: ** default password is blank *

Run the below command in your terminal  
`sh 002-run-sqoop-import.sh`


#### Step 3 Create CTAS sql command

Run the below command in your terminal
`cat 003-ctas-hive.sql | beeline -u "jdbc:hive2://localhost:10000/default" -n hive -p hive -d org.apache.hive.jdbc.HiveDriver'

![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/1-sqoop-lineage.png)

![](https://github.com/hortonworks/tutorials/blob/atlas-ranger-tp/assets/cross-component-lineage-with-atlas/2-hive-table-details.png)


STORM - based lineage 


This demo will show the lineage of data between Kafka TOPIC (mytopic@erietp) to STORM topology (erie_demo_topology),
which stores the output in the HDFS folder (/user/storm/storm-hdfs-test)



# Create a Kafka topic to be used in the demo

001-create_topic.sh


# Create a HDFS folder for output

002-create-hdfs-outdir.sh


 Download STORM job jar file - Source is available at https://github.com/yhemanth/storm-samples 

003-download-storm-sample.sh


# Run the Storm JOB 


004-run-storm-job.sh


 View ATLAS UI for the lineage

http://localhost:21000/

Search for: kafka_topic
Click on: my-topic@erietp











