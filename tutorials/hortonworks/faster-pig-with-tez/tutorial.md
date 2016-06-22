---
layout: tutorial
title: Faster Pig with Tez
tutorial-id: 280
tutorial-series: Basic Development
tutorial-version: hdp-2.4.0
intro-page: true
components: [ pig, tez, ambari ]
---


# Faster Pig With Tez

### Introduction

In this tutorial, you will explore the difference between running pig with execution engine of MapReduce and Tez. By the end of the tutorial, you will know advantage of using Tez over MapReduce.

## Pre-Requisite
*  Downloaded and Installed latest [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
*  [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/)
*  Allow yourself around one hour to complete this tutorial

## Outline
- [What is Pig?](#what-is-pig)
- [What is Tez?](#what-is-tez)
- [Our Data Processing Task](#our-data-processing-task)
- [Step 1: Download the data](#download-the-data)
- [Step 2: Upload Data into HDFS](#upload-data-into-hdfs)
- [Step 3: Run Pig on MapReduce Using Ambari Pig UI](#use-ambari-pig-ui-run-pig-on-mapreduce)
- [Step 4: Run Pig on Tez Using Ambari Pig UI](#run-pig-on-tez)
- [Further Reading](#further-reading)

## What is Pig? <a id="what-is-pig"></a>

Pig is a high level scripting language that is used with Apache Hadoop. Pig excels at describing data analysis problems as data flows. Pig is complete in that you can do all the required data manipulations in Apache Hadoop with Pig. In addition through the User Defined Functions(UDF) facility in Pig you can have Pig invoke code in many languages like JRuby, Jython and Java. Conversely you can execute Pig scripts in other languages. The result is that you can use Pig as a component to build larger and more complex applications that tackle real business problems.

A good example of a Pig application is the ETL transaction model that describes how a process will extract data from a source, transform it according to a rule set and then load it into a datastore. Pig can ingest data from files, streams or other sources using the User Defined Functions(UDF). Once it has the data it can perform select, iteration, and other transforms over the data. Again the UDF feature allows passing the data to more complex algorithms for the transform. Finally Pig can store the results into the Hadoop Data File System.

Pig scripts are translated into a series of MapReduce jobs or a Tez DAG that are run on the Apache Hadoop cluster. As part of the translation the Pig interpreter does perform optimizations to speed execution on Apache Hadoop. We are going to write a Pig script that will do our data analysis task.

## What is Tez? <a id="what-is-tez"></a>

Tez – Hindi for “speed” provides a general-purpose, highly customizable framework that creates simplifies data-processing tasks across both small scale (low-latency) and large-scale (high throughput) workloads in Hadoop. It generalizes the [MapReduce paradigm](http://en.wikipedia.org/wiki/MapReduce) to a more powerful framework by providing the ability to execute a complex DAG ([directed acyclic graph](http://en.wikipedia.org/wiki/Directed_acyclic_graph)) of tasks for a single job so that projects in the Apache Hadoop ecosystem such as Apache Hive, Apache Pig and Cascading can meet requirements for human-interactive response times and extreme throughput at petabyte scale (clearly MapReduce has been a key driver in achieving this).

## Our data processing task <a id="our-data-processing-task"></a>

We are going to read in a baseball statistics file. We are going to compute the highest runs by a player for each year. This file has all the statistics from 1871-2011 and it contains over 90,000 rows. Once we have the highest runs we will extend the script to translate a player id field into the first and last names of the players.

### Step 1: Download the data <a id="download-the-data"></a>

Download the data file from [lahman591-csv.zip](http://seanlahman.com/files/database/lahman591-csv.zip). After the file is downloaded, extract it.

### Step 2: Upload Data into HDFS <a id="upload-data-into-hdfs"></a>

Let's login to ambari under `maria_dev` using the following credentials: `maria_dev/maria_dev`. Once the ambari dasbhoard screen appears, go to **HDFS Files View** as shown below:

![Ambari_HDFS_Files_View](/assets/faster-pig-with-tez/Ambari_HDFS_Files_View_pig_tez.png)

Navigate to `/user/maria_dev` path. Then upload `Batting.csv` file using the upload button:

![Upload_batting_csv_maria_dev_dir](/assets/faster-pig-with-tez/Upload_batting_csv_maria_dev_dir_pig_tez.png)


The data file will appear in the maria_dev directory as shown below:

![maria_dev_batting_csv_uploaded](/assets/faster-pig-with-tez/maria_dev_batting_csv_uploaded_pig_tez.png)


### Step 3: Run Pig on MapReduce Using Ambari Pig UI <a id="use-ambari-pig-ui-run-pig-on-mapreduce"></a>

We will first run Pig without Tez. Click on Pig Views:

![pig_views](/assets/faster-pig-with-tez/pig_views_pig_tez.png)


Click on **New Script** and name it `pig-mapreduce` or a name of your choice. Then press **Create**:

![created_new_script_pig_mapreduce](/assets/faster-pig-with-tez/created_new_script_pig_mapreduce_pig_tez.png)


Let's create our first pig script. Copy the pig script below and paste it in Ambari's Pig Script editor:

~~~
    batting = LOAD '/user/maria_dev/Batting.csv' USING PigStorage(',');
    raw_runs = FILTER batting BY $1>0;
    runs = FOREACH raw_runs GENERATE $0 AS playerID, $1 AS year, $8 AS runs;
    grp_data = GROUP runs BY (year);
    max_runs = FOREACH grp_data GENERATE group as grp, MAX(runs.runs) AS max_runs;
    join_max_runs = JOIN max_runs BY ($0, max_runs), runs BY (year, runs);
    join_data = FOREACH join_max_runs GENERATE $0 AS year, $2 AS playerID, $1 AS runs;
    DUMP join_data;
~~~

Then hit the **Save** button on the left of the editor. 

![save_pig_mapreduce_script](/assets/faster-pig-with-tez/save_pig_mapreduce_script.png)


Now we can execute the pig script using the MapReduce engine by simply clicking the blue **Execute** button:

![execute_pig_mapreduce_script](/assets/faster-pig-with-tez/execute_pig_mapreduce_script_pig_tez.png)


After the script executes, we can view **Results**, **Logs** and **Script Details**. 

![results_logs_pig_mapreduce_script](/assets/faster-pig-with-tez/results_logs_pig_mapreduce_script_pig_tez.png)

> Notice, in the **Logs**, the script typically takes a little more than one minutes to finish on our single node pseudocluster.

### Step 4: Run Pig on Tez Using Ambari Pig UI <a id="run-pig-on-tez"></a>

Let's run the same Pig script with Tez by clicking on **Execute on Tez** button near the Execute button:

![execute_on_tez_pig_tez_script](/assets/faster-pig-with-tez/execute_on_tez_pig_tez_script_pig_tez.png)


Notice the time after the execution completes:

![log_time_run_pig_script](/assets/faster-pig-with-tez/log_time_run_pig_script_pig_tez.png)

On our machine it took around 24 seconds with Pig using the Tez engine. That is more than 2X faster than Pig using MapReduce even without any specific optimization in the script for Tez.

Tez definitely lives up to it's name.

## Further Reading <a id="further-reading"></a>
- [Apache Pig](http://hortonworks.com/hadoop/pig/)
- [Pig Latin Basics](https://pig.apache.org/docs/r0.12.0/basic.html#store)
- [Apache Tez](http://hortonworks.com/hadoop/tez/)
- [Apache MapReduce](http://hortonworks.com/hadoop/mapreduce/)
