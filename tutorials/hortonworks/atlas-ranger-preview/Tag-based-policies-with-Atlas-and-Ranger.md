---
layout: tutorial
title: Tag based policies with Apache Ranger and Apache Atlas
tutorial-id: 600
tutorial-series: Atlas-Ranger Tech Preview
tutorial-version: hdp-2.4.0
intro-page: false
components: [ ranger, atlas, ambari ]
---

# Tag based policies with Apache Ranger and Apache Atlas

##Introduction

Hortonworks has recently announced the integration of Apache Atlas and Apache Ranger, and introduced the concept of tag or classification based policies. Enterprises can classify data in Apache Atlas and use the classification to build security policies in Apache Ranger.

This tutorial walks through an example of tagging data in Atlas and building a security policy in Ranger. 

##Prerequisites

Atlas - Ranger TP VM. You can download it from here (link)


## Steps

### Services Check
After downloading the VM, check if the following services are running in Ambari. Also check if UI is available

Apache Ranger - Ranger UI is available at http://127.0.0.1:6080/  
Apache Atlas -Atlas UI is available at http://127.0.0.1:21000/  
Apache Ambari - Ambari UI is available at http://127.0.0.1:8080/  

![](/assets/tag-based-policies-with-atlas-and-ranger/1-AmbariUI.png)

Ambari and Ranger can be accessed using the following credentials  
User id - *admin*  
Password - *admin*

In the tutorial steps below, we are going to be using user ids *hr_admin* and *hr_user*. You can login into Ambari view using the following credentials  

User id - *hr_admin*  
Password - *hr_admin*  

User id - *hr_user*  
Password - *hr_user*  

It is strongly recommended to change the default passwords in Ambari and Ranger, once you have installed the VM in your environment. 

### Access without tag based policies

In the first scenario, there is an *employee* data table in Apache Hive with *ssn* and *location* as part of the columns. The *ssn* and *location* information is deemed sensitive and users do not have access to it. 

There is a policy in Ranger which allows for access to all columns except *ssn* and *location*. This policy is assigned to *hr* group to which *hr_user* and *hr_admin* users belong to.
![](/assets/tag-based-policies-with-atlas-and-ranger/2-Ranger-Resource-Policy.png)

To check access, login into Ambari – Hive View as *hr_user* user:
![](/assets/tag-based-policies-with-atlas-and-ranger/3-Ambari-Login.png)

After login, invoke the Hive View from the Ambari menu
![](/assets/tag-based-policies-with-atlas-and-ranger/4-Ambari-Hive-View.png)

In the Hive, click on *hr* database and employee table. Run the query `select * from employee LIMIT 100;`
![](/assets/tag-based-policies-with-atlas-and-ranger/5-Hive-View-Error-hruser.png)

You will get an authorization error. This is expected as the user does not have access to 2 columns in this table ( *ssn* and *location*)

You can try running a query for selective columns. `Select id, name, join_date FROM employee LIMIT 100;`
![](/assets/tag-based-policies-with-atlas-and-ranger/6-Hive-View-access-hruser.png)
The query runs successfully. 

Even, *hr_admin* user cannot not see all the columns the location and SSN. We would provide access to this user to all columns later.
![](/assets/tag-based-policies-with-atlas-and-ranger/7-Hive-View-Error-hradmin.png)


### Create tag and tag based policy
Let us assume that the compliance team has designated *ssn* and *location* columns as PII. We can classify the columns in Atlas and Ranger automatically inherits the tags from Atlas. In Ranger, we can use the classification to build a tag based policy for *PII* and give access only to *hr_admin* user. 

As a first step, login into ATLAS web app using http://127.0.0.1:21000/  and execute the query *hive_table* in the search bar to get all Hive tables. Pick the table *hr.employee@erietp*  from the list by clicking on the name

![](/assets/tag-based-policies-with-atlas-and-ranger/8-Atlas-Search.png)

The table view will look like below

![](/assets/tag-based-policies-with-atlas-and-ranger/9-Atlas-table-view.png)

Then, click on the *schema* tab. The click on the tools icon against the *ssn* row 

![](/assets/tag-based-policies-with-atlas-and-ranger/10-Atlas-Schema-View.png)

Then, Select *PII* from the list of tags and click *Save*.

![](/assets/tag-based-policies-with-atlas-and-ranger/11-Assign-PII-Tag.png)
Once the ‘PII’ tag is saved, please click Cancel to go back to the main screen.

Repeat the same for the *location* row from the above list.  Now, you should see both *ssn* and *location* columns are marked with *PII* tag. What essentially it means that we have classified any data in the *ssn* and *location* columns as *PII*

The tag and entity relationship will be automatically inherited by Ranger. In Ranger, we can create a tag based policy by accessing it from the top menu.

![](/assets/tag-based-policies-with-atlas-and-ranger/13-Ranger-Tag-based-Policy-Menu.png)

Click on erietp_tag repository

![](/assets/tag-based-policies-with-atlas-and-ranger/14-Ranger-Tag-based-Repo.png)

Click on the PII Column Access policy

![](/assets/tag-based-policies-with-atlas-and-ranger/15-Ranger-Tag-based-summary.png)

Enable the policy and check the following values

![](/assets/tag-based-policies-with-atlas-and-ranger/16-Ranger-Tag-based-policy.png)

Policy Name - PII column access policy  
*Tag* - PII  
*Description* - Any description  
*Audit logging* - Yes  

In the Allow Conditions, it should have the following values  
*Select Group* - blank, no input  
*Select User* - hr_admin  
*Component Permission* - Hive - Select  

You can select the component permission through this popup

![](/assets/tag-based-policies-with-atlas-and-ranger/17-Ranger-Tag-based-policy-permission.png)
Click to save the policy. 

The Ranger tag based policy is now enabled for *hr_admin* user. You can test it by running the query on all columns in hr.employee table.

![](/assets/tag-based-policies-with-atlas-and-ranger/18-Hive-View-access-hradmin.png)


The query executes successfully. The query can be checked in the Ranger audit log which will show the access granted and associated policy which granted access

![](/assets/tag-based-policies-with-atlas-and-ranger/19-Ranger-audit-tags.png)

*Note* that there are 2 policies which provided access to “hr_admin” user, one is a tag based policy and the other is hive resource based policy. The associated tags (*PII*) is also denoted in the tags column in the audit record)

### Summary
Ranger traditionally provided group or user based authorization for resources such as table, column in Hive or a file in HDFS. 
With the new Atlas -Ranger integration, administrators can conceptualize security policies based on data classification, and not neccessarily in terms of tables or columns. Data stewards can easily classify data in Atlas and use in the classification in Ranger to create security policies.  
This represents a paradigm shift in security and governance in Hadoop, benefiting customers with mature Hadoop deployments as well as customers looking to adopt Hadoop and big data infrastructure for first time. 


