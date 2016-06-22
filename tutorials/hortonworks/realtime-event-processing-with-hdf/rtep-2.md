---
layout: tutorial
title: Capture Real Time Event Stream with Apache Kafka
tutorial-id: 220
tutorial-series: Streaming
tutorial-version: hdp-2.4.0
intro-page: false
components: [ kafka, nifi ]
---


# Lab 1: Capture Real Time Event Stream with Apache Kafka

## Introduction

[Apache Kafka](http://kafka.apache.org/) can be used on the Hortonworks Data Platform to capture real-time events. We will begin with showing you how to configure Apache Kafka and Zookeeper. Next we will show you how to capture truck event data from Apache NiFi using Kafka.

## Pre-Requisites
*  Lab 0 Ingest, Route and Land Real Time Events with Apache NiFi
*  Downloaded and Installed latest [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
*  You will need admin privileges for this tutorial, refer to [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/) to setup your Ambari admin password
*  Memory must be at least 8GB RAM, preferably 4 processor cores, else errors may occur in third tutorial

## Outline

*   [Apache Kafka](#apache-kafka-lab1)
*   [Step 1: Navigate to Ambari User Views](#navigate-ambari-user-views-lab1)
*   [Step 2: Start Apache Kafka](#start-kafka-lab1)
*   [Step 3: Configure Kafka with Zookeeper](#configure-kafka-with-zookeeper-lab1)
*   [Step 4: Create Kafka Topic](#define-kafka-topic-lab1)
*   [Step 5: Create NiFi PutKafka Processor](#create-nifi-putkafka-lab1)
*   [Summary](#summary-lab1)
*   [Appendix A: Install Kafka](#install-kafka-lab1)
*   [Further Reading](#further-reading-lab1)

## Apache Kafka <a id="apache-kafka-lab1"></a>

[Apache Kafka](http://kafka.apache.org/) is an open source messaging system designed for:

*   Persistent messaging
*   High throughput
*   Distributed
*   Multi-client support
*   Real time

![Kafka Producer-Broker-Consumer](/assets/realtime-event-processing-with-hdf/lab1-kafka/Kafka-Broker-Diagram.png)

Kafka Producer-Broker-Consumer

## Tutorial Overview

1.  Install and Start Kafka on latest [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/).
2.  Review Kafka and ZooKeeper Configs
3.  Create Kafka topics for Truck events.
4.  Write Kafka Producers for Truck events.

### Step 1: Navigate to Ambari <a id="navigate-ambari-user-views-lab1"></a>

### 1.1 Start the Hortonworks Sandbox

After downloading the Sandbox and running the VM, login to the Sandbox using the URL displayed in the console. For example, the URL in the screenshot below is `http://127.0.0.1:8888/`

![sandbox_vm_welcome_screen](/assets/realtime-event-processing-with-hdf/lab1-kafka/sandbox_vm_welcome_screen.png)

### 1.2 Login to Ambari

After resetting your Ambari admin password with assistance from [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/), Go to port 8080 of your Sandbox’s IP address to view the Ambari login page. For example, `http://127.0.0.1:8080:`

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/login_page_ambari.png)


### Step 2: Start Apache Kafka <a id="start-kafka-lab1"></a>

### 2.1  View the Kafka Services page

From the Dashboard page of Ambari, click on Kafka from the list of installed services. (If Kafka is not installed, perform the steps in [Appendix A](#appendix-a) first.):

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/kafka_service_on_off.png)

### 2.2 Enable Kafka

From the Kafka page, click on Service Actions -> Start:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/start_kafka_service_iot.png)

Check the box and click on Confirm Start:

![Screen Shot 2015-06-04 at 3.06.10 PM.png](/assets/realtime-event-processing-with-hdf/lab1-kafka/confirmation_kafka_service_start.png)

Wait for Kafka to start.

### Step 3: Configure Kafka with ZooKeeper <a id="configure-kafka-with-zookeeper-lab1"></a>

ZooKeeper serves as the coordination interface between the Kafka broker and consumers:

![Single Broker based Kakfa cluster](/assets/realtime-event-processing-with-hdf/lab1-kafka/zookeeper-kafka-producer-broker-consumer.jpg)

The important Zookeeper properties can be checked in Ambari.

### 3.1  Configure ZooKeeper

Click on **ZooKeeper** in the list of services, then click on the Configs tab. Verify ZooKeeper is running on port 2181:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/zookeeper_port_config.png)

If this port 2181 is busy or is consumed by other processes, then you could change the default port number of ZooKeeper to any other valid port number. If ZooKeeper is not running, you can start the Zookeeper service from Ambari:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/zookeeper_start_service_iot.png)

### 3.2 Configure Kafka

From the Kafka page, click on the **Configs** tab. Verify the `zookeeper.connect` property points to your ZooKeeper server name and port:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/verify_zookeeper_connect_pts_zoo_server_kafka.png)

### Step 4: Define a Kafka Topic <a id="define-kafka-topic-lab1"></a>

### 4.1 SSH into the Sandbox

SSH into the Sandbox to define the Kafka topic. Type the following command:

~~~bash
ssh root@127.0.0.1 -p 2222
~~~

![ssh_into_sandbox_shell_kafka_iot](/assets/realtime-event-processing-with-hdf/lab1-kafka/ssh_into_sandbox_shell_kafka_iot.png)

> NOTE: You can also SSH using a program like Putty for Windows or the Terminal application on a Mac.

### 4.2 Create a new Topic

Use the `kafka-topics.sh` script (which should be in your PATH), create a new topic named `truck_events`:

~~~bash
[root@sandbox ~]# kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 2 --topic truck_events
~~~

If the `kafka-topics.sh` script is not in your PATH and you get a command not found error, then change directories to where the Kafka scripts are installed:

~~~bash
[root@sandbox ~]# cd /usr/hdp/current/kafka-broker/bin/
~~~

You will need to **add a dot and a slash (./)** to the beginning of the commands:

~~~bash
[root@sandbox bin]# ./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 2 --topic truck_events
~~~

Also note that sometimes ZooKeeper does not listen on `localhost`, so you may need to use the Sandbox’s IP address instead.


The output should show your topic was created:

![created_kafka_topic_iot](/assets/realtime-event-processing-with-hdf/lab1-kafka/created_kafka_topic_iot.png)


### 4.3 Verify the topic was created successfully

Check if topic `truck_events` was created successfully with the following command:

~~~bash
[root@sandbox ~]# ./kafka-topics.sh --list --zookeeper localhost:2181
~~~

You should see `truck_events` in the list of topics (and probably your only topic):

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/verify_kafka_topic_created_iot.png)


### Step 5: Create NiFi PutKafka Processor <a id="create-nifi-putkafka-lab1"></a>

In the previous tutorial, we stored the truck event data into a file. Now we can use the [PutKafka](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi.processors.kafka.PutKafka/index.html) processor since the Kafka service is running, we have access to the "Known Broker", "Topic Name" and "Client Name." We will send the truck event contents of a FlowFile to Kafka as a message. Similar to the Kafka Producer, NiFi acts as a producer since it creates messages and publishes them to the Kafka broker for further consumption.

1\. If not already open, navigate to the NiFi Web Interface at `http://127.0.0.1:6434/nifi/`. For vmware and azure, the host and port may be different.

2\. If your data flow is still running, click on the stop button ![stop_symbol_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/stop_symbol_nifi_iot.png) in the **actions** toolbar to stop the flow.

### 5.1 Add PutKafka Processor

1\. Drag the processor icon onto your graph. Add the **PutKafka** processor.

2\. Right click on the Putkafka processor, click on **configure**.

3\. The warning message tells us, we need to add "Known Broker", "Topic Name" and "Client Name" values to our Properties section. Enter the following information into the **Properties** section in the **Configure Processor** window:

~~~
# Property = Value
Known Brokers = sandbox.hortonworks.com:6667
Topic Name = truck_events
Message Delimiter = press “Shift+enter”
Client Name = truck_events_client
~~~

![putkafka_processor_config_nifi_iot](/assets/realtime-event-processing-with-hdf/lab1-kafka/putkafka_processor_config_nifi_iot.png)

> Note: Every property above is required except **Message Delimiter**, but this property is helpful with splitting apart the contents of the FlowFile.

**Known Brokers** can be found in **Kafka** configs under listeners

**Topic Name** is the name you created earlier for Kafka. Type the following command to see your topic name: ./kafka-topics.sh --list --zookeeper localhost:2181.

**Message Delimiter** set as "Shift+enter" in the value field makes each line of incoming FlowFile a single message, so kafka does not receive an enormous flowfile as a single message.

**Client Name** can be named to your liking, it is the name that is used when communicating with kafka.

4\. Open the Configure Processor window again, navigate to the **Settings** tab. Set the **Auto termination relationship** to `success` and `failure`. Click **apply**.

5\. Connect the **MergeContent(truck_events)** processor to **PutKafka**. A Create Connection window will appear, set the **For relationship** to `merged`.

You should obtain a similar dataflow as the following:

![dataflow_final_withkafka_iot](/assets/realtime-event-processing-with-hdf/lab1-kafka/dataflow_final_withkafka_iot.png)

> Note: If there is a warning symbol after updating the PutKafka, verify that the property values are correct. Check **3**. in case you need to review the values changed.

6\. Let’s start our Hortonworks DataFlow to see a real live stream of truck event data be read from NiFi and written to a Kafka cluster. In the **actions** toolbar, hit the **start** button.

![dataflow_withKafka_running_iot](/assets/realtime-event-processing-with-hdf/lab1-kafka/dataflow_withKafka_running_iot.png)

> Dataflow generates data, filtering truck events from the dataflow and sending those events to kafka.

### 5.2 Verify PutKafka Published Message to Kafka

After a few seconds, stop the NiFi DataFlow using the **stop** button in the **actions** toolbar.
To verify that the PutKafka processor successfully published messages to the Kafka cluster, execute the following command to start a consumer to see the produced events:

~~~bash
[root@sandbox ~]# cd /usr/hdp/current/kafka-broker/bin/
[root@sandbox bin]# ./kafka-console-consumer.sh --zookeeper sandbox.hortonworks.com:2181 --topic truck_events --from-beginning
~~~

Your terminal should show that messages successfully published to Kafka:

![messages_published_toKafka](/assets/realtime-event-processing-with-hdf/lab1-kafka/messages_published_toKafka.png)


## Summary <a id="summary-lab1"></a>

This tutorial gave you brief glimpse of how to use Apache Kafka to capture real-time event data as a message and verify that Kafka successfully received those messages. In our next tutorial, you will create HBase and Hive tables to ingest real-time streaming data using Storm.


## Appendix A: Install Kafka <a id="install-kafka-lab1"></a>

Follow these steps if your version of the Sandbox does not have Kafka installed:


1\.  From the Ambari Dashboard, select Actions -> Add Service:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/add_service_kafka.png)

2\.  Select Kafka from the list of Services and click Next:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/addServiceWizard_kafka_service_iot.png)

3\.  Keep clicking Next with the selected defaults until you reach the following screen:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/log_dirs_kafka_broker_iot.png)

4\.  Set the value of logs.dir to  /tmp/kafka-logs

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/change_kafka_broker_log_dirs_iot.png)

5\.  Click the Deploy button:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/deploy_kafka_service_iot.png)

6\.  Wait for Kafka to install:

![](/assets/realtime-event-processing-with-hdf/lab1-kafka/wait_kafka_service_install_iot.png)


7\.  After Kafka is installed, you may be asked to restart some dependent Services. Please select the appropriate Services and click Restart.


## Further Reading <a id="further-reading-lab1"></a>
- [Apache Kafka Documentation](http://kafka.apache.org/)
- [Kafka Overview](http://hortonworks.com/hadoop/kafka/)
- [Apache Zookeeper Documentation](https://zookeeper.apache.org/)
