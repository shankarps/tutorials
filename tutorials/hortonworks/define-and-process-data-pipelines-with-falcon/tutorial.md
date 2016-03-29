## Introduction

Apache Falcon is a framework to simplify data pipeline processing and management on Hadoop clusters.

It makes it much simpler to onboard new workflows/pipelines, with support for late data handling and retry policies. It allows you to easily define relationship between various data and processing elements and integrate with metastore/catalog such as Hive/HCatalog. Finally it also lets you capture lineage information for feeds and processes.In this tutorial we are going walk the process of:

*   Defining the feeds and processes
*   Defining and executing a job to mirror data between two clusters
*   Defining and executing a data pipeline to ingest, process and persist data continuously

## Prerequisite

- [Download Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
- You will need to refer to [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/) for logging into ambari as an admin user.

Once you have download the Hortonworks sandbox and run the VM, navigate to the Ambari interface on the port `8080` of the IP address of your Sandbox VM. Login with the username of `admin` and the password as what you set it to when you changed it. You should have a similar image as below:

![](/assets/falcon-processing-pipelines/amabari_dashboard_falcon_processing.png)  

## Outline
- [Scenario](#scenario)
- [Start Falcon](#start-falcon)
- [Download and stage the dataset](#download-and-stage-the-dataset)
- [Create the cluster entities](#create-the-cluster-entities)
- [Define the rawEmailFeed entity](#define-the-rawEmailFeed-entity)
- [Define the rawEmailIngestProcess entity](#define-the-rawEmailIngestProcess-entity)
- [Define the cleansedEmailFeed](#define-the-cleansedEmailFeed)
- [Define the cleansedEmailProcess](#define-cleansedemailprocess)
- [Run the feeds](#run-the-feeds)
- [Run the processes](#run-the-processes)
- [Input and Output of the pipeline](#input-and-output-of-the-pipeline)
- [Summary](#summary)

## Scenario <a id="scenario"></a>

In this tutorial, we will walk through a scenario where email data lands hourly on a cluster. In our example:

*   This cluster is the primary cluster located in the Oregon data center.
*   Data arrives from all the West Coast production servers. The input data feeds are often late for up to 4 hrs.

The goal is to clean the raw data to remove sensitive information like credit card numbers and make it available to our marketing data science team for customer churn analysis.

To simulate this scenario, we have a pig script grabbing the freely available Enron emails from the internet and feeding it into the pipeline.

![](/assets/falcon-processing-pipelines/arch.png)  


## Start Falcon <a id="start-falcon"></a>

By default, Falcon is not started on the sandbox. You can click on the Falcon icon on the left hand bar:

![](/assets/falcon-processing-pipelines/falcon_icon_ambari.png)  


Then click on the `Service  Actions` button on the top right:

![](/assets/falcon-processing-pipelines/start_falcon_ambari.png)  


Then click on `Start`:

![](/assets/falcon-processing-pipelines/start_falcon_service.png)  


Once, Falcon starts, Ambari should clearly indicate as below that the service has started:

![](/assets/falcon-processing-pipelines/falcon_service_started.png)  


## Download and stage the dataset <a id="download-and-stage-the-dataset"></a>

Now let’s stage the dataset using the command line. Although we use the command line, we can also do the same file operations with `HDFS Files` view in Ambari.

First, access your Hortonworks Sandbox through your preferred shell client. We'll use SSH in this tutorial. Use the following command:

~~~bash
ssh root@127.0.0.1 -p 2222
~~~

![](/assets/falcon-processing-pipelines/ssh_into_sandbox_falcon.png)  

Then login as user falcon

~~~bash
su falcon
~~~

Then change to a directory you can write in:

~~~bash
cd ~
~~~

Then download the file [falcon.zip](http://hortonassets.s3.amazonaws.com/tutorial/falcon/falcon.zip) with the command:

~~~bash
wget http://hortonassets.s3.amazonaws.com/tutorial/falcon/falcon.zip
~~~

![](/assets/falcon-processing-pipelines/writeable_directory_save_falconzip.png)

and then unzip the file.

~~~bash
unzip falcon.zip
~~~

![](/assets/falcon-processing-pipelines/unzip_falcon_zip.png)

First let's switch to user hdfs to change the file permissions.

~~~bash
exit
su hdfs
~~~

Now let’s give ourselves permission to upload files:

~~~bash
hadoop fs -chmod -R 777 /user/ambari-qa
~~~

![](/assets/falcon-processing-pipelines/change_ambari_qa_permissions_terminal.png)

Let's login back in as falcon user

~~~bash
exit
su falcon
~~~

then let’s create a folder falcon under ambari-qa with the command

~~~bash
hadoop fs -mkdir /user/ambari-qa/falcon
~~~

![](/assets/falcon-processing-pipelines/create_falcon_folder_falcon_user.png)


Now let’s upload the decompressed folder to the newly created falcon folder:

~~~bash
cd ~
hadoop fs -copyFromLocal demo /user/ambari-qa/falcon/
~~~


![](/assets/falcon-processing-pipelines/copy_demo_to_falcon_folder.png)  


## Create the cluster entities <a id="create-the-cluster-entities"></a>

Before creating the cluster entities, we need to create the directories on HDFS representing the two clusters that we are going to define, namely `primaryCluster` and `backupCluster`.

After navigating through the directory path `/apps/falcon/`, use the following command to create the directories `/primaryCluster` and `/backupCluster` on HDFS.

~~~bash
hadoop fs -mkdir /apps/falcon/primaryCluster
hadoop fs -mkdir /apps/falcon/backupCluster
~~~

![](/assets/falcon-processing-pipelines/cluster_dir_for_falcon_terminal.png)  

To verify the directories were created, open `HDFS Files` view from Ambari and navigate to
`/apps/falcon`, you should see the two new folders you just created.

![](/assets/falcon-processing-pipelines/falcon_folders_created_in_falcon.png)
> Notice: the owner is falcon

Further create directories called `staging` inside each of the directories we created above:

~~~bash
hadoop fs -mkdir /apps/falcon/primaryCluster/staging
hadoop fs -mkdir /apps/falcon/backupCluster/staging
~~~

![](/assets/falcon-processing-pipelines/staging_dir_cluster_folders.png)  


Next we will need to create the `working` directories for `primaryCluster` and `backupCluster`

~~~bash
hadoop fs -mkdir /apps/falcon/primaryCluster/working
hadoop fs -mkdir /apps/falcon/backupCluster/working
~~~

![](/assets/falcon-processing-pipelines/working_dir_cluster_folders.png)  

To verify that the directories were created, click on `backupCluster` and `primaryCluster`,
you should see your newly created directories.

Finally you need to set the proper permissions on the staging/working directories. Below example
is how you would set permissions using command line:

~~~bash
hadoop fs -chmod 777 /apps/falcon/primaryCluster/staging
hadoop fs -chmod 755 /apps/falcon/primaryCluster/working
hadoop fs -chmod 777 /apps/falcon/backupCluster/staging
hadoop fs -chmod 755 /apps/falcon/backupCluster/working
~~~

![](/assets/falcon-processing-pipelines/set_permissions_terminal_falcon.png)

Let’s open the Falcon Web UI. You can easily launch the Falcon Web UI from Ambari:

![](/assets/falcon-processing-pipelines/web_falcon_ui_ambari.png)  


You can also navigate to the Falcon Web UI directly on our browser. The Falcon UI is by default at the url `http://127.0.0.1:15000/#/`. The default username is `ambari-qa`.

![](/assets/falcon-processing-pipelines/falcon_login_ui.png)  


This UI allows us to create and manage the various entities like Cluster, Feed, Process and Mirror. Each of these entities are represented by a XML file which you either directly upload or generate by filling up the various fields.

You can also search for existing entities and then edit, change state, etc.

![](/assets/falcon-processing-pipelines/search_edit_change_state_falcon.png)  


Let’s first create a couple of cluster entities. To create a cluster entity click on the `Cluster` button on the top.

A cluster entity defines the default access points for various resources on the cluster as well as default working directories to be used by Falcon jobs.

To define a cluster entity, we must specify a unique name by which we can identify the cluster.  In this tutorial, we use:

~~~
primaryCluster
~~~

Next enter a data center name, location or `colo` of the cluster and a `description` for the cluster.  The data center name can be used by Falcon to improve performance of jobs that run locally or across data centers.

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

![](/assets/falcon-processing-pipelines/new_cluster_falcon.png)


Once you have verified that the entities are the correct values, press `Next`. Click `Save` to persist the entity.

![](/assets/falcon-processing-pipelines/save_cluster_falcon.png)  

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



![](/assets/falcon-processing-pipelines/backupcluster_new_cluster_falcon.png)  


Once you have verified that the entities are the correct values, press `Next`. Click `Save` to persist the entity. Click `Save` to persist the `backupCluster` entity.

![](/assets/falcon-processing-pipelines/save_backupcluster_falcon.png)  


## Define the rawEmailFeed entity <a id="define-the-rawEmailFeed-entity"></a>

To create a feed entity click on the `Feed` button on the top of the main page on the Falcon Web UI.

Then enter the definition for the feed by giving the feed a unique name and a description.  For this tutorial we will use

~~~
rawEmailFeed
~~~

and

~~~
Raw customer email feed.
~~~

Let’s also enter a tag, so we can easily locate this Feed later:

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

Under Schema, set the Location and Provider fields to

~~~
/none
~~~

![](/assets/falcon-processing-pipelines/newfeed_falcon.png)

Next we specify how often the job should run.

Let’s specify to run the job hourly by specifying the frequency as 1 hour and late arrival as up to 1 hour.

![](/assets/falcon-processing-pipelines/properties_newfeed_falcon.png)

Click Next to enter the path of our data set:

~~~
/user/ambari-qa/falcon/demo/primary/input/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

We will set the stats and meta path to `/` for now.
Once you have verified that these are the correct values press `Next`.

![](/assets/falcon-processing-pipelines/data_path_falcon_newfeed.png)  

On the Clusters page enter today’s date and the current time for the validity start time and enter an hour or two later for the end time.  The validity time specifies the period during which the feed will run.  For many feeds, validity time will be set to the time the feed is scheduled to go into production and the end time will be set into the future. Because we are running this tutorial on the Sandbox, we want to limit the time the process will run to conserve resources.


Click `Next`

![](/assets/falcon-processing-pipelines/cluster_page_falcon.png)  


Save the feed

![](/assets/falcon-processing-pipelines/save_thefeed_falcon.png)  


## Define the rawEmailIngestProcess entity <a id="efine-the-rawEmailIngestProcess-entity"></a>

Now lets define the `rawEmailIngestProcess`.

To create a process entity click on the `Process` button on the top of the main page on the Falcon Web UI.

Use the information below to create the process:

~~~
process name: rawEmailIngestProcess
Tags: email
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


![](/assets/falcon-processing-pipelines/newprocess_falcon.png)  

Accept the default values and click next

For the properties, set the number of maximum parallel instances to `1`, this prevents a new instance from starting prior to the previous one completing.

Specify the order as `first-in, First-out (FIFO)` and the Frequency to `1 hour`.

For retry, set the policy to `exp-backoff`, attempts to `3` and delay up to `3 minutes`.

![](/assets/falcon-processing-pipelines/properties_page_newprocess_falcon.png)  


On the Clusters Page, this job will run on the `primaryCluster`.
Again, set the validity to start now and end in an `hour or two`.

![](/assets/falcon-processing-pipelines/clusters_page_newprocess_falcon.png)

For inputs and output section, the start and end times (instances) control when the first instance of the job should be initiated. The end time is the date when the user no longer wants the job to execute. For the purpose of the tutorial, set the end time to a couple of hours from now to prevent the process from running continually and filling up the disk.

 Let's enter the rawEmailFeed we created in the previous step. For the Inputs Section, specify the start as now(0,0) and the end as now(0,0) for the instances. In the Output Section, specify now(0,0) for the instance.

For example:

![](/assets/falcon-processing-pipelines/output_input_newprocess_falcon.png)  


Let’s `Save` the process.

![](/assets/falcon-processing-pipelines/save_theprocess_falcon.png)  


## Define the cleansedEmailFeed <a id="define-the-cleansedEmailFeed"></a>

Again, to create a feed entity click on the `Feed` button on the top of the main page on the Falcon Web UI.

Use the following information to create the feed:

~~~
name: cleansedEmailFeed
description: Cleansed customer emails   
tag: key{cleanse}; value{cleaned}
Group churnAnalysisDataPipeline
Frequency 1 hour
~~~


We then set the ownership information for the Feed:

~~~
Owner:  ambari-qa
Group:  users
Permissions: 755
~~~

![](/assets/falcon-processing-pipelines/newfeed_cleansedemailfeed_falcon.png)


For the properties page, enter the following information:

~~~
Frequency: 1 hour
Late Arrival: 4 hours
Timezone: (area of your timezone)
~~~


![](/assets/falcon-processing-pipelines/properties_newfeed_cleansed_falcon.png)

Set the default storage location to

~~~
/user/ambari-qa/falcon/demo/processed/enron/${YEAR}-${MONTH}-${DAY}-${HOUR}
~~~

![](/assets/falcon-processing-pipelines/newfeed_storage_location.png)

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


![](/assets/falcon-processing-pipelines/source_target_cluster_values_process_falcon.png)   


Accept the default values and click Save

![](/assets/falcon-processing-pipelines/save_thefeed_cleansedemailfeed.png)  


## Define the cleanseEmailProcess <a id="define-cleansedemailprocess"></a>

Now lets define the `cleanseEmailProcess`.
Again, to create a process entity click on the `Process` button on the top of the main page on the Falcon Web UI.

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


We then set the ownership information:

~~~
Owner:  ambari-qa
Group:  users
Permissions: 755
~~~

![](/assets/falcon-processing-pipelines/process_cleanseEmailProcess_falcon.png)


~~~
Frequency: 1 hour
Maximum Parallel Instances: 1
Order: FIFO
Policy: exp-backoff
Attemps: 3
Delay: 5 minutes
~~~

![](/assets/falcon-processing-pipelines/properties_cleansedemailprocess_falcon.png)

This job will run on the primaryCluster.

Again, set the validity to start now and end in an hour or two.

![](/assets/falcon-processing-pipelines/cluster_cleansedemailprocess_falcon.png)

For inputs, enter the `rawEmailFeed` we created in the previous step and specify it as input and now(0,0) for the start/end instance.  

Add an output using `cleansedEmailFeed` and specify now(0,0) for the instance.  

![](/assets/falcon-processing-pipelines/process_input_output_cleansedemailprocess_falcon.png)  


Select the Input and Output Feeds as shown below and Save

![](/assets/falcon-processing-pipelines/save_cleansedEmailProcess_falcon.png)  


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
