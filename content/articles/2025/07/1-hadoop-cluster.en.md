---
title: Creating a Computer Cluster with Apache Hadoop
type: docs
weight: 1
editURL: "https://devnotes.msglabs.com.br/articles/hadoop-cluster/"
prev: /articles/2025/08/1-zabbix-and-grafana
---

This article was originally created in 2023 as part of the assessment project for the 2nd unit of the Computer Architecture course in the Computer Science bachelor's degree at the Federal Institute of Sergipe, where I am a student.

I kept the document only in my Google Drive, but I thought it would be a good idea to write it here and revise part of the content.

In the college project, we had the following scenario: 3 *Raspberry Pi* SBCs, where one was the ***Name Node*** (also called ***main***/***master***), meaning it controlled the entire *cluster*, and two acted as ***Data Nodes*** (also called ***nodes***/***slaves***) which were responsible for the actual processing.
It is possible to include the *main* machine as one of the *nodes*, meaning that besides coordinating the entire *cluster*, it also processes data. However, that is not the approach adopted here. Actually, this revision will use virtual machines and an improved configuration. Nevertheless, there will be an indication for those who decide to set the *main* to process data along with the *nodes*.

> [!IMPORTANT]
> The values discussed and parameters used were based on the project's needs and tests performed, and therefore should not be followed strictly.
> Adjust all parameters according to your project's needs and your hardware capacity.

## Updating repositories and packages:

Before starting, it is important to update the repositories and packages installed on the system. In our case, we are using a Debian-based system, so just type the following command:

```bash
sudo apt update && sudo apt upgrade
```

## Steps required to create the cluster:

The following steps are fundamental for the *cluster* operation:



### 1 - Java OpenJDK Installation (main/nodes):

Install Java OpenJDK on both the *main* machine and the *nodes*:
```bash
sudo apt install openjdk-11-jdk
```




### 2 - Hadoop Download (main/nodes):

Use the command below to download Hadoop version 3.3.6:
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
```

If there are issues, access the repository and download the corresponding package:
[https://dlcdn.apache.org/hadoop/common/](https://dlcdn.apache.org/hadoop/common/)

> [!WARNING]
> The package name must look like `hadoop-VERSION.tar.gz`!

Extract the package:
```bash
tar xzf hadoop-3.3.6.tar.gz
```

Preferably, rename the folder just to “hadoop” and move the folder to an easily accessible location, such as `/` or `/usr/local/`. The following command performs both operations:
```bash
sudo mv hadoop-3.3.6 /usr/local/hadoop
```




### 3 - Hadoop Configuration (main/nodes):

Access the hadoop *environment* script:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Look for `export JAVA_HOME`, uncomment it, and indicate the Java OpenJDK path:
```sh {filename="hadoop-env.sh"}
export JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64
```

> [!WARNING]
> The package name must match the system architecture version!
> To find the file name, you can navigate to `/usr/lib/jvm` and list the directories inside with the `ls` command.

Press `Ctrl + S` to save, `Ctrl + X` to exit.



### 4 - Configuring the Environment (main/nodes):

Access the *environment* file:
```bash
sudo nano /etc/environment
```

Indicate the hadoop *PATH* and the JAVA_HOME path:
```sh {filename="environment"}
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/hadoop/bin:/usr/local/hadoop/sbin"

JAVA_HOME="/usr/lib/jvm/java-1.11.0-openjdk-amd64"
```

> [!WARNING]
> The paths `/usr/local/hadoop/bin` and `/usr/local/hadoop/sbin` may change depending on where your Hadoop is located. Check and change the path if necessary.
> The Java package name must match the system architecture version!

Press `Ctrl + S` to save, `Ctrl + X` to exit.




### 5 - Hadoop User (main/nodes):

> [!WARNING]
> The following step is only necessary if your default user is not hadoop. If the hadoop user already exists, just skip to the next step.

Add hadoop user:
```bash
sudo adduser hadoop
```

> [!TIP]
> Information such as name, number, etc., can be skipped by pressing `ENTER`.

Grant administrator permissions to the hadoop user and assign ownership of the hadoop folder:
```bash
sudo usermod -aG hadoop hadoop
sudo chown hadoop:root -R /usr/local/hadoop/
sudo chmod g+rwx -R /usr/local/hadoop/
sudo adduser hadoop sudo
```




### 6 - Network Settings (main/nodes):

#### 6.1 - Enable SSH:

Type one of the commands below to enable the SSH service. The `enable` command starts the service so it launches automatically on system boot, and the `start` command starts the service immediately; `&&` allows both commands to run in sequence. The `enable --now` command combines both actions, enabling the service to start automatically and starting it immediately, without running the commands separately or using `&&`.

```bash
sudo systemctl enable ssh && sudo systemctl start ssh
```

or

```bash
sudo systemctl enable --now ssh
```

#### 6.2 - Configure Static IP (Debian):

> [!CAUTION]
> In our scenario, tests were being performed at college, and to avoid major issues, we placed the machines on an isolated network connected only to a simple L2 switch.
> Depending on the configuration, the machine may lose internet access!
> So, if you have any optional configuration that requires downloading packages from the internet, such as monitoring programs, this might be a good time to perform that configuration.

Access the IP configuration file:
```bash
sudo nano /etc/network/interfaces
```

In the file, you will find the network interface configurations. The primary *interface* will be something like:
```sh {filename="interfaces"}
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp
```

Remove the `dhcp` parameter and add the machine's IP settings. Example:
```sh {filename="interfaces"}
allow-hotplug enp0s3
iface enp0s3 inet static
    address 192.168.0.X
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 192.168.0.1 8.8.8.8
    dns-search cluster.local

# 'allow-hotplug enp0s3': can be 'auto enp0s3'.
# 'iface enp0s3 inet static:': Replace with interface name and disable DHCP.
# 'address 192.168.0.X': Defines the machine's IP address.
# 'netmask 255.255.255.0': Defines the subnet mask.
# 'gateway 192.168.0.1': Defines the Gateway address.
# 'dns-nameservers 192.168.0.1 8.8.8.8': Defines DNS server addresses, e.g.: 8.8.4.4 9.9.9.9 1.1.1.1
# 'dns-search cluster.local': Defines the search domains. It can be another domain, e.g.: lab.local.
```

> [!NOTE]
> The `X` will be the machine number. For example, `192.168.0.10/24` for the *main*/*master*.
> In `dns-nameservers`, you can configure more than one DNS server, like Google's: `8.8.8.8` and `8.8.4.4`, or your router's IP if you have one on the network.
> You can configure other IP ranges, but machines will only be able to communicate if they are on the same network.

> [!WARNING]
> If the network where the machines are connected has an active DHCP server, be careful to set IP addresses outside the DHCP server range. In the scenario discussed here, the machines are connected to a switch and are on an isolated network.

Press `Ctrl + S` to save, `Ctrl + X` to exit.

Apply the new network settings:
```bash
sudo ifdown enp0s3 && sudo ifup enp0s3
```

If it doesn't work, you can apply the settings by restarting the computer or the *network* service.
If you are accessing the machine via ssh, you will likely lose connection and may have trouble connecting with the new IP.

To restart the service:
```bash
sudo systemctl restart networking
```

Restart the machine:
```bash
sudo reboot
```

#### 6.3 - Configure Static IP (Ubuntu):

> [!CAUTION]
> In our scenario, tests were being performed at college, and to avoid major issues, we placed the machines on an isolated network connected only to a simple L2 switch.
> Depending on the configuration, the machine may lose internet access!
> So, if you have any optional configuration that requires downloading packages from the internet, such as monitoring programs, this might be a good time to perform that configuration.

Type the following command to find out the network *interface* where the IP is configured:
```bash
ip address
```

or:
```bash
ifconfig
```

Navigate to the `/etc/netplan` folder:
```bash
cd /etc/netplan
```

Now list the files:
```bash
ls
```

Access or create the file in the folder:
```bash
sudo nano 01-network.yaml
```

Enter the IP address on the desired *interface*. Example:
```yaml {filename="01-network.yaml"}
network:
    version: 2
    renderer: networkd
    ethernets:
        enp0s3:
            dhcp4: false
            addresses:
            - 192.168.0.X/24
            routes:
                - to: default
                  via: 192.168.0.1
            nameservers:
                search:
                    - cluster.local
                addresses:
                    - 192.168.0.1
                    - 8.8.8.8
# RESPECT INDENTATION!
# 'enp0s3': Replace with interface name, can be enp0s3, eth0, etc.
# 'dhcp4: false' disables DHCP.
# 'addresses: 192.168.0.X/24' Defines the machine's IP address.
# 'routes: to: default' Defines the default route.
# 'routes: via: 192.168.0.1' Defines the Gateway address.
# 'nameservers: search:' Defines the search domains. It can be another domain, e.g.: lab.local.
# 'nameservers: addresses:' Defines DNS server addresses, e.g.: 8.8.4.4 9.9.9.9 1.1.1.1
```

> [!NOTE]
> The `X` will be the machine number. For example, `192.168.0.10/24` for the *main*/*master*.
> If you decide to use the `50-cloud-init.yaml` file, you must disable **cloud init** as instructed in the comments at the beginning of the file.
> You can configure other IP ranges, but machines will only be able to communicate if they are on the same network.

> [!WARNING]
> If the network where the machines are connected has an active DHCP server, be careful to set IP addresses outside the DHCP server range. In the scenario discussed here, the machines are connected on an isolated network.

Press `Ctrl + S` to save, `Ctrl + X` to exit.

Change the file permissions to avoid access issues:
```bash
sudo chmod 600 01-network.yaml
```

Apply the new network settings:
```bash
sudo netplan try
```

The command above first tests the configuration, and if the syntax and indentations are correct, an option will appear to press `ENTER` confirming the changes within 120 seconds. If the time runs out and no action is taken, the modifications are discarded.

If you wish to apply the modifications directly without testing, use the following command:
```bash
sudo netplan apply
```

#### 6.4 - Configure the hosts file and hostname:

Access the *hosts* file:
```bash
sudo nano /etc/hosts
```

In the file, you should insert the IPs and the *hostname* of the machines, for example:
```sh {filename="hosts"}
192.168.0.10 main
192.168.0.11 node1
192.168.0.12 node2
```

Press `Ctrl + S` to save, `Ctrl + X` to exit.

To change the *host* name, access:
```bash
sudo nano /etc/hostname
```

The *hostname* must match the machine. Example: "node1" for the machine `node1`.

Press `Ctrl + S` to save, `Ctrl + X` to exit.

Restart the machine:
```bash
sudo reboot
```




### 7 - Configuring SSH Access (main):

If the `hadoop` user is not the default user and you created the user in step `5 - Hadoop User`, switch to the `hadoop` user:
```bash
su - hadoop
```

Run the following command to generate an ssh key:
```bash
ssh-keygen -t rsa
```

> [!TIP]
> When asked to fill in the location where the *key* will be created and the *passphrase*, just skip by pressing `ENTER`.

Send this key to the other machines:
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop@X
```

> [!NOTE]
> The parameter “X” corresponds to the machine name. Example: `hadoop@node1` to send to the `node1` machine.

> [!WARNING]
> It is necessary to send keys from *main* to all other *nodes*.




### 8 - Hadoop Settings - Core, HDFS, MapReduce, YARN - (main):

#### 8.1 - Configure Hadoop Core file:

Access the file:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/core-site.xml
```

Between the `<configuration>` and `</configuration>` tags, insert the following information:
```xml {filename="core-site.xml"}
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://main:9000</value>
</property>
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>hadoop</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `fs.defaultFS` | Default URI of the Hadoop file system. |
| `hadoop.http.staticuser.user` | Defines which user will be assumed by the Hadoop HTTP server, allowing file management via web *interface*. |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.


#### 8.2 - Configure Hadoop HDFS file:

Access the file:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hdfs-site.xml
```

Between the `<configuration>` and `</configuration>` tags, insert the following information:
```xml {filename="hdfs-site.xml"}
<property>
    <name>dfs.namenode.name.dir</name>
    <value>/usr/local/hadoop/data/nameNode</value>
</property>
<property>
    <name>dfs.datanode.data.dir</name>
    <value>/usr/local/hadoop/data/dataNode</value>
</property>
<property>
    <name>dfs.replication</name>
    <value>2</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `dfs.namenode.name.dir` | Local directory for NameNode metadata. |
| `dfs.datanode.data.dir` | Local directory where DataNodes store blocks. |
| `dfs.replication` | Number of replicas per block in HDFS. |

{{% /details %}}

> [!NOTE]
> The default value is 3, meaning each data block will be replicated on 3 different *DataNodes*. This ensures high availability and fault tolerance but also increases disk space usage.
> The configuration depends on the number of *nodes* you have and the desired resilience for the *cluster*.

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 8.3 - Configure Hadoop MapReduce file:

Access the file:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/mapred-site.xml
```

Between the `<configuration>` and `</configuration>` tags, insert the following information:
```xml {filename="mapred-site.xml"}
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
<property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
</property>
<property>
    <name>mapreduce.application.classpath</name>
    <value>
        /usr/local/hadoop/share/hadoop/mapreduce/*,
        /usr/local/hadoop/share/hadoop/mapreduce/lib/*,
        /usr/local/hadoop/share/hadoop/common/*,
        /usr/local/hadoop/share/hadoop/common/lib/*,
        /usr/local/hadoop/share/hadoop/yarn/*,
        /usr/local/hadoop/share/hadoop/yarn/lib/*,
        /usr/local/hadoop/share/hadoop/hdfs/*,
        /usr/local/hadoop/share/hadoop/hdfs/lib/*
    </value>
</property>
<!-- OPTIONAL - Settings to view the history of completed jobs.
Complementary to `yarn.log-aggregation-enable` -->
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>main:10020</value>
</property>
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>main:19888</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `mapreduce.framework.name` | Defines which execution *framework* will be used by *MapReduce*. The value `yarn` indicates that **YARN (Yet Another Resource Negotiator)** will be used. |
| `yarn.app.mapreduce.am.env` | Defines environment variables for the *MapReduce* **ApplicationMaster**. |
| `mapreduce.map.env` | Defines environment variables for *Map* tasks. |
| `mapreduce.reduce.env` | Defines environment variables for *Reduce* tasks. |
| `mapreduce.application.classpath` | Defines the classpath required for executing *MapReduce* jobs. |
| `mapreduce.jobhistory.address` | Address and port where the **JobHistory Server** listens for RPC requests from CLI/API clients. |
| `mapreduce.jobhistory.webapp.address` | Address and port of the **web (HTTP)** *interface* of the *JobHistory Server*. |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.


#### 8.4 - Configure Hadoop YARN file:

Access the file:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Between the `<configuration>` and `</configuration>` tags, insert the following information:
```xml {filename="yarn-site.xml"}
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>main</value>
</property>
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<property>
    <name>yarn.nodemanager.auxservices.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
</property>
<!-- OPTIONAL - Log aggregation settings -->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>
<property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>172800</value>
</property>
<property>
  <name>yarn.log.server.url</name>
  <value>http://main:19888/jobhistory/logs</value>
</property>
<property>
    <name>yarn.nodemanager.remote-app-log-dir</name>
    <value>/tmp/logs</value>
</property>
<property>
    <name>yarn.nodemanager.remote-app-log-dir-suffix</name>
    <value>logs</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `yarn.resourcemanager.hostname` | Host where the YARN *ResourceManager* is listening. |
| `yarn.nodemanager.aux-services` | Auxiliary service enabled on the *NodeManager*. |
| `yarn.nodemanager.auxservices.mapreduce.shuffle.class` | Java class implementing the *shuffle* service. |
| `yarn.log-aggregation-enable` | Enables **log aggregation** for *containers* after applications finish. |
| `yarn.log-aggregation.retain-seconds` | Time in seconds that aggregated logs should be kept in HDFS. |
| `yarn.log.server.url` | Configures the base URL to access aggregated application logs using log redirection from *ResourceManager* to *JobHistory Server*. |
| `yarn.nodemanager.remote-app-log-dir` | HDFS directory where aggregated application logs will be stored. |
| `yarn.nodemanager.remote-app-log-dir-suffix` | Suffix for the final remote log path (useful for organizing subfolders by application/user). |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 8.5 - Configure nodes file in hadoop:

Access the file:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/workers
```

Remove the name *localhost* and add the *hostnames* of the *nodes*. Example:
```sh {filename="workers"}
node1
node2
```

If you want to keep the *main* machine as a *node*, add the parameter `main`:
```sh {filename="workers"}
main
node1
node2
```

In this guide, we will keep only `node1` and `node2` machines.

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 8.6 - Send the folder with modified files to nodes:

Run the command below:
```bash
scp /usr/local/hadoop/etc/hadoop/* X:/usr/local/hadoop/etc/hadoop/
```

> [!NOTE]
> The parameter “X” corresponds to the machine name. Example: "node1:/usr/local/hadoop/etc/hadoop/" to send to the `node1` machine.

> [!WARNING]
> It is necessary to send the modified files from *main* to all other *nodes*.

#### 8.7 - Export paths:

Type the commands below **on all machines** under the Hadoop user to export the application *PATHs*:
```bash
export HADOOP_HOME="/usr/local/hadoop"
export HADOOP_COMMON_HOME="/usr/local/hadoop"
export HADOOP_CONF_DIR="/usr/local/hadoop/etc/hadoop"
export HADOOP_HDFS_HOME="/usr/local/hadoop"
export HADOOP_MAPRED_HOME="/usr/local/hadoop"
export HADOOP_YARN_HOME="/usr/local/hadoop"
```




### 9 - Create nameNode folder (main):

Type the following command only on the *main* machine:
```bash
mkdir -p /usr/local/hadoop/data/nameNode
```




### 10 - Create dataNode folder (nodes):

Type the following command only on the *nodes*:
```bash
mkdir -p /usr/local/hadoop/data/dataNode
```




### 11 - Format the Hadoop Distributed File System - HDFS (main):

Type the command below to load environment variables:
```bash
source /etc/environment
```

Type the command below to format HDFS:
```bash
hdfs namenode -format
```

> [!WARNING]
> It is common to delete the nameNode and dataNode folders to fix some possible errors, or after changing hadoop files. In both cases, performing a new HDFS format is mandatory to apply changes!




### 12 - Initialize, Monitor, and Stop the Cluster (main):

#### 12.1 - Initializing the cluster:

Whenever you wish to initialize the *cluster*, you will first have to load the environment variables:
```bash
source /etc/environment
```

Then, type the following command to initialize HDFS:
```bash
start-dfs.sh
```

Right after, type the following command to initialize YARN:
```bash
start-yarn.sh
```

To start all Hadoop services at once, you can use the command:
```bash
start-all.sh
```

If you have configured the **JobHistory Server**, type the following command to start the service:
```bash
mapred --daemon start historyserver
```

> [!NOTE]
> If an error appears: "main: hadoop@main: Permission denied (publickey,password).", it means the SSH service is failing to authenticate the *main* machine with its SSH key. To fix it, send the *main* machine's SSH key to itself using the `ssh-copy-id` command: `ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop@main`

To stop the *cluster* services, just replace `start` with `stop` in the command.

To verify if the *cluster* initialized correctly, you can type on both *main* and *nodes*, the command below:
```bash
jps
```


This command should return something similar to the images below:  
![JPS Output main][image1]
![JPS Output node 1][image2]
![JPS Output node 2][image3]

If you are using the *main* machine as a *node*, the `DataNode` and `NodeManager` processes will also appear.

![JPS Output main as node][image4]

If you are using **JobHistory Server**, the parameter `JobHistoryServer` should appear in the `jps` output.


#### 12.2 - Monitoring the cluster:

##### 12.2.1 - Accessing NameNode web interface:

The `start-dfs` command, besides initializing the Hadoop file system, will also initialize a web interface with information about the **NameNode** *daemon*. There you will see information about the *cluster* and *HDFS*.

To access, type in the browser: `nameNode_IP:9870` or `main:9870`.

In the Datanodes tab, you will see the *nodes* that are connected to the *cluster*.

##### 12.2.2 - Accessing ResourceManager web interface:

The `start-yarn` command, besides initializing the *cluster* services, will also initialize a web interface with information about the **ResourceManager** *daemon*. There you can see information about the *nodes* connected to the *cluster* and the submitted, running, and finished applications.

To access, type in the browser: `resourceManager_IP:8088` or `main:8088`.

You should see information about the *cluster* similar to the previous example with information about connected *nodes*, such as: number of containers, memory, vCores, etc., besides information about jobs/applications as previously stated.

##### 12.2.3 - Accessing JobHistory Server:

The command `mapred --daemon start historyserver` will start the **MapReduce JobHistory Server** *daemon*, which is responsible for storing the history of jobs executed in the *cluster*.

With `JobHistory_IP:19888/jobhistory` or `main:19888/jobhistory`, you can access the history of jobs that were executed in the *cluster*.

#### 12.3 - Other useful commands:

* `yarn node -list`: lists the *nodes* connected to the *cluster*.
* `yarn application -list`: lists applications running in the *cluster*.
* `mapred --daemon stop historyserver`: stops the **JobHistory Server** service.
* `hdfs help`: displays available commands for HDFS.
* `hdfs dfsadmin -safemode leave`: leaves HDFS safe mode, allowing write operations to be performed (in case a write error appears).
* `hdfs dfs -put /local-source /HDFS-destination`: sends a file from the local file system to HDFS.
* `hdfs dfs -get /HDFS-source /local-destination`: downloads a file from HDFS to the local file system.
* `hdfs dfs -ls /`: lists files and directories in HDFS.
* `hdfs dfs -mkdir /directory`: creates a directory in HDFS.
* `hdfs dfs -rm /file`: removes a file from HDFS.
* `hdfs dfs -rmdir /directory`: removes an empty directory from HDFS.
* `hdfs dfs -rm -r /directory`: removes a directory and all its contents from HDFS.




## Optional Steps:

The following steps are optional but can be useful for those who wish to monitor the *cluster* or perform benchmark tests. It is not necessary to implement all of them; feel free to add only what you wish.

### 13 - Configuring **start-history** and **stop-history**:

Access the **bashrc** file:
```bash
sudo nano ~/.bashrc
```

Paste the startup and shutdown command for **JobHistory Server**:
```sh {filename=".bashrc"}
# Start JobHistory Server
alias start-history='mapred --daemon start historyserver'

# Stop JobHistory Server
alias stop-history='mapred --daemon stop historyserver'
```

Press `Ctrl + S` to save, `Ctrl + X` to exit.

Load the **bashrc** file:
```bash
source ~/.bashrc
```

Now you can start everything with `start-all.sh && start-history` and stop everything with `stop-all.sh && stop-history`.




### 14 - Configuring resource limits:

#### 14.1 - Application Limits on Nodes (via YARN):

When not configured, YARN assumes default values defined in `yarn-default.xml`, which may not be ideal in a *cluster* with nodes that have low resource capacity, like the *Raspberry Pi* cluster, for example. Hadoop is also capable of automatically detecting machine resources. To perform this configuration, see section [15 - Configuring automatic resource detection](#15---configuring-automatic-resource-detection).

<!-- The default values for YARN are: -->
{{% details title="Default YARN values (Click to expand)" closed="true" %}}

| Parameter | Default Value | Explanation |
| --- | --- | --- |
| `yarn.nodemanager.resource.memory-mb` | `-1` | If defined as `-1` and `yarn.nodemanager.resource.detect-hardware-capabilities` is `true`, it will be calculated automatically (in the case of Windows and Linux). In other cases, the default is 8192 MiB (8GiB). |
| `yarn.nodemanager.resource.cpu-vcores` | `-1` | If defined as -1 and `yarn.nodemanager.resource.detect-hardware-capabilities` is `true`, the number will be determined automatically by the hardware in the case of Windows and Linux. In other cases, the number of vCores is 8 by default. |
| `yarn.scheduler.minimum-allocation-mb` | `1024` | Memory requests lower than this value will be set to this property's value. Also, a *node manager* configured to have less memory than this value will be shut down by the *resource manager*. |
| `yarn.scheduler.maximum-allocation-mb` | `8192` | Memory requests greater than this will throw an `InvalidResourceRequestException`. |
| `yarn.scheduler.minimum-allocation-vcores` | `1` | Requests lower than this value will be set to this property's value. Also, a *node manager* configured to have fewer virtual cores than this value will be shut down by the *resource manager*. |
| `yarn.scheduler.maximum-allocation-vcores` | `4` | Requests greater than this will throw an `InvalidResourceRequestException`. |

{{% /details %}}

> [!NOTE]
> You can check YARN default values on the [official Apache Hadoop yarn-default page](https://hadoop.apache.org/docs/r3.4.0/hadoop-yarn/hadoop-yarn-common/yarn-default.xml).
> For other configurations, consult the [official Apache Hadoop documentation (3.4.0)](https://hadoop.apache.org/docs/r3.4.0/).

##### 14.1.1 - Configuring resource limits for applications (nodes):

Access the `yarn-site.xml` file on each *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Between the `<configuration>` and `</configuration>` tags, and below previous settings, add the following properties:
```xml {filename="yarn-site.xml"}
<!-- Limits for applications - nodes -->
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>3072</value>
</property>
<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>3</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `yarn.nodemanager.resource.memory-mb` | Defines the total amount of physical memory, in MiB, that YARN can use on this node. |
| `yarn.nodemanager.resource.cpu-vcores` | Defines the number of vCores (virtual CPU cores) that YARN can use on this node. |

{{% /details %}}

> [!TIP]
> Set the amount of memory to 75-80% of the machine's total RAM. If a node has 8GiB, use 6144 (6GiB).
> Set the number of vCores to 75-80% of the machine's total CPU cores. If a node has 6 cores, use 4 or 5.

Press `Ctrl + S` to save, `Ctrl + X` to exit.

##### 14.1.2 - Configuring resource limits for Applications (main):

Access the `yarn-site.xml` file on the *main* machine:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Between the `<configuration>` and `</configuration>` tags, and below previous settings, add the following properties:
```xml {filename="yarn-site.xml"}
<!-- Limits for applications - main -->
<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>512</value>
</property>
<property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>2048</value>
</property>
<property>
    <name>yarn.scheduler.minimum-allocation-vcores</name>
    <value>1</value>
</property>
<property>
    <name>yarn.scheduler.maximum-allocation-vcores</name>
    <value>3</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `yarn.scheduler.minimum-allocation-mb` | The smallest unit of memory that YARN will allocate to a container. Generally, it is good to align this with the RAM of a *map/reduce* container. |
| `yarn.scheduler.maximum-allocation-mb` | The largest amount of memory that **A SINGLE** task (container) can request. This prevents a misconfigured task from trying to use all of a *node*'s memory. This value **CANNOT** be greater than `yarn.nodemanager.resource.memory-mb`. |
| `yarn.scheduler.minimum-allocation-vcores` | The smallest unit of vCores that YARN will allocate. |
| `yarn.scheduler.maximum-allocation-vcores` | The maximum number of vCores that **A SINGLE** task can request. This value **CANNOT** be greater than `yarn.nodemanager.resource.cpu-vcores`. |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 14.2 - Limits for Hadoop Daemons (JVM Heap):

In the case of Hadoop *Daemons*, it is important to configure the JVM *heap* size (data structure type) for each of them. This ensures they have enough memory to function correctly, especially in larger *clusters*.
There is no default value, as it scales automatically based on machine capacity.

##### 14.2.1 - Configuring resource limits for Daemons (main):

Access the `hadoop-env.sh` file on the *main* machine:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

At the end of the file, add the following lines:
```sh {filename="hadoop-env.sh"}
# Heap for NameNode (VERY important)
# Depends on the number of files/blocks in HDFS.
export HDFS_NAMENODE_OPTS="-Xms1024m -Xmx2048m"

# Heap for ResourceManager
# Depends on the number of nodes and apps.
export YARN_RESOURCEMANAGER_OPTS="-Xms1024m -Xmx2048m"

# Heap for MapReduce JobHistory Server
# Increase if the UI becomes slow.
export HADOOP_JOB_HISTORYSERVER_OPTS="-Xms1024m -Xmx2048m"
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `HDFS_NAMENODE_OPTS` | Defines minimum and maximum memory options for the *NameNode*. |
| `YARN_RESOURCEMANAGER_OPTS` | Defines minimum and maximum memory options for the *ResourceManager*. |
| `HADOOP_JOB_HISTORYSERVER_OPTS` | Defines minimum and maximum memory options for the *MapReduce JobHistory Server*. |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.

##### 14.2.2 - Configuring resource limits for Daemons (nodes):

Access the `hadoop-env.sh` file on each *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

At the end of the file, add the following lines:
```sh {filename="hadoop-env.sh"}
# Heap for DataNode
# Generally doesn't need much memory.
export HDFS_DATANODE_OPTS="-Xms512m -Xmx1024m"

# Heap for NodeManager
# The daemon itself doesn't need much.
export YARN_NODEMANAGER_OPTS="-Xms512m -Xmx1024m"
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `HDFS_DATANODE_OPTS` | Defines minimum and maximum memory options for the *DataNode*. |
| `YARN_NODEMANAGER_OPTS` | Defines minimum and maximum memory options for the *NodeManager*. |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 14.3 - MapReduce Limits:

When not configured, MapReduce assumes default values defined in `mapred-default.xml`, which may not be ideal in a *cluster* with nodes that have low resource capacity, like the *Raspberry Pi* cluster, for example.

<!-- The default values for MapReduce are: -->
{{% details title="Default MapReduce values (Click to expand)" closed="true" %}}

| Parameter | Default Value | Explanation |
| --- | --- | --- |
| `mapreduce.map.memory.mb` | `-1` | The amount of memory to request from the *scheduler* for each map task. If not specified or not positive, it will be inferred from `mapreduce.map.java.opts` and `mapreduce.job.heap.memory-mb.ratio`. If java-opts is also not specified, the value will be set to `1024`. |
| `mapreduce.reduce.memory.mb` | `-1` | The amount of memory to request from the *scheduler* for each reduce task. If not specified or not positive, it will be inferred from `mapreduce.reduce.java.opts` and `mapreduce.job.heap.memory-mb.ratio`. If java-opts is also not specified, the value will be set to `1024`. |
| `mapreduce.map.java.opts` | `1024m` | The effective value is usually **-Xmx1024m**. (Hadoop can infer this value based on container memory if not explicitly set). |
| `mapreduce.reduce.java.opts` | `1024m` | The effective value is usually **-Xmx1024m**. |
| `mapreduce.job.heap.memory-mb.ratio` | `0.8` | The ratio between heap size and container size. If `-Xmx` is not specified, it will be calculated as (`mapreduce.{map|reduce}.memory.mb` * `mapreduce.heap.memory-mb.ratio`). If `-Xmx` is specified but `mapreduce.{map|reduce}.memory.mb` is not, it will be calculated as (`heapSize` / `mapreduce.heap.memory-mb.ratio`). |

{{% /details %}}

> [!NOTE]
> You can check MapReduce default values on the [official Apache Hadoop mapred-default page](https://hadoop.apache.org/docs/r3.4.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml).
> For other configurations, consult the [official Apache Hadoop documentation (3.4.0)](https://hadoop.apache.org/docs/r3.4.0/).

Access the `mapred-site.xml` file on each *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/mapred-site.xml
```

Between the `<configuration>` and `</configuration>` tags, and below previous settings, add the following properties:
```xml {filename="mapred-site.xml"}
<!-- Limits for Map and Reduce - main/nodes (recommended) -->
<property>
    <name>mapreduce.map.memory.mb</name>
    <value>512</value>
</property>
<property>
    <name>mapreduce.map.java.opts</name>
    <value>-Xmx1024m</value>
</property>
<property>
    <name>mapreduce.reduce.memory.mb</name>
    <value>512</value>
</property>
<property>
    <name>mapreduce.reduce.java.opts</name>
    <value>-Xmx1024m</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `mapreduce.map.memory.mb` | Defines the total size, in MiB, of the **YARN container** that will be requested for each **Map** task. |
| `mapreduce.map.java.opts` | Defines JVM options, primarily max heap (`-Xmx`), for the Java process running **inside** the **Map** task container. Its value must be smaller than `mapreduce.map.memory.mb`. |
| `mapreduce.reduce.memory.mb` | Defines the total size, in MiB, of the **YARN container** that will be requested for each **Reduce** task. |
| `mapreduce.reduce.java.opts` | Defines JVM options, primarily max heap (`-Xmx`), for the Java process running **inside** the **Reduce** task container. Its value must be smaller than `mapreduce.reduce.memory.mb`. |

{{% /details %}}

> [!TIP]
> It is a **best practice to keep this configuration file synchronized across all cluster nodes** (main and nodes) to ensure consistent behavior.

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 14.4 - Apply configurations:

If the cluster is running, it is necessary to restart the *cluster* for changes to take effect.

> [!NOTE]
> It is not necessary to delete the `nameNode` and `dataNode` folders to apply memory changes and format HDFS again. If you wish, you can do so, but in this case, I won't.

You can do this with the following commands:
```bash
stop-all.sh && start-all.sh
```

#### 14.5 - Important considerations:

If an error about resource limits appears when running a job, it is a sign that the limits are working, but it is necessary to understand how YARN actually allocates resources.

##### Example Scenario:

- 1 - You ran a job that asked YARN to create containers for Map tasks, and each of these containers needed 1536 MB of RAM.

- 2 - You configured the theoretical maximum limit of YARN to 2048 MB (`yarn.scheduler.maximum-allocation-mb`).

- 3 - However, on your worker nodes, you configured the total memory each node offers to YARN as only 1024 MB (`yarn.nodemanager.resource.memory-mb`).

- 4 - YARN is smart. It cannot promise a 2048 MB container if your strongest node only offers 1024 MB in total. Therefore, it reduces the effective maximum limit to the highest value your nodes can actually support, which in your case is 1024 MB.

- 5 - Your job, by asking for 1536 MB, fell exactly into this middle ground: larger than the nodes' actual capacity (1024 MB), but smaller than your theoretical configuration (2048 MB). And that's why it failed.

It is important to understand that YARN will not allocate more resources than what the nodes can actually offer. Therefore, keep this in mind when setting resource limits. Always align resource settings between the ***ResourceManager (main)*** and the ***NodeManagers (nodes)*** to avoid allocation errors.




### 15 - Configuring automatic resource detection:

Hadoop is capable of automatically detecting some *hardware* configurations, but it is necessary to indicate that this should happen; otherwise, if this setting is absent and no limits are defined, default values will be applied. Following are instructions on how to configure automatic detection.

#### 15.1 - Configure the nodes.

Access the `yarn-site.xml` file on each *node*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Between the `<configuration>` and `</configuration>` tags, and below previous settings, add the following properties:
```xml {filename="yarn-site.xml"}
<!-- Automatic resource detection - nodes -->
<property>
    <name>yarn.nodemanager.resource.detect-hardware-capabilities</name>
    <value>true</value>
</property>
<property>
    <name>yarn.nodemanager.resource.system-reserved-memory-mb</name>
    <value>2048</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `yarn.nodemanager.resource.detect-hardware-capabilities` | When `true`, the NodeManager ignores manual **memory** and **vCores** configuration and tries to detect the machine's actual values. |
| `yarn.nodemanager.resource.system-reserved-memory-mb` | When detection is on, this value is subtracted from the total RAM detected. This ensures the Operating System and Hadoop *daemons* have memory to function without competing with jobs. |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 15.2 - Adjust limits on the main machine.

If you haven't defined the limits an application can use, as shown in section [14.1.2 - Configuring resource limits for Applications](#1412---configuring-resource-limits-for-applications-main), let's define them now.

It might be confusing to think about defining a limit if it's supposed to be automatic. However, what we define as automatic is the detection of machine resources, but it is still important to define the minimum and maximum limits that an application can run on *nodes* in a way that **maintains the health** of the node and the *cluster*. The idea is that the maximum limit is no longer an arbitrary number, but a direct reflection of your hardware's actual capacity.

> [!NOTE]
> The configuration discussed here is simpler than the one presented in section [14.1.2 - Configuring resource limits for Applications](#1412---configuring-resource-limits-for-applications-main), so I strongly recommend reading it if you haven't already.

Access the `yarn-site.xml` file on *main*:
```bash
sudo nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Between the `<configuration>` and `</configuration>` tags, and below previous settings, add the following properties:
```xml {filename="yarn-site.xml"}
<!-- Limits for applications - main -->
<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>512</value> 
</property>
<property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>6144</value>
</property>
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `yarn.scheduler.minimum-allocation-mb` | The smallest unit of memory that YARN will allocate to a container. Generally, it is good to align this with the RAM of a *map/reduce* container. |
| `yarn.scheduler.maximum-allocation-mb` | The largest amount of memory that **A SINGLE** task (container) can request. This prevents a misconfigured task from trying to use all of a *node*'s memory. This should be equal to available memory of your largest node (Total RAM - Reserved Memory). |

{{% /details %}}

Press `Ctrl + S` to save, `Ctrl + X` to exit.

#### 15.3 - Apply configurations:

If the cluster is running, it is necessary to restart the *cluster* for changes to take effect.

> [!NOTE]
> It is not necessary to delete the `nameNode` and `dataNode` folders to apply memory changes and format HDFS again. If you wish, you can do so, but that won't be the case here.

You can do this with the following commands:
```bash
stop-all.sh && start-all.sh
```




### 16 - Configuring Swap:

*Swap* is an area on the hard drive that the operating system uses as an extension of RAM. It is useful when physical RAM is full, allowing the system to keep running, but with a significant performance drop. It is important to configure *Swap* to prevent the system from running out of memory and crashing. On machines with little RAM, this can be especially important.

#### 16.1 - Check Swap space:

To check if *Swap* is configured, run the following command:
```bash
sudo swapon --show
```

If there is no output, it means *Swap* is not configured.

If there is output, you will see something like:
```bash
NAME      TYPE SIZE USED PRIO
/swapfile file 2G   0B   -2
```

#### 16.2 - Create Swap file:

To create a *Swap* file, run the following commands:
```bash
sudo fallocate -l 2G /swapfile
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `-l 2G` | Specifies the file size (*Length*). 2G for 2 Gigabytes. You can use 4G, 8G, etc. |
| `/swapfile` | Is the path where the *Swap* file will be created. |

{{% /details %}}

If `fallocate` is not available, you can use the following alternative command:
```bash
sudo dd if=/dev/zero of=/swapfile bs=1G count=2
```

{{% details title="Parameter explanation (Click to expand)" closed="true" %}}

| Parameter | Function |
| --- | --- |
| `if=/dev/zero` | Specifies that the file will be filled with zeros. |
| `of=/swapfile` | Is the path where the *Swap* file will be created. |
| `bs=1G` | Specifies the block size as 1 Gibibyte. |
| `count=2` | Specifies that 2 blocks of 1 Gibibyte will be created. |

{{% /details %}}

##### 16.2.1 - Set Swap file permissions:

For security, only the root user should have permission to read and write to the swap file.
```bash
sudo chmod 600 /swapfile
```

##### 16.2.2 - Format the Swap file and enable it:

This command prepares the file to be used as swap:
```bash
sudo mkswap /swapfile
```

Now, tell the system to start using this file as swap memory:
```bash
sudo swapon /swapfile
```

#### 16.3 - Making Swap permanent:

To make the *Swap* space permanent, add the following line to the `/etc/fstab` file:
```bash
sudo nano /etc/fstab
```

Add the following line to the end of the file:
```bash
/swapfile none swap sw 0 0
```

#### 16.4 - Checking Swap space again:

After creating the *Swap* space, run the command again:
```bash
sudo swapon --show
```

You should see output similar to:
```bash
NAME      TYPE SIZE USED PRIO
/swapfile file 2G   0B   -2
```

It is also possible to check *Swap* usage with the command:
```bash
free -h
```

The output of `free -h` on the "Swap" line should now show the total combined capacity (the old swap, if any, plus the 2 GB added).




### 17 - Installing the cluster monitoring system (Zabbix and Grafana):

* [Installing Zabbix and Grafana to monitor the Hadoop cluster](/articles/2025/08/1-zabbix-and-grafana)




### 18 - Automating the cluster configuration process with Ansible:

* [Automating server configuration with Ansible](/articles/2026/01/1-ansible)




### 19 - Benchmark tests to measure cluster performance:

* [Performing benchmark tests with Hadoop (coming soon)](#)

<!-- Imagens -->

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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>

[image4]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAMEAAACACAYAAACyRg1VAAAAAXNSR0IArs4c6QAAHLNJREFUeF7tXX2MVsX1Po0i7Gr52F1KEajEgEDcovKhVirGiglUMZKQIoSmXW2yBgoxqNBUg4ZIKVQtCVlSkypNakCbbTRYqo1SgpaIIG6xa6GyMVIESyWLFkQqLf3lGX7nehjmzpl7931334+5/8C7987MmTPnmTkz88yZL9XV1f2PqvRZt26dqXltbS01NjZSXV0d3X777fTGG29UqUayVbtS9PelagbBrl276OKLL6bPPvuM3nvvPVq/fj099dRT2Syhir+uFP1VNQiq2H5j1YUGIggqwBy2bdtG7777LjU1NVVAbbq/Cl4QrHtrOv3xN+/Tr3/6l4JIVuj8CiJUATL54c8m0JirG2hV8+u0f+8nBcgxWxZdBcE111xDGzZsoJ07d9KsWbOyFV4BX1c0CPoP7ENzFzfS+JsGm6batflDenpVO3380cmCNt29LdfSpY39ux0Et912G61YscJM6PEcOnSIWlpaMs9rGASYF02ZMqWguimHzCoaBNxD/7ZlL9VceD59+/sjaPuLBws2svV0A7/zzjtUU1NDa9eupf79+9OMGTPoueeeowcffLCnRSur8lUQbPvdB3TJqL40dGRf6tjdSS2LdyU96Xd/9HX61neGmwrj3eZn36ftLx1MFHBL0whjeLVf7mXcKnwr3atLRvejuUsaacQVdXTi2Clq23qYfrm0LUmvvYd7BflGjaujhotrjQxPr2w3LglGgZ//4WZqXbOHRk9oMD31nh1HqF9Db1retI2unTqEmn8yzowOcGUADsjX/vpH9Nj87UYGrX4PrJtkZOenadwLZzW+Sz6pP3zcFVfkww8/NKtakyZNymV0d955Jy1fvjxJa+eF/F999dVk+RjvFy1alCwhYyRauHAhXX755UkeL730UtnNTVQQsHH2q+9Njd8YSJvW7aPWNXtNpWFII688YwRjv/kVY4gPz3nVGCEM+OH1k41xw8C+NqqvMRgJAjYiGDLnD6PdtK7D5Km9h5GxfH1qzjNuDxsxG/kTP37LGDvyvXLyIAOO+2/dnIAA8kD2f+z/1JSJOrIx++qHbwHyukE1Sd1cIPDpj0Hw/PPPE3p12xX591fvp9O9BtN5n75JF3Sup8+GPU6nGr5HfdvqjayYC1x66aUEw1u6dCkdOHAgExgAwLvvvtukmTp16jmAAgiwfIy5AvZSJkyYcJacPBLh/YkTJ0w+GIk2btyYSY6e/lgFgewZXRNbGM1F/S6g+sE1NHPBmAQkMBD8xqQahsY9M4OAQQIAcO/fsnUqHXrvmOmptfdQHOSR8sH9ARBgjDYI8Dfu2e33N80aTn//279MW2A0kMacVj/ZcDJf+XdbvrSFARijvUH3+cC76eTQL3rp849vo9O9BtHp3iMSEAwbNoywYYWeGMa6devW3L2wa1TB3yQ4X3jhBQOEwYPPzLEwMqBcyPDss89mBmFPGz+Xr4JA9tyyEWHU81eNP8sdQKb8PRsGemJ2kWR6NlJ7ZEAe0l1Je88gkO/TjBwjQVYQaPULBUGa/jQD4F7/or+Op/9ceB2dGvgD+m/tFdSrs5Vq9jeflVy6JXndkTQQyPxg7BgxGAT33XcfzZ4922w44gFgHnjggbLbcc8NArun556bG71URgK4Z3DL2B2CSxYyEmj1KyQIXCOBBhLXe/TMMOY8c4Q8IwHLgBGpubmZ7rrrLnrzzTdp+vTpecTvsTS5QTBzwWi6pWmkcX8+2HeMJk0fZvxpnpyiRjC+I4dO0Nt/+mfR5wQob9KtQ8+ZEwAEcMt4Yuxyl1zukFY/zHsw8uCR8x385n0V2/1xuUMAQNqcwGcV7Art2bOH2tra6Prrrze9dF4jTAMBzwkgy+TJk89yj1555ZXEBWpoaDCuUt7yewwBRJQbBNJdwOTv97/qMMYmXSJ7deiyq+ro3bbOxEi01R/tPa++DBpWa9wyuToEOTDHSFsSlXMGFwhC6ofyXQ/PKUJBkGejyt4jYB/9jjvuyOWbp4EAq0NDhw41E3B7dQi/sUSLB2CBOzRv3rxc5ZcsCHpSsJCytR1oniPARdv35046/snnZqSotKerO8YYVXbs2GGWQ+WOMYCRd45RTjoua+6QBgI0BIDAy7cYKTDprrQnLwgw0YUbg14eu86Y1EoWbQRBGVhKCAjKoBo9JiLcGXajXDTyCIIea5pYcNRA92qgrN2h7lVVLK1SNRBBUKktG+sVrAH/Eun/7xDK3ORqwSOPPELTpk1LdgyxRiyXyPD+xhtvJCzb3XzzzYasJSdf2ntfLWzyF77FEh0OlmTl0ARrq4Q+lPW/+uqradmyZWafAA/v6JaQuCUtShAIYPj8vPbaa2YFgdmPR48epbfffpuwzAYOS2trKy1YsMB8Dq4J/obVB240CQLtfQgIALwjR44Yghc2c7rCqizplrKEY312dnbSY489RnPmzDHGj1WeCIJsLamC4IYbbjBG7Hrkdj/vfEoQYOkOD7bxXSDQ3oeAQIKKWZVsBNhQWrJkiZGfdz7lOrhGBUadHn/8cWd6rg+PjGm/UQcYKXNs5I4qOg6cBUBHgU0n+1CMT34uD6MfRj6MBu3t7aYj4Poz1wcyoHN48sknz1kCtanScrNNS4+RHOCD7NADZMB+Ax/zDJHfp59sppz/ay8IYFSsUFTU3jFEsTAUbNmjkU+ePGlcH3ZHmI8Ow3OBQHufBwSQl0HLVF+wK3lbX4JUowIzqGAo9fX1xlhhSDi0ooEABs4uIIy7o6MjyQM8G9CNOX8ezWBE0OH48eNN1X3yy/JR1oABA8yILAlu+AZtgwd52yFlJFWa6yf140vPnR6zV8eOHWvcYuku++QP0U9+s86WUgVBnz59jHLZiGx3A/wRGAca+sUXX0xONaEXeOKJJxL3yAaB9l6rBudnu0PcCJw/fsOFw3PvvffSxx9/nBDMfFRgbmS5iyoJahoIWH5JR+aeceXKlXT48GHDGZL5o0zoEZ2IJr/UJ+ZarAcJAsiAfHhDDAQ3aeQ2Vdq1L5CWHqMA8sMo9+ijjxp3GKNAFv1DvjT9dOeZhEyrQzafnBsaCli8eDHNnDkz6SlBs4XRceNIghUUiB7K914LgOWaGEuAut6zW8AsSx8V2DZypHW5b2nukASBi3rgyl8CX5NfggDHKnfv3k1DhgxJRgK0yTPPPHOOKytlsY1e/tbSs6sk3VGZXpNf04/WCRbyfSoIuNfasmVL0rungUBWiA1R+pO2wFAcsx5dlbG3713f2CMLuxYcQY57UpsP48rLRQXOOhIw6G2DT9t11dijmvwu91Ly/e2emssLBYGWPnQk0PRfCrvS3pGAt9Xl8To5sWPOCgzrsssuM72OHG5dPVuagbsaNcucQLpHzGe3fW7kJ4//aVRg35yAh3LbJ4bOMC+Cfww3kV0VlGtHt+P8kQZxg3geAPeCRx7olEdTKb8GgjVr1piRGe0BqjUmsJBHzut8I4GWHrLAncPqFNwgXh2UIPPpH6DU9FPI3t6XlxcE6I0eeuih1H0ADsOHAlgZaQGgNCPX3tuVSFttgtHwaGCvviAP9mHZNfJRgX2rQ0jPqyP4P1Z54N7hSRvp7KVLyAeDnzhxYkJJlj2nT34NBNKdAVDBDYIPj4cNNdQdSktvrw5hPiJB4JPf5Sn01NJupjlBdyEzllN+GnC5W+VSiwiCcmmpEpQzRqUuwUaJInWvBmJU6u7VdywtaqBoGojuUNFUGzMuFw1EEHRjS5XCmng3VrfgRRVLfyqBjum5XCO5BIYNIrzn9Wc73o29TGdHSHMtk2U52G0v4ZZ68KdiNaLL2uSOreQq4ds8cYmyWnQxyi+W/oJA4KJSQynYDMEaMtZ3JSeHFSY3mzhsBxPQ8A2DIC1/TfGSAMexMrUdSi3PYr4vViNqIOCORdI+illP5C1BUKjyi6U/FQQ+KrU0druHSaMdyBj4AEFI/mkNZsfKsSO5aVRqjcqsbZYxC5Yv/ePdYmbRaptJPqqyNCIXFRuxP0GblhcNSloLpwchDw+YqTYIQsrHbjVGeoziGPVlbNIQqrSvfE2/mv609g0Fusoi1ajUPCLYIJA7mjj0AtYkuPmS6hxC1fZVBLQH7BCjgVxRmTUqtUZlDqVNgFZiU5FtqjHTSqS756Mqa1Rj6EWydPEbIOROhvWPkRFgAZUahDrZTr7yJcEPaQF4PAAE20QI1dtXvk+/IfrT2rdgINCo1CEgAH+GBYbRshKhhJD80yrDtAM0MtyyTZs2JafaNCqya6SSVGaNQAeZfFRkjWDGdfJRne0yJBUbVGPolM8fcH1d5x0AQBAhcdTV7qzSypedGOgWzG1iqramXwkiV/mafjX9aeWHAgDfZVodSmORunxNqcSrrrqKXn75ZXPKS44EtqAaSzWtYlAoDrogFiYT+DQqbyiV2SaESSPycW80qrFGVea6+vxg6RLhngEYKLtHsn7Hjx+n4cOHJ9c6YWKsla+BIIt+XeVrVHVNf1r5BQFBFiq1CwQupKPn4gl0lvxDK2TnD3chbaKsUZm1nop76TRqstaTaVTlEBDII63XXXdd4vvjP9LIwGBdvXp14tIABFr5GghCqd7Qj6t8Tb+a/rTyQ21GHQk0KjVzR+Az4gGl9uDBg8n5A82n1vL3VQRKBDUZeWDO4YqKrFGpQ6nMruOVGgi4kZlda88JQqjKIVRjUBfgUuLopD3fgBvKf0MHgW/4vIdWPpcNRqzLHWI3WKN6p5Uv07v0q+lPK79gINCo1DxZkgXK013aPoGWvwYCPgSP71xRkTUqtUZl1lYvfO4QZLJXN3AOF0dVQTfXqM5Ib+/RuKjGbMz4nvcD7JEA5bGrye2jlY8jqRwiJw0EIVRvBoFdPmTU9OvTH9Jr7RsKhExzgtBM43fdqwFeYOAD+t1bevmXFkFQpm2I3Xq4LOxmhRxJLdOqFl3sCIKiq7g4BfDKEDajcMkHH8ksTmmVnWsEQWW3b6xdgAYiCAKUFD+pbA1EEFR2+8baBWhAJdD5qNS8TIXAW+PGjTM8HjlB81Gt03b87KW+gDokm0D4tloC8oboJX4TpoEgEKRRnbFOi9gziLHD8Taxds/R43xUaxknCJtdeHjTDaseWR65aeeidGfJK35bfRpQQeCjOvMGiDwj4FKhxi3Czm/aDYpZmsQux0V1XrRoUQJSLSp1lrLjt+WrgdxUajZa8M3BDUFYRUmZkCpJ4xaB9PWLX/zCGCVzReSuZ1a1ukDAIdn50I3kw2tRqbOWH78vTw2oIEijOrM7Ax9c3l/gIqyFnGjC2QDQArK6Qj6w2VRnm6Xqi0pdns0Zpc6jgUyrQ66TS3xFEgrn+YHNcdFAkMcV4liWMr6naySQpDIZsBby+qJS51FmTFOeGsgdlZpHAjkfyHLeQKorjyvExDHpPoFRiUMmfJBcGwlYBldU6vJszih1Hg10KSq1PN7mujNMo1qzwHxMMu1aKFfFbKqtKyqyvIkFeeAEmpwTaFGp8yg0pik/DXQpKrU86Iyq27dHalRrpJETbA6pHqpGuDOgCYMnzyHSZVRsXh3iSBf2dVP47YtKHSpH/K68NZBpTlBuVS1WiI5y00OU16+BCIJoIVWvgQiCqjeBqICKBkFs3qiBEA1EEIRoKX5T0RqIICih5o0T+Z5pDC8IsImFqGW4jZFvZ5dUaS1aRDGp1DYV275MPFSdeQ2vnKIuh+qiWr/zggA7wODyYBPLvi0R6/uIAYoH/6bF/UmLWt1VKrWdHjRsO/ZOSKMWAgSlHnU5RA/V/I1KoINyQEOwQeCiTYC2MGDAgHNuUS8GldqWB3La5fuiLnPgLbvxXVwjfIONNVBEmKsUEvU5a1wd+wrUQkVdrmYDD6m7FwS84zpr1qxUEMh7gfkitxACHQykK1RqFwiYT8Qumy/qMly9IUOGmABX8jJtBJ2Shg6KOB4eaexYn+UQdTnEEKr5Gy+BTob+to3ODp0Nd4hp0DYHSGORogGyUqldIHD9LSTqs+92HC1qM9KWetTlajbwkLqnggCTWtzQjkMzdqxP9KJ8EGbOnDmGf4MeccSIEWexOFkADQR5qNTaSIAo2IjHbwPSNvi0OUFo1GbkV+pRl0MMoZq/SQWB6z4xVpQr2hnToV1HLTUQ5KFSu0DAfj7cMS3qMtcFIADQbfKell6L+lxKUZer2cBD6h68T+AyOjT03LlzacyYMcYVso2pmFTqtNUhPtmmRV3mYABww1h2jHh8RFRLz1GbyyHqcoghVPM3XQIB3zmGiSWMwQ4FWEwqtWufgCM+o0G1qMtMubYP22eN2lwOUZer2cBD6h4MgpDM4jdRA+WogQiCcmy1KHNBNRBBUFB1xszKUQMRBOXYalHmgmoggqCg6oyZlaMGIghKqNXykvlKqAplKUrRqdSzZ882AXs5HCJ4SPxoVG2fRotBZc7SgsUoP4IgSwsU7tvcVGqIwFewgkrNcX94s0qGacRt6Hy3lh2sK42qrVVRGmGhqMxamfJ9McqPIMjSAoX7NjeVGiK4rjCVwa3Q0+OmeTzMD0q7Id61Ix0yEuAwDR7c3GjTM3xUarnjDCACyGCUSvl9VOZIpS6cEfZ0Trmp1BCcd4z5UA2M6Z577qGNGzeeUy++gby1tZUWLFhg3vuo2ppi2Ah9VGYflVpyf0CT5t1t1IGp4DLCHh8aYvlDyvddZm6zcO3LvlF/X/mafuL7cA3kplKjCBjC/Pnzjc+PBwYJ9umBAwfOkQC3LU6cOJFw3wHe26DIOxKkUZlZAI0KnXZjO8uH/HHGAA9YtXwJiARRpFKHG1wpfpmbSo3KIAo1k+bQs+EwCnrUKVOmnFVXF8M0hKod4g6lUZlDqdBpILC5SSwLc4skCCKVuhRNO1ym3FRqFLF8+XKSJ8sklZlFYL/cplhnpWrbVdKozKFUaG0kcN23wKMg6g8Q4pKS1atXJy4VjqNGKnW4Efb0l8H7BGknyzo7O2nHjh3JQXv7JhgcfsffpIskg+ayArriDiE/lIGyuKcOpUKngQByMaj5YBH+BoPHnEeC0FW+TA8g1dfXG8o2dwZ2VG3XnMBXfk8bTiWVnxsEUILrkgsYBBu8i0qNdPYZZNmzug7suBRuGyHfjZCVCu0DAVyqtWvXGuPl6NU88mnlQ+asB+3Hjh1LNh08rfxKMsKerkswCHpa0Fh+1ECxNBBBUCzNxnzLRgMRBGXTVFHQYmkggqBYmo35lo0GIgjKpqmioMXSQASB0GwksBXLzEo7Xy8IXBtakgCnUaHxftq0aQmtAuvt8+bNS5ZQtajWPtX1NJVZlg/u0bJlywwBL20JuLTNoLqlCwIBDJ8fGavTF7Uaa+QbNmygo0ePmrVvplozAS0kqnUoCApFpc4yEjAIsFmIvQNE4sP+BzbsXPsg1W1mpV17FQQgvKXdLyypy2nBuTjIFe+Q2ixMSadIi2rtUmFPU5m5fN4Nx2jQ3t5u7kpmEPio3KgTs2gbGxuT3W7cBcGbjVp6jLQcBhMdAWTA7r2MqbRkyRLTfvahJjmSAcR8+MkVja+0Tbjr0qnnCbhBsWNq3wMcQoWG8SOyM5SM2+a5kbkRQqJa+0DQU1Gh5Y4xRjWEpMeIB5eIdeajcjMI2DiZViGp5r70NhUbu81g80p31UfFhsx88QrOZHR0dCTUjubmZicdvuvmVpo5qCDo06ePaVzm0zMtIZQKzWEO7ZtkskS19oGgp6JCy5EPRDrmF0kQQG5fVGx0IpJr5XLH0tLbBEH70JJGBWedShn4ENHKlSsjCNLwyvwc9HRZqNBooMWLF9PMmTMTAhnKkMO5L6q1BoKeoDJLEMyYMYN2796d3HcA/WhUbh4JZM8tQaClZ1dJcq1keo0KLkHgC01fmn13YaXyHqqBP7lly5bkiKQEQR4qNBqJRxK7Gr6o1hoIeoLK7JoDsU5Co2K7jqeyQWpU8NCRII0KHkHwhVV53SE+SL9z506qra2lCRMmOMOYIzuXUWDijEP2eJgqLH1eLap1yOpQT0WF1kCgUbmxYOADgZYeusGhJqay8+qbfYYbk2IXFRy658jaeI+OhG/oKWw/W/q5eUGQZR3fZRR8fRPUwI0lzxJoUa2zgMCmUiNtManMGghComL7QBCS3l4dsu8881HBXSN5tS7txh3j0u+ogiTkhYZq9++DlGV9FEGQR2slkoYvQYGrynsNfLFgiYhYFmJEEJRFM7mFZHcTew2Yv61fv75q/fquNGMEQVe0F9NWhAYiCCqgGXkVzhXAoAKqV/Qq+LlDb02nP/7mffr1T/9SEEHWFTi/gghVgEx++LMJNObqBlrV/Drt3/tJAXLMlkUEQTZ92V9XNAj6D+xDcxc30vibBpt679r8IT29qp0+/uhk17Rmpb635Vq6tLF/t4MAS9grVqww5Ds8oKa0tLTEeUHG1q1oEHAP/duWvVRz4fn07e+PoO0vHizYyJZR1wX/nAlyCMvSv39/An0Dm14cBLngBVZohioItv3uA7pkVF8aOrIvdezupJbFu5Ke9Ls/+jp96zvDjWrwbvOz79P2lw4mqrqlaYQxvNov9zJuFb6V7tUlo/vR3CWNNOKKOjpx7BS1bT1Mv1zalqTX3sO9gnyjxtVRw8W1RoanV7YblwSjwM//cDO1rtlDoyc0mJ56z44j1K+hNy1v2kbXTh1CzT8ZZ0YHuDIAB+Rrf/0jemz+diODVr8H1k0ysvPTNO6Fs8zEJZ/UHz7mcxfYlZd3N4TYm4+GEpI+fnNGAyoI2Dj71femxm8MpE3r9lHrmr0mMQxp5JVnjGDsN79iDPHhOa8aI4QBP7x+sjFuGNjXRvU1BiNBwEYEQ+b8YbSb1nWYPLX3MDKWr0/NecbtYSNmI3/ix28ZY0e+V04eZMBx/62bExBAHsj+j/2fmjJRRzZmX/3wLUBeN6gmqZsLBD79MQhAf5BsUjbOf3/1fjrdazCd9+mbdEHnevps2ON0quF71Let3nzCEeqwQbZ06VJnIORo6LoGVBDIntE1sYXRXNTvAqofXEMzF4xJQAIDwW9MqmFo3DMzCBgkAAD3/i1bp9Kh946Znlp7j6pBHikf3B8AAcZogwB/457dfn/TrOH097/9y2gLo4E05rT6SdXKfOXfbfnSFgYwGvDhI07/+cC76eTQ5Ul25x/fRqd7DaLTvUckIAAtAhtm4ABhrwAh8uMKkW709hcqCGTPLRsRRj1/1fiz3AFkzt+zYaAnZhdJpmcjtUcG5CHdlbT3DAL5Ps3IMRJkBYFWv1AQpOlPayru9S/663j6z4XX0amBP6D/1l5BvTpbqWZ/81nJMUFeuHChAUOkTWiaPfd9bhDYPT333NzopTISwD2DW8buEFyykJFAq18hQeAaCbI35ZnrszBPQFTs+IRrIDcIZi4YTbc0jTTuzwf7jtGk6cOMP82TU4gA4zty6AS9/ad/Fn1OgPIm3Tr0nDkBQAC3jCfGLnfJ5Q5p9cO8ByMPHjnfwW/eV7HdH5c7xMQ315zA14zsCu3Zs4fa2trMEVawSKvxjHC4ubu/zA0C6S5g8vf7X3UYY5Mukb06dNlVdfRuW2diJNrqj/aeV18GDas1bplcHYIcmGOkLYnKOYMLBCH1Q/muh+cUoSBAVI6sq0P2HgHkwEggD+r7jONw55mFgLRnUN2FXbWtsklf1rQJbQea5whw0fb9uZOOf/K5GSkq7cmzYxxB8IUVVDQIUE0AgZdvMVJg0l1pTwRB11q04kHQNfVUbuo4ElTISFC5Jhpr1p0aKOuRoDsVFcuqXA1EEFRu28aaBWoggiBQUfGzytVABEHltm2sWaAGIggCFRU/q1wN/B9myP3wN4GxGAAAAABJRU5ErkJggg==>

