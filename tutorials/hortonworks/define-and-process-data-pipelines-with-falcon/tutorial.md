---
layout: tutorial
title: Define and Process Data Pipelines in Hadoop Using Apache Falcon
tutorial-id: 330
tutorial-series: Governance
tutorial-version: hdp-2.4.0
intro-page: true
components: [ falcon, ambari ]
---

## Introduction

Apache Falcon is a framework to simplify data pipeline processing and management on Hadoop clusters.

It makes it much simpler to onboard new workflows/pipelines, with support for late data handling and retry policies. It allows you to easily define relationship between various data and processing elements and integrate with metastore/catalog such as Hive/HCatalog. Finally it also lets you capture lineage information for feeds and processes.In this tutorial we are going walk the process of:

*   Defining the feeds and processes
*   Defining and executing a data pipeline to ingest, process and persist data continuously

## Prerequisite

- [Download Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
- You will need to refer to [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/) for logging into ambari as an admin user.
- Complete the [Creating Falcon Cluster tutorial](http://hortonworks.com/hadoop-tutorial/create-falcon-cluster/) to start the falcon service, prepare HDFS directories for Falcon cluster and to create Falcon cluster entities.

<!---Once you have download the Hortonworks sandbox and run the VM, navigate to the Ambari interface on the port `8080` of the IP address of your Sandbox VM. Login with the username of `admin` and the password as what you set it to when you changed it. You should have a similar image as below:

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-19%2016.28.48.png?dl=1)  -->

## Outline
- [Scenario](#scenario)
<!--- - [Start Falcon](#start-falcon)-->
- [Download and stage the dataset](#download-and-stage-the-dataset)
<!--- - [Create the cluster entities](#create-the-cluster-entities) -->
- [Define the rawEmailFeed entity using Wizard](#define-the-rawEmailFeed-entity-wizard)
- [Define the rawEmailFeed entity using XML](#define-the-rawEmailFeed-entity-XML)
- [Define the rawEmailIngestProcess entity using Wizard](#define-the-rawEmailIngestProcess-entity-wizard)
- [Define the rawEmailIngestProcess entity using XML](#define-the-rawEmailIngestProcess-entity-XML)
- [Define the cleansedEmailFeed using Wizard](#define-the-cleansedEmailFeed-wizard)
- [Define the cleansedEmailFeed using XML](#define-the-cleansedEmailFeed-XML)
- [Define the cleanseEmailProcess using Wizard](#define-the-cleansedEmailProcess-wizard)
- [Define the cleanseEmailProcess using XML](#define-the-cleansedEmailProcess-XML)
- [Run the feeds](#run-the-feeds)
- [Run the processes](#run-the-processes)
- [Input and Output of the pipeline](#input-and-output-of-the-pipeline)
- [Summary](#summary)

## Scenario <a id="scenario"></a>

In this tutorial, we will walk through a scenario where email data lands hourly on a cluster. In our example:

*   This cluster is the primary cluster located in the Oregon data center.
*   Data arrives from all the West Coast production servers. The input data feeds are often late for up to 4 hrs.

The goal is to clean the raw data to remove sensitive information like credit card numbers and make it available to our marketing data science team for customer churn analysis.

To simulate this scenario, we have a Pig script grabbing the freely available Enron emails from the internet and feeding it into the pipeline.

![](/assets/falcon-processing-pipelines/arch.png)  


<!---## Start Falcon <a id="start-falcon"></a>

By default, Falcon is not started on the sandbox. You can click on the Falcon icon on the left hand bar:

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-19%2016.29.22.png?dl=1)  


Then click on the `Service  Actions` button on the top right:

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-19%2016.29.44.png?dl=1)  


Then click on `Start`:

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-19%2016.30.07.png?dl=1)  


Once, Falcon starts, Ambari should clearly indicate as below that the service has started:

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-19%2016.34.32.png?dl=1)  -->


## Download and stage the dataset <a id="download-and-stage-the-dataset"></a>

Now let’s stage the dataset using the commandline. Although we perform many of these file operations below using the command line, you can also do the same with the `HDFS Files  View` in Ambari.

First, enter the shell with your preffered shell client. For this tutorial, we will SSH into Hortonworks Sandbox with the command:

~~~bash
ssh root@127.0.0.1 -p 2222;
~~~

![sshTerminal](/assets/falcon-processing-pipelines/sshTerminal.png)  

The default password is `hadoop`.

Then login as user `hdfs`.

~~~bash
su - hdfs
~~~

Then download the file falcon.zip with the following command"

~~~bash
wget http://hortonassets.s3.amazonaws.com/tutorial/falcon/falcon.zip
~~~

and then unzip with the command

~~~bash
unzip falcon.zip
~~~

![unzippingFalconFiles](/assets/falcon-processing-pipelines/unzippingFalconFiles.png)  


Now let’s give ourselves permission to upload files

~~~bash
hadoop fs -chmod -R 777 /user/ambari-qa
~~~

then let’s create a folder `falcon` under `ambari-qa` with the command

~~~bash
hadoop fs -mkdir /user/ambari-qa/falcon
~~~

![creatingDirectory](/assets/falcon-processing-pipelines/creatingDirectory.png)  


Now let’s upload the decompressed folder with the command

~~~bash
hadoop fs -copyFromLocal demo /user/ambari-qa/falcon/
~~~

![copyingDemoFile](/assets/falcon-processing-pipelines/copyingDemoFile.png)  

<!---
## Create the cluster entities <a id="create-the-cluster-entities"></a>

Before creating the cluster entities, we need to create the directories on HDFS representing the two clusters that we are going to define, namely `primaryCluster` and `backupCluster`.

Use `hadoop fs -mkdir` commands to create the directories `/apps/falcon/primaryCluster` and `/apps/falcon/backupCluster` directories on HDFS.

~~~bash
hadoop fs -mkdir /apps/falcon/primaryCluster
hadoop fs -mkdir /apps/falcon/backupCluster
~~~

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.29.58.png?dl=1)  


Further create directories called `staging` inside each of the directories we created above:

~~~bash
hadoop fs -mkdir /apps/falcon/primaryCluster/staging
hadoop fs -mkdir /apps/falcon/backupCluster/staging
~~~
![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.31.37.png?dl=1)  


Next we will need to create the `working` directories for `primaryCluster` and `backupCluster`

~~~bash
hadoop fs -mkdir /apps/falcon/primaryCluster/working
hadoop fs -mkdir /apps/falcon/backupCluster/working
~~~

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.36.12.png?dl=1)  


Finally you need to set the proper permissions on the staging/working directories:

~~~bash
hadoop fs -chmod 777 /apps/falcon/primaryCluster/staging
hadoop fs -chmod 755 /apps/falcon/primaryCluster/working
hadoop fs -chmod 777 /apps/falcon/backupCluster/staging
hadoop fs -chmod 755 /apps/falcon/backupCluster/working
~~~

Let’s open the Falcon Web UI. You can easily launch the Falcon Web UI from Ambari:

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-19%2016.31.12.png?dl=1)  


You can also navigate to the Falcon Web UI directly on our browser. The Falcon UI is by default at port 15000\. The default username is `ambari-qa`.

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.45.40.png?dl=1)  


This UI allows us to create and manage the various entities like Cluster, Feed, Process and Mirror. Each of these entities are represented by a XML file which you either directly upload or generate by filling up the various fields.

You can also search for existing entities and then edit, change state, etc.

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.46.23.png?dl=1)  


Let’s first create a couple of cluster entities. To create a cluster entity click on the `Cluster` button on the top.

A cluster entity defines the default access points for various resources on the cluster as well as default working directories to be used by Falcon jobs.

To define a cluster entity, we must specify a unique name by which we can identify the cluster.  In this tutorial, we use:

~~~
primaryCluster
~~~

Next enter a data center name or location of the cluster and a description for the cluster.  The data center name can be used by Falcon to improve performance of jobs that run locally or across data centers.

All entities defined in Falcon can be grouped and located using tags.  To clearly identify and locate entities, we assign the tag:

~~~
EntityType
~~~

We then need to specify the owner and permissions for the cluster.  

So we enter:

~~~
Owner:  ambari-qa
Group: users
Permissions: 755
~~~

With the value

~~~
Cluster
~~~

Next, we enter the URI for the various resources Falcon requires to manage data on the clusters.  These include the NameNode dfs.http.address, the NameNode IPC address used for Filesystem metadata operations,  the Yarn client IPC address used for executing jobs on Yarn, and the Oozie address used for running Falcon Feeds and Processes, and the Falcon messaging address.  The values we will use are the defaults for the Hortonworks Sandbox,  if you run this tutorial on your own test cluster, modify the addresses to match those defined in Ambari:

~~~
Readonly hftp://sandbox.hortonworks.com:50070
Write hdfs://sandbox.hortonworks.com:8020"
Execute sandbox.hortonworks.com:8050         
Workflow http://sandbox.hortonworks.com:11000/oozie/
Messaging tcp://sandbox.hortonworks.com:61616?daemon=true
~~~

The versions are not used and will be removed in the next version of the Falcon UI.

You can also override cluster properties for a specific cluster.  This can be useful for test or backup clusters which may have different physical configurations.  In this tutorial, we’ll just use the properties defined in Ambari.

After the resources are defined, you must define default staging, temporary and working directories for use by Falcon jobs based on the HDFS directories created earlier in the tutorial.  These can be overridden by specific jobs, but will be used in the event no directories are defined at the job level.  In the current version of the UI, these directories must exist, be owned by falcon, and have the proper permissions.

~~~
Staging  /apps/falcon/primaryCluster/staging
Temp /tmp         
Working /apps/falcon/primaryCluster/working
~~~

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.49.25.png?dl=1)

Once you have verified that the entities are the correct values, press `Next`.

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.50.01.png?dl=1)  


Click `Save` to persist the entity.

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.50.18.png?dl=1)  

You should receive a notification that the operation was successful.


Falcon jobs require a source and target cluster.  For some jobs, this may be the same cluster, for others, such as Mirroring and Disaster Recovery, the source and target clusters will be different.  Let’s go ahead and create a second cluster by creating a cluster with the name:

~~~
backupCluster
~~~

Reenter the same information you used above except for the directory information.  For the directories, use the backupCluster directories created earlier in the tutorial.

~~~
Staging  /apps/falcon/backupCluster/staging
Temp /tmp         
Working /apps/falcon/backupCluster/working
~~~



![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.51.14.png?dl=1)  


Click `Save` to persist the `backupCluster` entity.

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-07%2010.51.33.png?dl=1)  
-->

## Define the rawEmailFeed entity using Wizard <a id="define-the-rawEmailFeed-entity-wizard"></a>

To create a feed entity click on the `Feed` button on the top of the main page on the Falcon Web UI.

**NOTE : If you want to create it from XML, skip this section, and move on to the next one.**

Then enter the definition for the feed by giving the feed a unique name and a description.  For this tutorial we will use

~~~
rawEmailFeed
~~~

and

~~~
Raw customer email feed.
~~~

Let’s also enter a tag key and value so we can easily locate this Feed later:

~~~
externalSystem=USWestEmailServers
~~~

Feeds can be further categorised by identifying them with one or more groups.  In this demo, we will group all the Feeds together by defining the group:

~~~
churnAnalysisDataPipeline
~~~

We then set the ownership information for the Feed:

~~~
Owner:  ambari-qa
Group:  users
Permissions: 755
~~~

For the Schema Location and Provider, enter “/none”, then click Next.

![feed1](/assets/falcon-processing-pipelines/feed1.png)

On the Properties page, specify to run the job hourly by specifying the frequency as 1 hour, check Late Arrival Checkbox and specify the value as 1 hour. Change the timezone to UTC and click `Next`.

![feed2](/assets/falcon-processing-pipelines/feed2.png)

Enter the path of our data set on the Locations page:

~~~
/user/ambari-qa/falcon/demo/primary/input/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

We will set the stats and meta paths to / for now. Click `Next`.

![feed3](/assets/falcon-processing-pipelines/feed3.png)

On the Clusters page select the cluster you created, then enter today’s date and the current time for the validity start time, and enter an hour or two later for the end time.  The validity time specifies the period during which the feed will run.  For many feeds, validity time will be set to the time the feed is scheduled to go into production and the end time will be set into the far future.  Because we are running this tutorial on the Sandbox, we want to limit the time the process will run to conserve resources.

**DO NOT forget to provide the validity time in the timezone you have selected.**

Specify the retention period of 90 days, then click `Next`.

![feed4](/assets/falcon-processing-pipelines/feed4.png)

Save the feed.

![feed5](/assets/falcon-processing-pipelines/feed5.png)

<!---

Next we specify how often the job should run.

Let’s specify to run the job hourly by specifying the frequency as 1 hour.
Click Next to enter the path of our data set:

~~~
/user/ambari-qa/falcon/demo/primary/input/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

We will set the stats and meta path to `/` for now.


![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.09.14.png?dl=1)  


Once you have verified that these are the correct values press `Next`.


![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.14.21.png?dl=1)  


On the Clusters page enter today’s date and the current time for the validity start time and enter an hour or two later for the end time.  The validity time specifies the period during which the feed will run.  For many feeds, validity time will be set to the time the feed is scheduled to go into production and the end time will be set into the future. Because we are running this tutorial on the Sandbox, we want to limit the time the process will run to conserve resources.


Click `Next`

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.15.35.png?dl=1)  


Save the feed

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.16.01.png?dl=1)  
-->

## Define the rawEmailFeed entity using XML <a id="define-the-rawEmailFeed-entity-XML"></a>

Then click on the `Edit` button over XML Preview area on the right hand side of the screen and replace the XML content with the XML document below:

~~~
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<feed name="rawEmailFeed" description="Raw customer email feed" xmlns="uri:falcon:feed:0.1">
    <tags>externalSystem=USWestEmailServers</tags>
    <groups>churnAnalysisDataPipeline</groups>
    <frequency>hours(1)</frequency>
    <timezone>UTC</timezone>
    <late-arrival cut-off="hours(1)"/>
    <clusters>
        <cluster name="primaryCluster" type="source">
            <validity start="2016-06-05T05:00Z" end="2016-06-05T06:00Z"/>
            <retention limit="days(90)" action="delete"/>
        </cluster>
    </clusters>
    <locations>
        <location type="data" path="/user/ambari-qa/falcon/demo/primary/input/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}"/>
        <location type="stats" path="/"/>
        <location type="meta" path="/"/>
    </locations>
    <ACL owner="ambari-qa" group="users" permission="0x755"/>
    <schema location="/none" provider="/none"/>
</feed>
~~~

**NOTE : DO NOT copy the validity start and end time. Change it as per your time.**

Click `Finish` on the top of the XML Preview area.

![feedXML1](/assets/falcon-processing-pipelines/feedXML1.png)

Falcon UI should have automatically parsed out the values from the XML and populated in the right fields. Once you have verified that these are the correct values, save the feed.

## Define the rawEmailIngestProcess entity using Wizard <a id="define-the-rawEmailIngestProcess-entity-wizard"></a>

Now lets define the `rawEmailIngestProcess`.

To create a process entity click on the `Process` button on the top of the main page on the Falcon Web UI.

**NOTE : If you want to create it from XML, skip this section, and move on to the next one.**

Use the information below to create the process:

~~~
process name rawEmailIngestProcess
Tags email
With the value: testemail
~~~

And assign the workflow the name:

~~~
emailIngestWorkflow
~~~

Select Oozie as the execution engine and provide the following path:

~~~
/user/ambari-qa/falcon/demo/apps/ingest/fs
~~~
Accept the default values for the Access Control List and click Next.

![process1](/assets/falcon-processing-pipelines/process1.png)

On the Properties page, select the time zone, and set the frequency to 1 hour. Set the number of parallel processes to 1. This prevents a new instance from starting prior to the previous one completing. Specify the order as first-in, first-out (FIFO).
Select the exp-backoff retry policy, then set the attempts to 3 and the delay to 3 minutes. Click `Next`.

![process2](/assets/falcon-processing-pipelines/process2.png)

On the Clusters page, select primaryCluster again. As you did previously, modify the validity to a time slice which is in the very near future; set the validity to start now and end in an hour or two. Click `Next`.

**Note : Time should be specified as per the timezone selected.**

![process3](/assets/falcon-processing-pipelines/process3.png)

On the Inputs & Outputs page, ignore the Inputs section. For the output, enter a name and select the `rawEmailFeed` we created in the previous step and specify now(0,0) for the instance.

![process4](/assets/falcon-processing-pipelines/process4.png)

Lets save the process.

![process5](/assets/falcon-processing-pipelines/process5.png)

<!---
This job will run on the primaryCluster.
Again, set the validity to start now and end in an hour or two.

For the properties, set the number of parallel processes to 1, this prevents a new instance from starting prior to the previous one completing.

Specify the order as `first-in, First-out (FIFO)` and the Frequency to `1 hour`.

For inputs and output, enter the rawEmailFeed we created in the previous step and specify now(0,0) for the instance.  And assign the workflow the name:

~~~
emailIngestWorkflow
~~~

Select Oozie as the execution engine and provide the following path:

~~~
/user/ambari-qa/falcon/demo/apps/ingest/fs
~~~


![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.17.01.png?dl=1)  


Accept the default values and click next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.17.19.png?dl=1)  


On the Clusters page ensure you modify the **validity** by setting the end time to the next day as in the picture below and then click next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.18.02.png?dl=1)  


Accept the default values and click Next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.18.15.png?dl=1)  


Let’s `Save` the process.

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.18.37.png?dl=1)  
-->

## Define the rawEmailIngestProcess entity using XML <a id="define-the-rawEmailIngestProcess-entity-XML"></a>

Click on the `Edit` button over XML Preview area on the right hand side of the screen and replace the XML content with the XML document below:

~~~
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<process name="rawEmailIngestProcess" xmlns="uri:falcon:process:0.1">
    <tags>email=testemail</tags>
    <clusters>
        <cluster name="primaryCluster">
            <validity start="2016-06-05T05:00Z" end="2016-06-05T06:00Z"/>
        </cluster>
    </clusters>
    <parallel>1</parallel>
    <order>FIFO</order>
    <frequency>hours(1)</frequency>
    <timezone>UTC</timezone>
    <outputs>
        <output name="output" feed="rawEmailFeed" instance="now(0,0)"/>
    </outputs>
    <workflow name="emailIngestWorkflow" version="4.0.1" engine="oozie" path="/user/ambari-qa/falcon/demo/apps/ingest/fs"/>
    <retry policy="exp-backoff" delay="minutes(3)" attempts="3"/>
    <ACL owner="ambari-qa" group="users" permission="0x755"/>
</process>
~~~

**NOTE : DO NOT copy the validity start and end time. Change it as per your time..**

Click `Finish` on the top of the XML Preview area.

![processXML1](/assets/falcon-processing-pipelines/processXML1.png)

Verify all the values and click `Next`.

Save the process.

## Define the cleansedEmailFeed using Wizard <a id="define-the-cleansedEmailFeed-wizard"></a>

Again, to create a feed entity click on the `Feed` button on the top of the main page on the Falcon Web UI.

**NOTE : If you want to create it from XML, skip this section, and move on to the next one.**

Use the following information to create the feed:

~~~
name cleansedEmailFeed
description Cleansed customer emails
tag cleanse with value cleaned
Group churnAnalysisDataPipeline
~~~

We then set the ownership information for the Feed:

~~~
Owner:  ambari-qa
Group:  users
Permissions: 755
~~~

For the Schema Location and Provider, enter “/none”, then click `Next`.

![feed21](/assets/falcon-processing-pipelines/feed21.png)

On the Properties page, specify to run the job hourly by specifying the frequency as 1 hour, check Late Arrival Checkbox and specify the value as 4 hours. Change the timezone to UTC, accept the default values and click `Next`.

![feed22](/assets/falcon-processing-pipelines/feed22.png)

Set the default storage location to

~~~
/user/ambari-qa/falcon/demo/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

Keep stats path and meta path as /.
Click `Next`.

![feed23](/assets/falcon-processing-pipelines/feed23.png)

On the Clusters page,select source cluster as a primary cluster ensure you modify the validity to a time slice which is in the very near future. Provide time in current UTC. Specify the path for the data as:

~~~
/user/ambari-qa/falcon/demo/primary/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

And enter / for the stats and meta data locations.
Keep the retention period as 90 hours.

Set the target cluster as backupCluster and again set the validity start for the current time and end time to an hour or two from now
And specify the data path for the target to

~~~
/falcon/demo/bcp/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

Set the statistics and meta data locations to /
Enter the retention period as 90 hours for target cluster as well.
Click `Next`.

![feed24](/assets/falcon-processing-pipelines/feed24.png)

![feed25](/assets/falcon-processing-pipelines/feed25.png)

Verify all the entries you have filled out and then click `Save`.

![feed26](/assets/falcon-processing-pipelines/feed26.png)

## Define the cleansedEmailFeed using XML <a id="define-the-cleansedEmailFeed-XML"></a>

Click on the `Edit` button over XML Preview area on the right hand side of the screen and replace the XML content with the XML document below:

~~~
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<feed name="cleansedEmailFeed" description="Cleansed customer emails" xmlns="uri:falcon:feed:0.1">
    <tags>cleanse=cleaned</tags>
    <groups>churnAnalysisDataPipeline</groups>
    <frequency>hours(1)</frequency>
    <timezone>UTC</timezone>
    <late-arrival cut-off="hours(4)"/>
    <clusters>
        <cluster name="primaryCluster" type="source">
            <validity start="2016-06-05T05:00Z" end="2016-06-05T06:00Z"/>
            <retention limit="hours(90)" action="delete"/>
            <locations>
                <location type="data" path="/user/ambari-qa/falcon/demo/primary/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}"/>
                <location type="stats" path="/"/>
                <location type="meta" path="/"/>
            </locations>
        </cluster>
        <cluster name="backupCluster" type="target">
            <validity start="2016-06-05T05:00Z" end="2016-06-05T06:00Z"/>
            <retention limit="hours(90)" action="delete"/>
            <locations>
                <location type="data" path="/falcon/demo/bcp/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}"/>
                <location type="stats" path="/"/>
                <location type="meta" path="/"/>
            </locations>
        </cluster>
    </clusters>
    <locations>
        <location type="data" path="/user/ambari-qa/falcon/demo/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}"/>
        <location type="stats" path="/"/>
        <location type="meta" path="/"/>
    </locations>
    <ACL owner="ambari-qa" group="users" permission="0x755"/>
    <schema location="/none" provider="/none"/>
</feed>
~~~

**NOTE : DO NOT copy the validity start and end time. Change it as per your time.**

Click `Finish` on the top of the XML Preview area. Save the feed.

![feedXML2](/assets/falcon-processing-pipelines/feedXML2.png)


<!---
~~~
name cleansedEmailFeed"
description Cleansed customer emails"     
tag cleanse with value cleaned
Group churnAnalysisDataPipeline
Frequency 1 hour
~~~

We then set the ownership information for the Feed:

~~~
Owner:  ambari-qa
Group:  users
Permissions: 755
~~~

Set the default storage location to

~~~
/user/ambari-qa/falcon/demo/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}"
~~~

Select the primary cluster for the source and again set the validity start for the current time and end time to an hour or two from now.

Specify the path for the data as:

~~~
/user/ambari-qa/falcon/demo/primary/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

And enter `/` for the stats and meta data locations

Set the target cluster as backupCluster and again set the validity start for the current time and end time to an hour or two from now

And specify the data path for the target to

~~~
/falcon/demo/bcp/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

Set the statistics and meta data locations to `/`


![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.35.10.png?dl=1)  


Accept the default values and click Next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.35.49.png?dl=1)  


Accept the default values and click Next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.35.58.png?dl=1)  


On the Clusters page ensure you modify the validity to a time slice which is in the very near future and then click Next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.36.35.png?dl=1)  


![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.37.05.png?dl=1)  


Accept the default values and click Save

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.37.21.png?dl=1)  
-->


## Define the cleanseEmailProcess using Wizard <a id="define-the-cleansedEmailProcess-wizard"></a>

Now lets define the `cleanseEmailProcess`.
Again, to create a process entity click on the `Process` button on the top of the main page on the Falcon Web UI.

**NOTE : If you want to create it from XML, skip this section, and move on to the next one.**

Create this process with the following information

~~~
process name cleanseEmailProcess
~~~

Tag cleanse with the value yes

Then assign the workflow the name:

~~~
emailCleanseWorkflow
~~~

Select Pig as the execution engine and provide the following path:

~~~
/user/ambari-qa/falcon/demo/apps/pig/id.pig
~~~

Accept the default values for Access Control List and click `Next`.

![process21](/assets/falcon-processing-pipelines/process21.png)

For the properties, set the number of parallel processes to 1, this prevents a new instance from starting prior to the previous one completing.
Specify the order as first-in, First-out (FIFO)
And the Frequency to 1 hour.
Select the exp-backoff retry policy, then set the attempts to 3 and the delay to 3 minutes. Click `Next`.

![process22](/assets/falcon-processing-pipelines/process22.png)

On the Clusters page ensure you modify the validity to a time slice which is in the very near future and then click `Next`.

**NOTE : Time should be specified as per the timezone selected.**

![process23](/assets/falcon-processing-pipelines/process23.png)

Select the Input and Output Feeds as shown below and click Next.
Select rawEmailFeed as an input feed and cleansedEmailFeed as an output feed.

![process24](/assets/falcon-processing-pipelines/process24.png)

Verify all the values you have entered and then click `Save`.

![process25](/assets/falcon-processing-pipelines/process25.png)

![process26](/assets/falcon-processing-pipelines/process26.png)

<!---
We then set the ownership information:

~~~
Owner:  ambari-qa
Group:  users
Permissions: 755
~~~


This job will run on the primaryCluster.

Again, set the validity to start now and end in an hour or two.



For the properties, set the number of parallel processes to 1, this prevents a new instance from starting prior to the previous one completing.

Specify the order as first-in, First-out (FIFO)

And the Frequency to 1 hour.

For inputs and output, enter the rawEmailFeed we created in the previous step and specify it as input and now(0,0) for the instance.  

Add an output using `cleansedEmailFeed` and specify now(0,0) for the instance.  

Then assign the workflow the name:

~~~
emailCleanseWorkflow
~~~

Select Pig as the execution engine and provide the following path:

~~~
/user/ambari-qa/falcon/demo/apps/pig/id.pig
~~~



![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.39.34.png?dl=1)  


Accept the default values and click Next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.39.53.png?dl=1)  


On the Clusters page ensure you modify the validity to a time slice which is in the very near future and then click Next

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.40.24.png?dl=1)  


Select the Input and Output Feeds as shown below and Save

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.40.40.png?dl=1)  
-->

## Define the cleanseEmailProcess using XML <a id="define-the-cleansedEmailProcess-XML"></a>

Click on the `Edit` button over XML Preview area on the right hand side of the screen and replace the XML content with the XML document below:

~~~
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<process name="cleanseEmailProcess" xmlns="uri:falcon:process:0.1">
    <tags>cleanse=yes</tags>
    <clusters>
        <cluster name="primaryCluster">
            <validity start="2016-06-05T05:00Z" end="2016-06-05T06:00Z"/>
        </cluster>
    </clusters>
    <parallel>1</parallel>
    <order>FIFO</order>
    <frequency>hours(1)</frequency>
    <timezone>UTC</timezone>
    <inputs>
        <input name="input" feed="rawEmailFeed" start="now(0,0)" end="now(0,0)"/>
    </inputs>
    <outputs>
        <output name="output" feed="cleansedEmailFeed" instance="now(0,0)"/>
    </outputs>
    <workflow name="emailCleanseWorkflow" version="pig-0.13.0" engine="pig" path="/user/ambari-qa/falcon/demo/apps/pig/id.pig"/>
    <retry policy="exp-backoff" delay="minutes(3)" attempts="3"/>
    <ACL owner="ambari-qa" group="users" permission="0x755"/>
</process>
~~~

**NOTE : DO NOT copy the validity start and end time. Change it as per your time.**

Click `Finish` on the top of the XML Preview area, verify all the values and save the process.

![processXML2](/assets/falcon-processing-pipelines/processXML2.png)

## Run the feeds <a id="run-the-feeds"></a>

From the Falcon Web UI home page search for the Feeds we created

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.41.34.png?dl=1)  


Select the rawEmailFeed by clicking on the checkbox

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.41.56.png?dl=1)  


Then click on the Schedule button on the top of the search results

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.42.04.png?dl=1)  


Next run the `cleansedEmailFeed` in the same way

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.42.30.png?dl=1)  


## Run the processes <a id="run-the-processes"></a>

From the Falcon Web UI home page search for the Process we created

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.42.55.png?dl=1)  


Select the `cleanseEmailProcess` by clicking on the checkbox

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.43.07.png?dl=1)  


Then click on the Schedule button on the top of the search results

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.43.31.png?dl=1)  


Next run the `rawEmailIngestProcess` in the same way

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.43.41.png?dl=1)  


If you visit the Oozie process page, you can seen the processes running

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.44.23.png?dl=1)  


## Input and Output of the pipeline <a id="input-and-output-of-the-pipeline"></a>

Now that the feeds and processes are running, we can check the dataset being ingressed and the dataset egressed on HDFS.

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2015.45.48.png?dl=1)  


Here is the data being ingressed

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2016.31.37.png?dl=1)  


and here is the data being egressed from the pipeline

![](/assets/falcon-processing-pipelines/Screenshot%202015-08-11%2017.13.05.png?dl=1)  


## Summary <a id="summary"></a>

In this tutorial we walked through a scenario to clean the raw data to remove sensitive information like credit card numbers and make it available to our marketing data science team for customer churn analysis by defining a data pipeline with Apache Falcon.
