---
layout: tutorial
title: Real Time Data Ingestion in Hbase and Hive using Storm
tutorial-id: 220
tutorial-series: Streaming
tutorial-version: hdp-2.4.0
intro-page: false
components: [ storm, hbase, hive, kafka, nifi ]
---

# Lab 2: Real Time Data Ingestion in Hbase and Hive using Storm

## Introduction

The Trucking business is a high-risk business in which truck drivers venture into remote areas, often in  harsh weather conditions and chaotic traffic on a daily basis. Using this solution illustrating Modern Data Architecture with Hortonworks Data Platform, we have developed a centralized management system that can help reduce risk and lower the total cost of operations.

This system can take into consideration adverse weather conditions, the driver's driving patterns, current traffic conditions and other criteria to alert and inform the management staff and the drivers themselves when risk factors run high.

In previous tutorial, we have explored generating and capturing streaming data with [Apache NiFi](#rtep-1.md) and [Apache Kafka](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/).

In this tutorial, you  will use [**Apache Storm**](http://hortonworks.com/labs/storm/) on the Hortonworks Data Platform to capture these data events and process them in real time for further analysis.

In this tutorial, we will build a solution to ingest real time streaming data into HBase and HDFS using [Storm](http://hortonworks.com/hadoop-tutorial/ingesting-processing-real-time-events-apache-storm/). Storm has a spout that reads truck_events data from Kafka and passes it to bolts, which process and persist the data into Hive & HBase tables.

## Pre-Requisites

- [Lab #0](rtep-1.md) Ingest, Route and Land Real Time Events with Apache NiFi
- [Lab #1](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/) Capture Real Time Events with Apache Kafka
- Downloaded and Installed the latest [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
- [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/)
- [Hive Quick Start](https://hbase.apache.org/book.html#quickstart)


## Outline

*   [HBase](#hbase-concept-lab3)
*   [Apache Storm](#apache-storm-concept-lab3)
*   [Step 1: Start HBase & Storm](#step-1-start-hbase-storm-lab3)
*   [Step 2: Create tables in HDFS and HBase](#step2-create-tables-hdfs-hbase-lab3)
*   [Step 3: Run Automation Script: Setup Demo Modules](#step3-run-auto-script-lab3)
*   [Step 4: Launch new Storm Topology](#step4-launch-new-storm-topology-lab3)
*   [Step 5: Verify Data in HDFS and HBase](#step5-verify-data-hdfs-hbase-lab3)
*   [Conclusion](#conclusion-lab3)
*   [Appendix A: Run the Trucking Demo with NiFi Integration](#run-the-trucking-demo-lab3)
*   [Appendix B: Update iot-truck-streaming Project](#update-iot-truck-streaming-project-lab3)
*   [Appendix C: Enable Remote Desktop and Set up Storm Topology as an Eclipse Project](#enable-remote-desktop-setup-topology-lab3)
*   [Further Reading](#further-reading-lab3)

## HBase <a id="hbase-concept-lab3"></a>

HBase provides near real-time, random read and write access to tables (or to be more accurate 'maps') storing billions of rows and millions of columns.

In this case, once we store this rapidly and continuously growing dataset from Internet of Things (IoT), we will be able to perform a swift lookup for analytics regardless of the data size.

## Apache Storm <a id="apache-storm-concept-lab3"></a>

Apache Storm is an Open Source distributed, reliable, fault–tolerant system for real time processing of large volume of data.
It's used for:
*   Real time analytics
*   Scoring machine learning modeles
*   Continuous statics computations
*   Operational Analytics
*   And, to enforce Extract, Transform, and Load (ETL) paradigms.


A Storm Topology is network of Spouts and Bolts. The Spouts generate streams, which contain sequences of tuples (data) while the Bolts process input streams and produce output streams. Hence, the Storm Toplogy can talk to databases, run functions, filter, merge or join data.
*   **Spout**: Works on the source of data streams. In the "Truck Events" use case, Spout will read data from Kafka topics.
*   **Bolt**: Spout passes streams of data to Bolt which processes and persists  it to a data store or sends it downstream to another Bolt.

Learn more about Apache Storm at the [Storm Documentation page](http://storm.apache.org/releases/1.0.0/index.html).

## Tutorial Overview

*   Create HBase & Hive Tables
*   Create Storm Topology
*   Configure a Storm Spout and Bolts.
*   Store Persisting data in HBase and Hive.
*   Verify Data Stored in HDFS and HBase.

### Step 1: Start HBase & Storm <a id="step-1-start-hbase-storm-lab3"></a>


1\.  **View the HBase Services page**

Started by logging into Ambari as an admin user. From the previous tutorials: HDFS, Hive, YARN and Kafka should already be running but HBase may be down. From the Dashboard page of Ambari, click on HBase from the list of installed services.

![hbase_service_on_off_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/hbase_service_on_off_iot.png)

2\. Start HBase

From the HBase page, click on Service Actions -> Start

![start_hbase_service_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/start_hbase_service_iot.png)

Check the box and click on Confirm Start:

![confirm_hbase_start_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/confirm_hbase_start_iot.png)

Wait for HBase to start (It may take a few minutes to turn green)

![hbase_started_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/hbase_started_iot.png)


3\. Start Storm the same way we started HBase in the previous steps. We will need it later for streaming real-time event data.

![storm_service_on_off_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/storm_service_on_off_iot.png)

4\. After starting storm, a green check symbol will be present:

![storm_service_started_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/storm_service_started_iot.png)


You can use the Ambari dashboard to check status of other components too. If **HDFS, Hive, YARN, Kafka, Storm or HBase** are down, you can start them in the same way: by selecting the service and then using the Service Actions to start it. The remaining components do not have to be up. (Oozie can be stopped to save memory, as it is not needed for this tutorial)


### Step 2: Create tables in HDFS & HBase <a id="step2-create-tables-hdfs-hbase-lab3"></a>

*   Create HBase tables

We will be working with 3 Hbase tables in this tutorial.

The first table stores all events generated, the second stores the 'driverId' and non-normal events count and third stores number of non-normal events for each driverId.

~~~
[root@sandbox ~]$ su hbase

[hbase@sandbox root]$ hbase shell

hbase(main):001:0> create 'driver_events', 'allevents'    
hbase(main):002:0> create 'driver_dangerous_events', 'events'
hbase(main):003:0> create 'driver_dangerous_events_count', 'counters'
hbase(main):004:0> list    
hbase(main):005:0> exit
~~~

> Note: 'driver_events' is the table name and 'allevents' is column family. In the script above, we have one column family. Yet, if we want we can have multiple column families. We just need to include more arguments.


![Screen Shot 2015-06-04 at 7.03.00 PM.png](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/hbase_tables_iot.png)

Next, we will create 4 Hive tables. For each table in the following section, the table was built based on the truck event data generated. The tables contain attributes associated with that data.


*   Create Hive tables

The first two tables will stores information about the vehicle.

Open the Hive view in Ambari in a browser, copy the below script into the query editor and click Execute: [http://localhost:8080/#/main/views/HIVE/1.0.0/Hive](http://localhost:8080/#/main/views/HIVE/1.0.0/AUTO_HIVE_INSTANCE)


~~~sql
create table truck_events_text_partition
(driverId int,
truckId int,
eventTime string,
eventType string,
longitude double,
latitude double,
eventKey string,
correlationId bigint,
driverName string,
routeId int,
routeName string)
partitioned by (eventDate string)
row format delimited fields terminated by ','
stored as textfile;
~~~


This hive query creates the Hive table to persist all events generated. The table is partitioned by date.

![truck_events_text_partition_table_hbase_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/truck_events_text_partition_table_hbase_iot.png)

Verify that the table has been properly created by refreshing the Database Explorer. Under Databases, click default to expand this table and the new table should appear. Clicking on the List icon next to truck_events_text_partition shows that the table was created but empty.

![verify_truck_events_text_partition_created](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/verify_truck_text_partition_created_iot.png)

*   Create ORC 'truck_events' Hive tables

The Optimized Row Columnar (ORC) file format provides a highly efficient way to store Hive data. It was designed to overcome limitations of the other Hive file formats. Using ORC files improves performance when Hive is reading, writing, and processing data.

Syntax for ORC tables:

`CREATE TABLE … STORED AS ORC`

`ALTER TABLE … [PARTITION partition_spec] SET FILEFORMAT ORC`

**Note**: This statement only works on partitioned tables. If you apply it to flat tables, it may cause query errors.

Next let's create the 'truck_events' table as per the above syntax. Paste the below into the worksheet of the Hive view and click Execute

~~~sql
create table truck_events_orc_partition_single
(driverId int,     
truckId int,
eventTime string,
eventType string,
longitude double,
latitude double,
eventKey string,
correlationId bigint,
driverName string,
routeId int,
routeName string)
partitioned by (eventDate string)
row format serde 'org.apache.hadoop.hive.ql.io.orc.OrcSerde'
stored as orc
TBLPROPERTIES ("orc.compress"="NONE");
~~~

Refresh the Database Explorer and you should see the new table appear under default:

![](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/truck_events_orc_partition_table.png)

The data in 'truck_events_orc_partition_single' table can be stored with ZLIB, Snappy, LZO compression options. This can be set by changing tblproperties ("orc.compress"="NONE")option in the query above.

The last two tables store data about the drivers.
~~~sql
CREATE TABLE `drivers`
(`driverid` bigint,
`name` string,
`certified` string,
`wage_plan` string)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';
~~~

This hive query creates hive table to persist all information regarding the driver.

![drivers_table_hbase_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/drivers_table_hbase_iot.png)


~~~sql
CREATE TABLE `timesheet`
(`driverid` bigint,
`week` bigint,
`hours_logged` bigint,
`miles_logged` bigint)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';
~~~

This hive query creates a hive table to persist data regarding the driver’s total time, distance and days driving.


![timesheet_table_hbase_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/timesheet_table_hbase_iot.png)


Set permissions on `/tmp/hive`

~~~bash
chmod -R 777 /tmp/hive/
~~~

![set_permissions_hive_iot_hbase](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/set_permissions_hive_iot_hbase.png)


### Step 3: Run the Automation script: Setup Demo Modules

Since this tutorial series is based on part of the trucking demo, there are many modules that need to be setup for the demo outside the scope of the tutorial. We manually setup NiFi, Kafka, HBase and Hive for the demo. Since there are other particular modules in the demo irrelevant from what we are learning in the lab series, we will run an automation script to setup the other modules that way we will be able to use storm for ingesting data in HBase and HDFS with no issues.

1\. Update ambari admin login variables defined at the top in **user-env.sh** file, so the automation script can have the privileges to setup the demo modules. Enter the **username and password** you use to login into to Ambari as an admin. Open a terminal, type:

~~~
vi ~/iot-truck-streaming/user-env.sh
~~~

The file will open as in the image below:

![user_env_sh_setup_auto_script_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/user_env_sh_setup_auto_script_iot.png)

For the ambari configuration credentials: user='admin', pass=what you set it up as manually. For example, after setting up my password, I would enter user='admin', pass='h@d0op.'

Press `esc` and then type `:wq` to exit the editor.

2\. After you update the **user-env.sh** file, we will also need to verify whether the hostnames in the **config.properties** file match the appropriate hostnames for services on HDP. If they do not match, then update the hostname. For example, let's check the **kafka.brokers** host, open Ambari dashboard. Hover to the left side bar, click on **Kafka**. At the top next to the `Summary` tab, click on the `Configs` tab. Under **Kafka Broker** Section, examine **Kafka Broker host** and **listeners** field. You should see the following image:

![kafka_broker_hostname_verify](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/kafka_broker_hostname_verify.png)

Notice Kafka Broker host = sandbox.hortonworks.com
Listeners = localhost:6667

Thus, our **Kafka Broker and Listenrs host** = `sandbox.hortonworks.com:6667`

In our **config.properties** file, under Stream Simulator Config, it shows:

![config_properties_file_verify_hosts_match](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/config_properties_file_verify_hosts_match.png)

> Note: In the config.properties file Kafka.brokers=sandbox.hortonworks.com:6667

Since the **kafka.brokers** hostname in the **config.properties** file matches kafka brokers hostname on HDP, we verified that hostname is up to date. Now let’s verify the other hostnames in the config.properties file match the ones on HDP. If there is a mismatch, update the config.properties file.

3\. Now we can run the installdemo.sh script to automatically setup the background services for the trucking demo. Type the following command:

~~~
cd iot-truck-streaming/
./installdemo.sh
~~~

Once we build and install the necessary modules for the demo, we are ready to deploy our storm topology.


### Step 4: Launch Storm Topology <a id="step4-launch-new-storm-topology-lab3"></a>


Recall that the source code is under directory path
`iot-truck-streaming/storm-streaming/src/`.

The pre-compiled jars are under the directory path
`iot-truck-streaming/storm-streaming/target/`.

**(Optional)** If you would like to modify/run the code:

*   refer to [Appendix B](#update-iot-truck-streaming-project-lab3) for the steps to run maven to compile the jars to the target subdir from terminal command line
*   refer to [Appendix C](#enable-remote-desktop-setup-topology) for the steps to enable VNC (i.e. 'remote desktop') access on your sandbox and open/compile the code using Eclipse

### 4.1 Verify Kafka is Running & Create Topology

1\. Verify that Kafka service is running using Ambari dashboard. If not, start the Kafka service as we did in lab 1.


2\. Create Storm Topology

We now have 'supervisor' daemon and Kafka processes running.
To do real-time computation on Storm, you create what are called "topologies". A topology is a Directed Acyclic Graph (DAG) of spouts and bolts with streams of tuples representing the edges. Each node in a topology contains processing logic, and links between nodes indicate how data should be passed around between nodes.

Running a topology is straightforward. First, you package all your code and dependencies into a single jar like we did in the lab0 with mvn clean package. Then, you run a command like the following: The command below will start a new Storm Topology for Truck Events.


~~~bash
[root@sandbox iot-truck-streaming]# storm jar storm-streaming/target/storm-streaming-1.0-SNAPSHOT.jar com.hortonworks.streaming.impl.topologies.TruckEventProcessorKafkaTopology /etc/storm_demo/config.properties
~~~


You should see that the topology deployed successfully:

![Screen Shot 2015-06-04 at 7.55.23 PM.png](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/storm_topology_submitted_success_storm_iot.png)

This runs the class **TruckEventProcessorKafkaTopology**. The main function of the class defines the topology and submits it to Nimbus. The storm jar part takes care of connecting to Nimbus and uploading the jar.

Open your Ambari Dashboard. Click the Storm Service located in the ambari service list. Click the Quick Links Dropdown button at the top middle between Configs and Service Actions, then click the Storm UI button to enter the Storm UI. You should see the new Topology **truck-event-processor**.


![Topology Summary](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/storm_ui_welcome_screen_iot.png)

Run the NiFi DataFlow to generate events.
Return to the Storm UI and click on truck-event-processor topology to drill into it.  Under Spouts, after 6-10 minutes, you should see that numbers of emitted and transferred tuples is increasing which shows that the messages are processed in real time by Spout


![](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/topology_spouts_bolts_tuples_increasing.png)


Under Topology Visualization: You shall see here that Kafka Spout has started writing to hdfs and hbase along with the other bolts.


![](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/topology_visualization_storm.png)

Note: You can also keep track of several statistics of Spouts and Bolts. For instance, to find Spouts Statistics, click on **kafkaSpout** located in the Spouts section.

![spout_statistics_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/spout_statistics_iot.png)

### Step 5: Verify Data in HDFS and HBase <a id="step5-verify-data-hdfs-hbase-lab3"></a>

Since the NiFi DataFlow was activated in the last step, let’s verify that Storm spout has started writing data to HDFS and HBase.


*   Verify the data is in HDFS by opening the Ambari Files view: `http://localhost:8080/#/main/views/FILES/0.1.0/MyFiles`

With the default settings for HDFS, users will see the data written to HDFS once every few minutes.

![hdfs_files_view_base_directory](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/hdfs_files_view_base_directory.png)

Drill down into `/truck-events-v4/staging` dir in HDFS

![](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/storm_loads_txt_files_events_lab2_iot.png)

Stop the NiFi DataFlow to no longer send messages to Kafka. Now go back to the staging directory, click on one of the txt files and confirm that it contains the events:

![Screen Shot 2015-06-04 at 9.20.24 PM.png](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/verify_hdfs_files_contain_data_iot.png)

> **Note:** It may take a 5-10 minutes, before you can access the txt files to see the data.

*   Verify data in Hive by navigating to the Hive view, expanding the default database and clicking the List icon next to **truck_events_text_partition table**

![Screen Shot 2015-06-04 at 9.13.23 PM.png](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/truck_text_partition_table_load_sample_iot.png)

*   If you haven't done so, you can you can stop the NiFi DataFlow. Press the stop symbol.
*   Verify that the data is in HBase by executing the following commands in HBase shell:

~~~
[root@sandbox Tutorials-master]# hbase shell

hbase(main):001:0> list
hbase(main):002:0> count 'driver_events'
hbase(main):003:0> count 'driver_dangerous_events'
hbase(main):004:0> count 'driver_dangerous_events_count'    
hbase(main):005:0> exit
~~~

The `driver_dangerous_events` table is updated upon every violation.

![verify_data_in_hbase_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/verify_data_in_hbase_iot.png)

*   Next let's populate the data into ORC table for interactive query by Excel (or any BI tool) via ODBC over Hive/Tez. Open the Hive view and enter the below and click Execute.

~~~sql
INSERT OVERWRITE TABLE truck_events_orc_partition_single
partition (eventDate)
select * from truck_events_text_partition;
~~~

![populate_orc_with_data_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/populate_orc_with_data_iot.png)

Notice that this launches a Tez job in the background. You can get more details on this using the Yarn resource manager UI. You can find for this under the link under Ambari -> Yarn -> Quick links but will be similar to `http://localhost:8088/cluster`

![yarn_resource_manager_tez_job_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/yarn_resource_manager_tez_job_iot.png)

Now query the ORC table by clicking the List icon next to it under Databases and notice it is also now populated

![](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/load_sample_orc_table_populated_iot.png)

*   Once done, stop the Storm topology

The Storm topology can be deactivated/killed from the Storm UI or shell

~~~bash
storm kill TruckEventProcessorKafkaTopology
~~~

![storm_topology_actions_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/storm_topology_actions_iot.png)


## Conclusion <a id="conclusion-lab3"></a>

Congratulations, you built your first Hortonworks DataFlow Application. When NiFi, Kafka and Storm are combined, they create the Hortonworks DataFlow. You have used the power of NiFi to ingest, route and land real-time streaming data. You learned to capture that data with Kafka and perform instant processing with Storm. A common challenge with many use cases, which is also observed in this lab series is ingesting a live stream of random data, and filtering the junk data from the actual data we care about. Through these labs, you learned to manipulate, persist and perform many other operations on random data.

### Appendix A: Run the Trucking Demo with NiFi Integration <a id="run-the-trucking-demo-lab3"></a>

The trucking demo shows realtime monitoring of alerts and predictions of driving violations by fleets of trucks. The demo visually illustrates these events on a map. Let's start the demo to observe these realtime events in action.

### A.1 Start the Trucking Demo

1\. Navigate to the base of the trucking demo project folder, make sh files executable, then execute the rundemo.sh script. Starting the demo may take 15 - 20 minutes:

~~~
cd ~/iot-truck-streaming
chmod 750 *.sh
./rundemo.sh clean
~~~

Note: rundemo.sh clean kills the storm topology, stops storm, cleans the storm directories, restarts storm and redeploys the topology. rundemo.sh is modified with the assumption that you completed the Lab Series, specifically you manually installed maven back in lab 0, Kafka, Storm, HBase services are running and updated the user-env.sh file in lab 2. rundemo.sh will setup and start the demo.

When you see **"[INFO] Started Jetty Server"** message up in the console, you will be able to access the demo at:

~~~
http://<hostname>:8081/storm-demo-web-app/index.html.
~~~

If on virtualbox, the hostname will be: http://127.0.0.1:8081/storm-demo-web-app/index.html.

If you receive the message, **"This site can't be reached"**, you will need to port forward `8081` onto your virtual machine. Refer to [lab 0 step 3](#step3-start-nifi) where we port forward NiFi port number if you need to review.

### A.2 Login to Trucking Demo Dashboard

Once connected to Jetty Server, the following login page appears, user and password are given by default, so press the **sign in** button:

![trucking_demo_sign_in](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/trucking_demo_sign_in.png)

The HDP Storm Demo Dashboard will appear:

![hdp_storm_demo_dashboard_iot](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/hdp_storm_demo_dashboard_iot.png)

### A.3 Run NiFi DataFlow & Topology Tuples Increase

Before entering one of these applications on the dashboard as in the image above, make sure your NiFi DataFlow is running and that your storm topology spout/bolt tuples are increasing. You should have images similar to as below:

![dataflow_withKafka_running_iot](/assets/realtime-event-processing-with-hdf/lab1-kafka/dataflow_withKafka_running_iot.png)

> DataFlow is running and sending events to Kafka. If you notice events stop being sent to kafka, stop and start the DataFlow.

![storm_topology_tuples_increasing](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/storm_topology_tuples_increasing.png)

> Storm Topology Spout/Bolt tuples increasing

### A.4 Troubleshooting Section

If storm shows an internal server error, refer to the **troubleshooting section below**, else skip to the next section:

If in the Storm UI and it shows that storm nimbus is not coming up, or you are getting an error similar to:
`java.lang.RuntimeException: Could not find leader nimbus from seed hosts [sandbox.hortonworks.com]. Did you specify a valid list of nimbus hosts for config nimbus.seeds`

Stop Storm and run the following commands to clean out the old data.

~~~
./iot-truck-streaming/setup/bin/cleanupstormdirs.sh
/usr/hdp/current/zookeeper-client/bin/zkCli.sh
rmr /storm
~~~

If any other issues, reset and restart the demo:

~~~
./iot-truck-streaming/setup/bin/cleanup.sh
~~~

Now let's start the Storm service. We'll need to redeploy our topology:

~~~
[root@sandbox ~]# cd iot-truck-streaming/
[root@sandbox iot-truck-streaming]# storm jar storm-streaming/target/storm-streaming-1.0-SNAPSHOT.jar com.hortonworks.streaming.impl.topologies.TruckEventProcessorKafkaTopology /etc/storm_demo/config.properties
~~~

We also need to run our NiFi DataFlow. Now we can we can explore the different applications within the demo.

### A.5 Explore Trucking Demo Applications

If you can see your NiFi DataFlow sending truck event data to Kafka and Storm tuples increasing, enter the applications and you shall the see the following maps and tables:


Real-Time Driver Monitoring Application

![driver_monitoring_app_map](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/driver_monitoring_app_map.png)
![driver_monitoring_app_table](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/driver_monitoring_app_table.png)


Real-Time Driver Behavior Predictions Application

![driver_behavior_predictions_app_map](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/driver_behavior_predictions_app_map.png)
![driver_behavior_predictions_app_table](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/driver_behavior_predictions_app_table.png)


Real-Time Drools Driven Monitoring Application

![drools_driven_driver_alerts_app_map](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/drools_driven_driver_alerts_app_map.png)
![drools_driven_driver_alerts_app_table](/assets/realtime-event-processing-with-hdf/lab2-hbase-hive-storm/drools_driven_driver_alerts_app_table.png)

Congratulations, you just incorporated NiFi into the trucking demo. Notice that each time the rows in the table turn red, it indicates a prediction that the driver committed a violation while driving. On the map, the green dots indicate probability that the driver will not commit a violation while red dots indicate the opposite.


### Appendix B: Update iot-truck-streaming Project <a id="update-iot-truck-streaming-project-lab3"></a>

*   Copy /etc/hbase/conf/hbase-site.xml to src/main/resources/ directory

~~~bash
[root@sandbox ~]# cd /iot-truck-streaming
[root@sandbox ~]# cp /etc/hbase/conf/hbase-site.xml src/main/resources/
~~~


*   Check pom.xml to ensure it includes the below dependencies (check after **line 104**)


~~~html
    <dependency>
      <groupId>xerces</groupId>
      <artifactId>xercesImpl</artifactId>
      <version>2.9.1</version>
    </dependency>

    <dependency>
      <groupId>xalan</groupId>
      <artifactId>xalan</artifactId>
      <version>2.7.1</version>
    </dependency>

    <dependency>
      <groupId>org.htrace</groupId>
      <artifactId>htrace-core</artifactId>
      <version>3.0.4</version>
    </dependency>

    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-hdfs</artifactId>
      <version>2.6.0</version>
    </dependency>
~~~

*   recompile the Maven project. This may run for 10+ min

~~~bash
[root@sandbox ~]# mvn clean package
~~~


The maven build should succeed.


### Appendix C: Enable remote desktop on sandbox and set up Storm topology as Eclipse project <a id="enable-remote-desktop-setup-topology-lab3"></a>

1.  Setup Ambari VNC service on the sandbox to enable remote desktop via VNC and install eclipse using steps here [https://github.com/hortonworks-gallery/ambari-vnc-service#setup-vnc-service](https://github.com/hortonworks-gallery/ambari-vnc-service%23setup-vnc-service)
2.  Import code as Eclipse project using steps here:

[https://github.com/hortonworks-gallery/ambari-vnc-service#getting-started-with-storm-and-maven-in-eclipse-environment](https://github.com/hortonworks-gallery/ambari-vnc-service%23getting-started-with-storm-and-maven-in-eclipse-environment)

## Further Reading <a id="further-reading-lab3"></a>
- [Apache HBase](http://hortonworks.com/hadoop/hbase/)
- [Getting Started with HBase](https://hbase.apache.org/book.html#quickstart)
- [Storm Hive Integration](http://storm.apache.org/documentation/storm-hive.html)
- [Storm Tutorials](http://hortonworks.com/hadoop/storm/#tutorials)
- [Getting Started with Apache Storm](http://storm.apache.org/documentation.html)
- [Apache Storm](http://hortonworks.com/hadoop/storm/)
