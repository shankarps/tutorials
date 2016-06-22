---
layout: tutorial
title: Ingest, Route and Land Real Time Events with Apache NiFi
tutorial-id: 220
tutorial-series: Streaming
tutorial-version: hdp-2.4.0
intro-page: false
components: [ nifi ]
---

# Lab0: Ingest, Route and Land Real Time Events with Apache NiFi

## Introduction

Apache NiFi can collect and transport data from numerous sources and provide interactive command and control of live flows with full and automated data provenance. We will install NiFi onto our Hortonworks Sandbox and become familiar with the NiFi Web Interface. We will create a flow of data using Hortonworks DataFlow to activate the truck stream simulator to generate truck data, remove the log data, extract the live truck events data and store the events into a file. We will use a file to verify that the correct data is being inserted into the file.

## Pre-Requisites
- Downloaded and Installed latest [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
- If you are new to the sandbox shell, refer to [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/)
- Memory must be at least 8GB RAM, preferably 4 processor cores, else errors may occur in third tutorial
- For windows users, to run linux terminal commands in these tutorials, download [Git Bash](https://openhatch.org/missions/windows-setup/install-git-bash).
- Read Table 1 to figure out some basic details about your sandbox

**Table 1: Virtual Machine Information** <a id="table1-virtual-machine-information"></a>

| Parameter  | Value (VirtualBox)  | Value(VMware)  | Value(MS Azure)  |
|---|---|---|---|
| Host Name  | 127.0.0.1  | 172.16.110.129  | 23.99.9.233  |
| Port  | 2222  | 2222  | 22  |
| Terminal Username  | root  | root  | {username-of-azure}  |
| Terminal Password  | hadoop  | hadoop  | {password-of-azure}  |

> Note: **Host Name** values are unique for VMware & Azure Sandbox compared to the table. For VMware and VirtualBox, **Host Name** is located on welcome screen. For Azure, **Host Name** is located under **Public IP Address** on Sandbox Dashboard. For Azure users, the terminal **username** and **password** is one you created while deploying the sandbox on azure. For VMware and VirtualBox users, terminal password changes after first login.

- Added `sandbox.hortonworks.com` to your `/private/etc/hosts` file (mac and linux users)
- Added `sandbox.hortonworks.com` to your `/c/Windows/System32/Drivers/etc/hosts` file (windows 7 users)

The following terminal commands in the tutorial instructions are performed in VirtualBox Sandbox and Mac machine. For windows users, to run the following terminal commands, download [Git Bash](https://openhatch.org/missions/windows-setup/install-git-bash).

If on mac or linux, to add `sandbox.hortonworks.com` to your list of hosts, open the terminal, enter the following command, replace {Host-Name} with the appropriate host for your sandbox:

~~~bash
echo '{Host-Name} sandbox.hortonworks.com' | sudo tee -a /private/etc/hosts
~~~

If on windows 7, to add `sandbox.hortonworks.com` to your list of hosts, open git bash, enter the following command, replace {Host-Name} with the appropriate host for your sandbox:

~~~bash
echo '{Host-Name} sandbox.hortonworks.com' | tee -a /c/Windows/System32/Drivers/etc/hosts
~~~

![changing-hosts-file.png](/assets/realtime-event-processing-with-hdf/lab0-nifi/changing-hosts-file.png)


## Outline
- [Stream simulator](#stream-simulator-lab0)
- [Step 1: Run Simulator By shell](#step1-run-simulator-shell-lab0)
- [Apache NiFi](#apache-nifi-lab0)
- [Step 2: Install NiFi](#step2-install-nifi-lab0)
- [Step 3: Start NiFi](#step3-start-nifi)
- [Step 4: Explore NiFi Web Interface](#step4-explore-nifi-interface-lab0)
- [Step 5: Understand NiFi DataFlow Build Process](#step5-create-nifi-dataflow-lab0)
- [Step 6: Build Stream Simulator DataFlow Section](#step6-build-stream-simulator-dataflow-lab0)
- [Step 7: Build Filter Logs & Enrich TruckEvents DataFlow Section](#step7-build-filter-logs-enrich-truckevents-lab0)
- [Step 8: Run NiFi DataFlow](#run-nifi-dataflow-lab0)
- [Summary](#summary-lab0)
- [Further Reading](#further-reading-lab0)

## Tutorial Overview
- Understand the Stream Simulator
- Run Stream Simulator From Terminal
- Install and Start NiFi
- Create NiFi DataFlow to Generate and Store Truck events

## Stream Simulator <a id="stream-simulator-lab0"></a>

The stream simulator is a lightweight framework that generates truck event data. The simulator uses [New York City Truck Routes (kml)](http://www.nyc.gov/html/dot/downloads/misc/all_truck_routes_nyc.kml) which defines driver road paths with Latitude and Longitude information.

The simulator uses **[Akka](http://akka.io/)** to simplify concurrency, messaging and inheritance. It has two **[Plain Old Java Objects (POJOS)](https://en.wikipedia.org/wiki/Plain_Old_Java_Object)**, one for Trucks and another for Drivers that generate the events. Consequently, the **AbstractEventEmitter** class becomes extended while the **onReceive** method generates events, creates new **[Actors](http://doc.akka.io/docs/akka/snapshot/java/untyped-actors.html)**, sends messages to Actors and delivers those events to an **EventCollector** class. This class’s purpose is to collect events generated from the domain objects and print them to standard output. Let’s run the simulator through the terminal and see the events that are generated.

### Step 1: Run the Simulator By shell <a id="step1-run-simulator-shell-lab0"></a>

Before we run the simulator, we need to perform some shell operations to install and download the stream simulator. The following terminal commands were performed in VirtualBox Sandbox and Mac machine. For VMware and Azure users, refer to [Table 1](#table1-virtual-machine-information) in the pre-requisites section to run ssh command. For windows users, to run the following terminal commands, download [Git Bash](https://openhatch.org/missions/windows-setup/install-git-bash). The other terminal commands should be the same for VirtualBox, VMware and Azure.

### 1.1 Download & Setup The Simulator

1\. Open the terminal, if using VirtualBox copy & paste the following command to access the sandbox through the shell.

~~~
ssh root@127.0.0.1 -p 2222
~~~

For VMware and Azure users, insert the appropriate values for username, hostname and port into the SSH Definition:

~~~
ssh <username>@<hostname> -p <port>
~~~

> Note: For VMware and Azure users, the hostname is different than VirtualBox and the **hostname** can be found on the welcome screen. Refer to Table 1 for **port** number. Username for VMware and Azure is same as VirtualBox. If you need help, refer to [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/).

2\. Clone the github repo to download the truck event simulator.

~~~
cd ~
git clone https://github.com/james94/iot-truck-streaming
~~~

3\. Install Apache Maven, so we can compile the stream simulator code and run the simulator. Run the following commands:

~~~
wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo

yum install apache-maven
~~~

> Note: You will be prompted to allow maven to install, type 'y' for yes

After your maven package installs, you should obtain the message: Complete!

4\. Navigate to iot-truck-streaming directory.

~~~
cd iot-truck-streaming
~~~

5\. Since we are at the base of our project, let’s export our demo configurations:

~~~
sudo mkdir /etc/storm_demo
sudo cp config.properties /etc/storm_demo
sudo cp -R storm-demo-webapp/routes/ /etc/storm_demo
~~~

6\. For maven to run, it needs to detect the pom.xml file. Rename pom24.xml to pom.xml, copy/paste the commands:

~~~
mv -f storm-streaming/pom24.xml storm-streaming/pom.xml

mvn clean package
~~~

Apache Maven command: mvn clean deletes everying in the target folder. The storm-streaming contains a target folder that is impacted. The package phase of the command compiles the code and packages it into jar files according to the pom file.

> Note: packaging may take around 9 minutes.

### 1.2 Run The Simulator

1\. To test the simulator, run `generate.sh` script. Use the commands:

~~~
cd stream-simulator
chmod 750 *.sh
./generate.sh
~~~

> Note: press **ctrl+c** stop the simulator

You should see message data generated as in the image. The data includes logs as can be seen in the top portion and truck events bottom portion. We will use NiFi to separate this data.

![generate_sh_data](/assets/realtime-event-processing-with-hdf/lab0-nifi/generateSH_data_logs_truckevents_iot.png)

> Note: generate.sh runs java source code located at `iot-truck-streaming/stream-simulator/src/main/java/com/hortonworks/streaming/impl/collectors/StdOutEventCollector.java`. If you would like to see modify/run the code.

## Apache NiFi <a id="apache-nifi-lab0"></a>

[Apache NiFi](https://nifi.apache.org/docs/nifi-docs/html/overview.html#what-is-apache-nifi) is an open source tool for automating and managing the flow of data between systems. To create an effective dataflow, users must understand the various types of processors ![nifi_processor_mini](/assets/realtime-event-processing-with-hdf/lab0-nifi/processor_grey_background_iot.png). This tool is the most important building block available to NiFi because it enables NiFi to perform:

- Data Transformation
- Routing and Mediation
- Database Access
- Attribute Extraction
- System Interaction
- Data Ingestion
- Data Egress/Sending Data
- Splitting and Aggregation
- HTTP
- Amazon Web Services

NiFi is designed to help tackle modern dataflow challenges, such as system failure, data access exceeds capacity to consume, boundary conditions are mere suggestions, systems evolve at different rates, compliance and security.

### Step 2: Install NiFi <a id="step2-install-nifi-lab0"></a>

NiFi will be installed on the Hortonworks Sandbox VirtualBox image because the sandbox does not come with NiFi preinstalled.

The following instructions will guide you through the NiFi installation process. All the steps throughout the tutorial will be done using the latest Hortonworks Sandbox 2.4 on VirtualBox.

1\. Make sure to exit from sandbox shell. Type `exit`. Open a terminal on **local machine**. Download the **install-nifi.sh** file from the github repo. Copy & paste the following commands:

~~~
cd ~
curl -o install-nifi.sh https://raw.githubusercontent.com/hortonworks/tutorials/hdp/assets/realtime-event-processing/install-nifi.sh
~~~

2\. Open a browser. Navigate to `http://hortonworks.com/downloads/`, click the DataFlow next Sandbox tab and download HDF(™) 1.2.0.1: Hortonworks DataFlow tar.gz:

![download_hdf_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/download_hdf_iot.png)

3\. Run the install-nifi.sh script. Below is a definition of how the install nifi command works:

~~~
install-nifi.sh {location-of-HDF-download} {sandbox-host} {ssh-port} {hdf-version}
~~~

> Note: For VMware and Azure users, sandbox-host can be found at the welcome screen after starting your sandbox and ssh-port can be found in Table 1. hdf-version is the digits in the tar.gz name you downloaded, for example the numbers in bold HDF-**1.2.0.1-1**.tar.gz.

After you provide the file path location to HDF Gzip file, sandbox hostname, ssh port number and HDF version, your command should look similar as follows:

~~~
bash install-nifi.sh ~/Downloads/HDF-1.2.0.1-1.tar.gz localhost 2222 1.2.0.1-1
~~~

> Note: You will be asked if you want to continue the download, type `yes`. You will also be asked twice for your ssh password to install NiFi on your Hortonworks Sandbox.

The script automatically installs NiFi onto your virtual machine. After successful completion, NiFi is transported onto the Hortonworks Sandbox and the HDF folder will be located at `~` folder.

### Step 3: Start NiFi <a id="step3-start-nifi"></a>

It is time to start Apache NiFi.

1\. SSH into the Hortonworks Sandbox

~~~
ssh root@127.0.0.1 -p 2222
~~~

2\. Navigate to the `bin` directory using the following command:

~~~
cd hdf/HDF-1.2.0.1-1/nifi/bin
~~~

3\. Run the `nifi.sh` script to start NiFi:

~~~
./nifi.sh start
~~~

> Note: to stop NiFi, type `./nifi.sh stop`

Open the NiFi DataFlow at `http://sandbox.hortonworks.com:6434/nifi/` to verify NiFi started. Wait 1 minute for NiFi to load. If NiFi HTML interface doesn't load, verify the value in the nifi.properties file matches **nifi.web.http.port=6434**.

4\. Navigate to the **conf** folder and open nifi.properties in the vi editor.

~~~
cd ../conf
vi nifi.properties
~~~

5\. Type `/nifi.web.http.port` and press enter. Verify `6434` is the value of nifi.web.http.port as below, else change it to this value:

~~~
nifi.web.http.port=6434
~~~

To exit the vi editor, press `esc` and then enter `:wq` to save the file.
Now that the configuration in the nifi.properties file has been updated, we need to port forward a new port for NiFi through the Port Forward GUI because the virtual machine is not listening for the port **6434**, so NiFi will not load on the browser. If your using VirtualBox Sandbox, refer to section 3.1. For Azure Sandbox users, refer to section 3.2.

### 3.1 Forward Port with VirtualBox GUI

1\. Open VirtualBox Manager

2\. Right click your running Hortonworks Sandbox, click **settings**

3\. Go to the **Network** Tab

Click the button that says **Port Forwarding**. Overwrite NiFi entry with the following values:

| Name  | Protocol  | Host IP  | Host Port  | Guest IP  | Guest Port  |
|---|---|---|---|---|---|
| NiFi  | TCP  | 127.0.0.1  | 6434  |   | 6434  |

![port_forward_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/port_forward_nifi_iot.png)


4\. Open NiFi at `http://sandbox.hortonworks.com:6434/nifi/`. You should be able to access it now. Wait 1 to 2 minutes for NiFi to load.


### 3.2 Forward Port with Azure GUI

1\. Open Azure Sandbox.

2\. Click the sandbox with the **shield icon**.

![shield-icon-security-inbound.png](/assets/realtime-event-processing-with-hdf/lab0-nifi/shield-icon-security-inbound.png)

3\. Under **Settings**, in the **General** section, click **Inbound Security Rules**.

![inbound-security-rule.png](/assets/realtime-event-processing-with-hdf/lab0-nifi/inbound-security-rule.png)

4\. Scroll to **NiFi**, click on the row.

![list-nifi-port.png](/assets/realtime-event-processing-with-hdf/lab0-nifi/list-nifi-port.png)

5\. Change the **Destination Port Range** value from 9090 to 6434.

![change-nifi-port.png](/assets/realtime-event-processing-with-hdf/lab0-nifi/change-nifi-port.png)

6\. Open NiFi at `http://sandbox.hortonworks.com:6434/nifi/`. You should be able to access it now. Wait 1 to 2 minutes for NiFi to load.

### Step 4: Explore NiFi Web Interface <a id="step4-explore-nifi-interface-lab0"></a>

NiFi’s web interface consists of 5 components to build data flows: The **components** toolbar, the **actions** toolbar, the **management** toolbar, the **search** bar and the **help** button. View the image below for a visualization of the user interface for orchestrating a dataflow. Near the top of the UI are multiple toolbars essential for building dataflows.

![nifi_dataflow_html_interface](/assets/realtime-event-processing-with-hdf/lab0-nifi/nifi_dataflow_html_interface.png)

### Step 5: Understand NiFi DataFlow Build Process <a id="step5-create-nifi-dataflow-lab0"></a>

We can begin to build a data flow by adding, configuring and connecting the processors. We will also troubleshoot common problems that occur when creating data flows.
By the end of the IoT Lab Series, you will have built the following dataflow. Refer back to this image if you want to replicate the processors positions on the graph:

![dataflow_withKafka_running_iot](/assets/realtime-event-processing-with-hdf/lab1-kafka/dataflow_withKafka_running_iot.png)

**Figure 1:** This [IoT_Lab_Series_DataFlow.xml](https://raw.githubusercontent.com/james94/tutorials/hdp/assets/realtime-event-processing-with-hdf/IoT_Lab_Series_DataFlow.xml) dataflow performs System Interaction, Splitting and Aggregation, Attribute Extraction, Routing and Mediation and Data Egress/Sending Data.

### 5.1 Download, Import and Drop the Template onto the Graph (Optional)

If you want to view and run the dataflow from the template, follow the steps below, else skip these two steps and move forward.

1\. To open the template xml in NiFi, hover over to the management toolbar and click on the template icon ![template_icon_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/template_icon_nifi_iot.png). Click on the Browse button and find the dataflow xml file that you downloaded and click open. The template should appear in your NiFi Flow Templates spreadsheet.

2\. To display your dataflow template xml onto the screen, drag the template icon from the components toolbar onto the graph. The dataflow should appear as in the dataflow image above, except this dataflow will be missing the PutKafka processor. We will add it in the next lab.

### 5.2 Overiew of Processors in NiFi DataFlow

Eight processors are needed to ingest, filter and store live stream data relating to truck events into your dataflow. Each processor holds a critical role in transporting the enriched data to a destination:

[ExecuteProcess](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi.processors.standard.ExecuteProcess/index.html) Runs the operating system command to activate the stream simulator and the StdOut is redirected such that the content is written to StdOut becomes the content of the outbound FlowFile.

[SplitText](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi.processors.standard.SplitText/index.html) takes in one FlowFile whose content is textual and splits it into 1 or more FlowFiles based on the configured number of lines. Each FlowFile is 1 line.

[UpdateAttribute](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi.processors.attributes.UpdateAttribute/index.html) updates each FlowFile with a unique attribute name. ${UUID()}.

[RouteOnContent](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi.processors.standard.RouteOnContent/index.html) search content of the FlowFile to see if it matches the regex regular expression, such as (Normal), (Overspeed), (“Lane Departure”). The expressions are driving event keywords. If so, the Flowfile is routed to the processor with the configured relationship.

[MergeContent(x2)](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi.processors.standard.MergeContent/index.html) merges many FlowFiles into a single FlowFile. In the tutorial, each FlowFile is merged by concatenating their content together. There will be two processors: one of them will merge all the truck events into a single FlowFile while the other merges all the log data together.

[PutFile(x2)](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi.processors.standard.PutFile/index.html) writes the contents of a FlowFile to a directory on a local file system. There will be two processors: one that writes the filtered logs data to log_data folder. The other that writes the filtered truck event data to truck_events folder.

### 5.3 Troubleshoot Common Processor Issues

You will notice each time you add a new processor, it will have a warning symbol ![warning_symbol_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/warning_symbol_nifi_iot.png) in the upper left corner of the processor face. These warning symbols indicate the processors are invalid.

1\. Hover over one of the processors to troubleshoot the issue. This message informs us of the requirements to make a processor valid, such as the ExecuteProcess.

![warning_message_executeprocess_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/warning_message_executeprocess_nifi_iot.png)

The warning message indicates: we need to enter a command into the value field of the **property** command since it is empty and connect this processor to another component to establish a relationship.
Each Processor will have its own alert message. Let’s configure and connect each processor to remove all the warning messages, so we can have a complete data flow.

### 5.4 Add, Configure & Connect processors

We will build our NiFi DataFlow by adding, configuring and connecting processors. When adding processors, you have three ways to find your desired processor from the **Add Processor** window: **Tags** section left of the table, **Processor List** located in the table and **filter bar** positioned above the table. After we add our processor, we can configure it from the Configure Processor window using the 4 tabs: **Settings**, **Scheduling**, **Properties** and **Commands**. For this lab, we will spend most of our time in the properties tab. The properties in **bold** must contain default or updated values for the processor to run. If you are curious to learn more about a specific property, hover over the help icon next to the Property Name to read a description on that property. Every processor has a relationship on how it transfers data to the next processor, another word for this is connection. Relationships affect how data is transferred between processors. For instance, you can have a **split** relationship that when true transfer a bunch of FlowFiles that were split from one large FlowFile to the next processor.

If you would like to read more about configuring and connecting processors, refer to [Hortonworks Apache NiFi User Guide](http://docs.hortonworks.com/HDPDocuments/HDF1/HDF-1.2.0.1/bk_UserGuide/content/ch_UserGuide.html), Building a DataFlow: section 6.2 and 6.5.

### Step 6: Build Stream Simulator DataFlow Section <a id="step6-build-stream-simulator-dataflow-lab0"></a>

### 6.1 ExecuteProcess

1\. Drag and drop the processor icon onto the graph. Add the **ExecuteProcess** processor. Click the **Add** button.

2\. Right click on ExecuteProcess processor and click the **Configure** button. Move to the **Properties** tab. Add the properties listed in Table 1 to the processor's appropriate properties and if their original properties already have values, update them.

**Table 1:** Update ExecuteProcess Property Values

| Property  | Value  |
|---|---|
| **Command**  | **sh**  |
| Command Arguments  | `/root/iot-truck-streaming/stream-simulator/generate.sh`  |
| Batch Duration  | 10 sec  |

**Command** instructs processor on what type of command to run
**Command Arguments** inform processor which particular directory to look for script files
**Batch Duration** instructs processor to run a task every 10 seconds

![executeProcess_properties_config](/assets/realtime-event-processing-with-hdf/lab0-nifi/executeProcess_properties_config.png)

**Figure 1:** ExecuteProcess Configuration Property Tab Window

3\. Move to the **Scheduling** tab. Modify the **Run Schedule** field from 0 sec to `1 sec`, which makes the processor every 1 second. Click **Apply**.

### 6.2 SplitText

1\. Add the **SplitText** processor below ExecuteProcess. Connect ExecuteProcess to SplitText processor. When the Create Connection window appears, verify **success** checkbox is checked, if not check it. Click **Add**.

2\. Open SplitText **Properties Tab**, add the properties listed in Table 2 to the processor's appropriate properties and if their original properties already have values, update them.

**Table 2:** Update SplitText Property Values

| Property  | Value  |
|---|---|
| **Line Split Count**  | **1**  |
| **Remove Trailing Newlines**  | **false**  |

**Line Split Count** adds 1 line to each split FlowFile
**Remove Trailing Newlines** controls whether newlines are removed at the end of each split file. With the value set to false, newlines are not removed.

![splittext_property_config_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/splittext_property_config_nifi_iot.png)

**Figure 2:** SplitText Configuration Property Tab Window

3\. Move to the **Settings** tab. Check the **failure** and **original** checkbox under **Auto terminate relationships**. Click the **Apply** button.

### 6.3 UpdateAttribute

1\. Add the **UpdateAttribute** processor below SplitText. Connect SplitText to UpdateAttribute processor. When the Create Connection window appears, verify **split** checkbox is checked, if not check it. Click **Add**.

2\. Open SplitText **Properties Tab**, add a new dynamic property for NiFi expression, select the **New property** button. Insert the following property name and value into your properties tab as shown in Table 3 below:

**Table 3:** Update UpdateAttribute Property Value

| Property  | Value  |
|---|---|
| filename  | `{UUID()}`  |

**filename** uses NiFi Expression language to assign each FlowFile a unique name

3\. Click **OK**.

### Step 7: Build Filter Logs & Enrich TruckEvents DataFlow Section <a id="step7-build-filter-logs-enrich-truckevents-lab0"></a>

### 7.1 RouteOnContent

1\. Add the **RouteOnAttribute** processor onto the right of the ExecuteProcess. Connect the UpdateAttribute processor to RouteOnContent. In the Create Connection window, check the **success** checkbox for the relationship.

2\. Open SplitText **Properties Tab**, add the properties listed in Table 4 to the processor's appropriate properties and if their original properties already have values, update them. For the second property and onward, add a new dynamic property for NiFi expression, select the **New property** button. Insert the following property name and value, refer to Table 4.

**Table 4:** Update RouteOnContent Property Values

| Property  | Value  |
|---|---|
| **Match Requirement**  | `content must contain match`  |
| search_for_truck_event_data  | `(Normal)|(Overspeed)|(Lane Departure)|(Unsafe tail distance)|(Unsafe following distance)`  |

**Match Requirements** specifies condition for FlowFile to be transferred to next processor
**search_for_truck_event_data** is Regex Expression that searches each FlowFile for the truck event keywords

![routeOnContent_filter_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/routeOnContent_filter_nifi_iot.png)

**Figure 3:** RouteOnContent Configuration Property Tab Window

3\. Click **OK**. Move to the **Settings** tab, under Auto terminate relationships, check the **unmatched** checkbox. Click **Apply**.

### 7.2 MergeContent(truck_events)

1\. Add the **MergeContent** processor below RouteOnContent. Connect the RouteOnContent processor to this MergeContent processor. In the Create Connection window, check the **search_for_truck_event_data** checkbox for the relationship. All the FlowFiles sent to this processor are truck events.

2\. Open MergeContent **Properties Tab**. Add the properties listed in Table 5 and if their original properties already have values, update them.

**Table 5:** Update MergeContent(truck_events) Property Values

| Property  | Value  |
|---|---|
| **Minimum Number of Entries**  | **50**  |
| Maximum Number of Entries  | 70  |

**Minimum Number of Entries** specifies minimum amount of FlowFiles to gather at the queue before FlowFiles merge together
**Maximum Number of Entries** specifies maximum amount of FlowFiles to gather at the queue before FlowFiles merge together

![mergeContent_property_configs](/assets/realtime-event-processing-with-hdf/lab0-nifi/mergeContent_property_configs.png)

**Figure 4:** MergeContent(truck_events) Configuration Property Tab Window

3\. Navigate to the **Settings** tab, rename the processor: `MergeContent(truck_events)`. Under Auto terminate relationships, check the **failure** and **original** checkboxes. Click **Apply**.


### 7.3 MergeContent(logs)

1\. Right click the MergeContent processor created in the previous step and copy it.  Move the mouse slightly to the right of MergeContent(truck_events) processor and paste it. Connect the RouteOnContent processor to this new MergeContent processor. In the Create Connection window, check the **unmatched** checkbox for the relationship. All the FlowFiles sent to this processor are logs and data we don’t want kafka to receive.

2\. Open configure **Settings** tab, and rename the processor `MergeContent(logs)`. Click **Apply**.

### 7.4 PutFile(truck_events)

1\. Add the **PutFile** processor below MergeContent slightly to the left. Connect the MergeContent(truck_events) to this new PutFile processor. In the Create Connection window, check the **merged** checkbox for the relationship. All the FlowFiles sent to this processor are truck event data we do want kafka to receive.


2\. Open PutFile **Properties Tab**. Add the properties listed in Table 6 and if their original properties already have values, update them.

**Table 6:** Update PutFile(truck_events) Property Values

| Property  | Value  |
|---|---|
| **Directory**  | `/root/nifi_output/truck_events`  |

**Directory** instructs processor which directory to store the output data files

![putfile_properties_truck_events_config_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/putfile_properties_truck_events_config_nifi_iot.png)

**Figure 5:** PutFile(truck_events) Configuration Property Tab Window

3\. Open configure **Settings** tab, and rename the processor `PutFile(truck_events)`. Then check the **failure** and **success** checkbox below the Auto terminated relationships. Click **Apply**.

### 7.5 PutFile(logs)

1\. Right click the PutFile(truck_events) processor created in the previous step and copy it. Move the mouse slightly above MergeContent(logs) processor and paste it. Connect the MergeContent(logs) to this new PutFile processor. In the Create Connection window, check the **merged** checkbox for the relationship. All the FlowFiles sent to this processor are logs and data we don’t want kafka to receive.

2\. Open PutFile **Properties Tab**. Add the properties listed in Table 7 and if their original properties already have values, update them.

Table 7: Update PutFile(logs) Property Values

| Property  | Value  |
|---|---|
| **Directory**  | `/root/nifi_output/log_data`  |

![putfile_properties_config_logs_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/putfile_properties_config_logs_nifi_iot.png)

**Figure 6:** PutFile(logs) Configuration Property Tab Window

3\. Open configure **Settings** tab, and rename the processor `PutFile(log_data)`. Click **Apply**.


We added, configured and connected all processors, your NiFi DataFlow should look similar as below:

![dataflow_lab0_complete_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/dataflow_lab0_complete_nifi_iot.png)

### Step 8: Run NiFi DataFlow <a id="run-nifi-dataflow-lab0"></a>

1\. The processors are valid since the warning symbols disappeared. Notice the processors have a red stop symbol ![stop_symbol_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/stop_symbol_nifi_iot.png) in the upper left corner and are ready to run. To select all processors, hold down the shift-key and drag your mouse across the entire data flow. This step is important if you have different dataflows on the same graph.

![dataflow_selected_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/dataflow_selected_nifi_iot.png)

2\. Now all processors are selected, go to the actions toolbar and click the start button ![start_button_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/start_button_nifi_iot.png). Your screen should look like the following:

![run_dataflow_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/run_dataflow_nifi_iot.png)

Note: To run the DataFlow again, you will need to copy & paste the ExecuteProcess processor onto the graph, then delete the old one, and connect the new one to the splittext procesor. You will need to repeat this process each time you want to run the DataFlow. This step will ensure dataflow flows through each processor. Currently,the ExecuteProcess processor is getting a patch to fix this problem.

3\. To quickly see what the processors are doing and the information on their faces, right click on the graph, click the **refresh status** button ![refresh_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/refresh_nifi_iot.png)

![refresh_dataflow_data_increase_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/refresh_dataflow_data_increase_nifi_iot.png)

> Note: On each processor face, the In, Read/Write and Out all have data increasing.

### 5.6 Check Data Stored In Correct Directory

To check that the log and truck event data were written to the correct directory, wait 20 seconds, then open your terminal and navigate to their appropriate directories. Make sure to SSH into your sandbox.

### 5.7 Verify Logs Stored In log_data Directory

1\. Navigate to through directory path: `/root/nifi_output/log_data`, view the files and open two random files to verify only log data is being sent to this directory.

~~~
cd /root/nifi_output/nifi_output/log_data
ls
vi 28863080789498
~~~

Once the file is opened, you should obtain similar output as below:

![logs_stream_simulator_nifi_output](/assets/realtime-event-processing-with-hdf/lab0-nifi/logs_stream_simulator_nifi_output.png)

> Note: to exit the vi editor, press `esc` and then type `:q`.

### 5.8 Verify Events Stored In truck_events Directory

1\. Navigate to truck events directory: `/root/nifi_output/truck_events`, view the files. Open two random file to verify only event data is being sent to this directory.

~~~
cd /root/nifi_output/truck_events
ls
vi 28918091050702
~~~

Once the file is opened, you should obtain similar output as below:

![truck_events_file_nifi_iot](/assets/realtime-event-processing-with-hdf/lab0-nifi/truck_events_file_nifi_iot.png)

> Note: to exit the vi editor, press `esc` and then type `:q`.

## Summary <a id="summary-lab0"></a>

Congratulations! You made it to the end of the tutorial and built a NiFi DataFlow that reads in a simulated stream of data, filters truck events from log data, stores the truck event data into a specific directory and writes that data to a file. You also learned how to use Apache Maven to package and compile code in order to run the simulator through the shell or by NiFi. You even explored troubleshooting common issues that occur when creating a flow of data. If you are interested in learning to integrate NiFi with other technologies, such as Kafka continue onto our next lab in the IoT tutorial series.

## Further Reading <a id="further-reading-lab0"></a>

- [Apache NiFi Documentation](https://nifi.apache.org/docs.html)
- [Hortonworks DataFlow Documentation](http://docs.hortonworks.com/HDPDocuments/HDF1/HDF-1.2/bk_UserGuide/content/index.html)
- [Apache NiFi Video Tutorials](https://nifi.apache.org/videos.html)
- [Apache Maven Documentation](https://maven.apache.org/)
- [Regex Expression Language](http://regexr.com/)
