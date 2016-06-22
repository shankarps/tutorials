---
layout: tutorial
title: Getting Started with Apache Zeppelin
tutorial-id: 368
tutorial-series: Zeppelin
tutorial-version: hdp-2.5.0
intro-page: true
components: [ Zeppelin, Spark, Hive, LDAP, Livy ]
---

### **Configuring Zeppelin Spark and Hive Interpreters**

Before you run a notebook to access Spark and Hive, you need to create and configure interpreters for the two components. To create the Spark interpreter, go to the Zeppelin Web UI. Switch to the “Interpreter” tab and create a new interpreter:

1.  Click on the **+Create** button to the right.

2.  Name the interpreter spark-yarn-client.

3.  Select **spark** as the interpreter type.

4.  The next section of this page contains a form-based list of spark interpreter settings for editing. The remainder of the page contains lists of properties for all supported interpreters.

    *  In the first list of properties, specify the following values (if they are not already set). To add a property, enter the name and value into the form at the end of the list, and click **+**.

        <pre>
        master           yarn-client
        spark.home       /usr/hdp/current/spark-client
        spark.yarn.jar   /usr/hdp/current/spark-client/lib/spark-assembly-1.6.0.2.4.0.0-169-hadoop2.7.1.2.4.0.0-169.jar
        </pre>

    *  Add the following properties and settings (HDP version may vary; specify the appropriate version for your cluster):

        <pre>spark.driver.extraJavaOptions -Dhdp.version=2.4.0.0-169
        spark.yarn.am.extraJavaOptions -Dhdp.version=2.4.0.0-169</pre>

    *  When finished, click **Save**.

      **Note:** Make sure that you save all property settings. Without `spark.driver.extraJavaOptions` and `spark.yarn.am.extraJavaOptions`, the Spark job will fail with a message related to bad substitution.

To configure the Hive interpreter:

1.  From the “Interpreter” tab, find the hive interpreter.

2.  Check that the following property references your Hive server node. If not, edit the property value.

    ~~~
    hive.hiveserver2.url
    jdbc:hive2://<hive_server_host>:10000
    ~~~

    **Note**: the default interpreter setting uses the default Hive Server port of 10000\. If you use a different Hive Server port, change this to match the setting in your environment.

3.  If you changed the property setting, click Save to save the new setting and restart the interpreter.

#### Adding and Referencing a spark.files Property

When you have a jar on the node where Zeppelin is running, the following approach can be useful: Add the spark.files property at SPARK_HOME/conf/spark-defaults.conf. For example:

<pre>spark.files  /path/to/my.jar</pre>

#### Adding and Referencing SPARK_SUBMIT_OPTIONS

When you have a jar on the node where Zeppelin is running, this approach can also be useful: Add the SPARK_SUBMIT_OPTIONS environment variable to the ZEPPELIN_HOME/conf/zeppelin-env.sh file. For example:

<pre>export SPARK_SUBMIT_OPTIONS="--packages group:artifact:version"</pre>

### **(Optional) Enabling Zeppelin for Security**

This section describes how to configure Zeppelin to authenticate an end user. (Zeppelin uses Livy to execute jobs, with Spark on YARN as the end user.) Here are the high-level steps to enable Zeppelin Security:

1.  Configure Zeppelin for Authentication

2.  Install the Livy Server and Configure Livy for Zeppelin

3.  (Optional) Enable access control on Zeppelin notebook

**Configure Zeppelin for Authentication**

1.  Create a Zeppelin User Account. When Zeppelin is authenticating end users and Livy propagates the end-user identity to Hadoop, the end user must exist on all nodes. In production you can leverage sssd or pam for this process. In this tech preview, manually add "user1" to all hosts in your cluster. For example, to run Zeppelin as user "user1", issue the following commands as the OS root equivalent on all worker nodes:

    <pre>useradd user1 -g hadoop</pre>

2.  As hdfs user, create an HDFS home directory for user1:

    <pre>su hdfs
    hdfs dfs -mkdir /user/user1
    hdfs dfs -chown user1 /user/user1</pre>

    **Note:** if you configure Zeppelin to run as another user, you must add that user to the OS and create an HDFS home directory for that user.

3.  Edit the Zeppelin shiro configuration. On the node where the Zeppelin server is installed, edit /usr/hdp/current/zeppelin-server/lib/conf/shiro.ini. Make sure that the following lines are in the URL section:

    <pre>[urls]
    /api/version = anon
    #/** = anon
    /** = authcBasic</pre>

    You can use user accounts defined in shiro for authentication. For example, to enable the section to authenticate as user1/password2:

    <pre>[users]
    admin = password1
    user1 = password2
    user2 = password3</pre>

    Alternately, to use LDAP as the identity store, configure the section for your LDAP settings. For example:

    <pre>[main]
    #ldapRealm = org.apache.shiro.realm.ldap.JndiLdapRealm
    #ldapRealm.userDnTemplate = cn={0},cn=engg,ou=testdomain,dc=testdomain,dc=com
    #ldapRealm.contextFactory.url = ldap://ldaphost:389
    #ldapRealm.contextFactory.authenticationMechanism = SIMPLE</pre>

4.  Use Ambari to restart the Zeppelin server. (Ignore the error in Ambari when Zeppelin restarts.)

5.  Access Zeppelin-Tutorial. Login as user1/password2 (or any user defined in your LDAP).

**Note:** Logout functionality is not available in this technical preview, but is being added.

**Install the Livy Server and Configure Livy for Zeppelin**

1.  On the Zeppelin node, install Livy:

    <pre>sudo yum install livy</pre>

2.  Configure the Livy Server:Create /etc/livy/conf/livy-env.sh with the following values. Make sure that the path to Java is accurate for the node.

    <pre>export SPARK_HOME=/usr/hdp/current/spark-client
    export JAVA_HOME=/usr/jdk64/jdk1.8.0_60
    export PATH=/usr/jdk64/jdk1.8.0_60/bin:$PATH
    export HADOOP_CONF_DIR=/etc/hadoop/conf
    export LIVY_SERVER_JAVA_OPTS="-Xmx2g"</pre>

    Create /etc/livy/conf/livy-defaults.conf with the following content:

    <pre>livy.impersonation.enabled = true</pre>

3.  On the node where Livy is installed, create a "livy" user. The Livy process will run as user livy.

    <pre>useradd livy -g hadoop</pre>

4.  Create a logs directory for Livy, and grant user livy permissions to write to it:

    <pre>mkdir /usr/hdp/current/livy-server/logs
    chmod 777 logs</pre>

5.  On the Livy node, edit /etc/spark/conf/spark-defaults.conf to add the following:

    <pre>spark.master yarn-client</pre>

6.  Grant user livy the ability to proxy users in the Hadoop core-site.xml file. Use Ambari to add the following lines to the /etc/hadoop/conf/core-site.xml file. Then restart HDFS using Ambari.

    ~~~
    <property>
        <name>hadoop.proxyuser.**livy**.groups</name>
        <value>*</value>
    </property>

    <property>
        <name>hadoop.proxyuser.**livy**.hosts</name>
        <value>*</value>
    </property>
    ~~~

7.  Start the Livy server as user livy:

    <pre>cd /usr/hdp/current/livy-server
    su livy
    ./bin/livy-server start</pre>

8.  Configure Zeppelin to use Livy. In Zeppelin, notebooks are run against configured interpreters. Go to your notebook and click on "interpreter binding."<br><br> ![interp-binding](http://hortonworks.com/wp-content/uploads/2016/05/interp-binding.png) <br><br> On the next page, select the interpreters you want to use. (To select an interpreter, click on the interpreter in a toggle manner. An un-selected interpreter appears in white text.)You can reorder the interpreters available to your notebook by dragging and dropping them. For example, in the following screenshot the Livy Spark interpreter is selected ahead of Spark. It will be launched with %lspark.<br><br> ![livy-interp](http://hortonworks.com/wp-content/uploads/2016/05/livy-interp-268x300.png)<br>

9.  Confirm Livy Interpreter settings. If you have Livy installed on another node, replace localhost in the Livy URL with your Livy host.If you made any changes to the Livy interpreter setting, re-start the Livy interpreter.<br><br> ![livy-interp-setting](http://hortonworks.com/wp-content/uploads/2016/05/livy-interp-setting-300x99.png)<br><br>

10.  Run Notebooks with the Livy Interpreter. Livy supports Spark, SparkSQL, PySpark, and SparkR.To run notes with Livy, use the corresponding "magic string" at the top of your note. For example, specify %lspark to run Scala code via Livy, or %lspark.sql to run against SparkSQL via Livy.To use SQLContext with Livy, do not create SQLContext explicitly. Zeppelin creates SQLContext by default. If necessary, remove the following lines from your SparkSQL note:

~~~
//val sqlContext = new org.apache.spark.sql.SQLContext(sc)
//import sqlContext.implicits._
~~~

**(Optional) Enable access control on Zeppelin notebook**

Zeppelin supports access control on each notebook. The following instructions configure Zeppelin to authorize end users to access Zeppelin notebooks.

1.  Click the lock icon on the notebook to configure access to that notebook:<br><br> ![default-button](http://hortonworks.com/wp-content/uploads/2016/05/default-button.png)<br><br>

2.  On the next popup menu, add users who should have access to the policy:<br><br>![add-users](http://hortonworks.com/wp-content/uploads/2016/05/add-users-300x112.png) <br><br>**Note:** with identity propagation enabled via Livy, data access is controlled by the data source being accessed. For example, when you access HDFS as user1, data access is controlled by HDFS permissions.

### **LDAP Authentication Configuration**

Zeppelin with LDAP authentication allows users to authenticate users and provide separation of notebooks.

**Note:** By default Zeppelin is enabled to receive requests over HTTP & not HTTPS. When you enable LDAP Authentication for Zeppelin, it will send username/password over HTTP. For better security, you should enable Zeppelin to listen in HTTPS by enabling SSL. You can use SSL properties specified in [this](https://zeppelin.incubator.apache.org/docs/0.5.6-incubating/install/install.html) doc.

To enable authentication, in /usr/hdp/current/zeppelin-server/conf/shiro.ini file edit the section and enable authentication [urls]

<pre>#/** = anon
/** = authcBasic
For local user configuration, enable the section [users]
admin = password1
user1 = password2
user2 = password3
Alternatively for LDAP integration, enable the section [main]
#ldapRealm = org.apache.shiro.realm.ldap.JndiLdapRealm
#ldapRealm.userDnTemplate = cn={0},cn=engg,ou=testdomain,dc=testdomain,dc=com
#ldapRealm.contextFactory.url = ldap://ldaphost:389
#ldapRealm.contextFactory.authenticationMechanism = SIMPLE

</pre>

For more information on Shiro please refer to [Apache Shiro Authentication Features](http://shiro.apache.org/authentication-features.html).

### **Stopping Zeppelin or the Livy Server**

To stop the Zeppelin server, use Ambari.

To stop the Livy server:

<pre>su livy; cd /usr/hdp/current/livy-server; ./bin/livy-server stop</pre>
