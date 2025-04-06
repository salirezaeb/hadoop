
# 🐘 Multi-Node Hadoop Cluster Deployment on Ubuntu 20.04 (HDFS + YARN)

This project documents the full manual setup of a multi-node Hadoop cluster using Ubuntu 20.04 virtual machines. The cluster is designed to enable distributed storage (HDFS) and resource management (YARN), following industry standards for system configuration and networked operation in distributed environments.

---

## 📌 Cluster Architecture

- **Master Node** (`node-master`):
  - Hadoop NameNode: Manages the metadata and namespace of HDFS
  - YARN ResourceManager: Allocates cluster resources for applications

- **Worker Nodes** (`node-worker-1`, `node-worker-2`, ...):
  - Hadoop DataNode: Stores actual data blocks
  - YARN NodeManager: Manages containers and monitors their resource usage

> 💡 The number of worker nodes can be scaled horizontally depending on cluster size and application needs.

---

## ⚙️ Technology Stack

| Component           | Version       |
|--------------------|---------------|
| Operating System    | Ubuntu 20.04 LTS |
| Hadoop              | 3.2.1         |
| Java                | OpenJDK 8     |
| Communication       | SSH, PDSH     |

---

## 🧱 Setup Guide

### 🔐 Step 1: Install SSH and PDSH (Remote Shell Tool)

Install required packages on all nodes:

```bash
sudo apt update
sudo apt install ssh pdsh -y
```

Enable SSH mode for PDSH in your shell environment:

```bash
echo 'export PDSH_RCMD_TYPE=ssh' >> ~/.bashrc
source ~/.bashrc
```

Generate an SSH key on the master node:

```bash
ssh-keygen -t rsa -P ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

Test passwordless access:

```bash
ssh localhost
```

> ⚠️ Ensure SSH server is enabled and port 22 is open on all machines.

---

### ☕ Step 2: Install Java (JDK 8)

Install Java Development Kit (required by Hadoop):

```bash
sudo apt install openjdk-8-jdk -y
```

---

### 📦 Step 3: Download and Install Hadoop

Download and extract Hadoop binaries:

```bash
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.2.2/hadoop-3.2.1.tar.gz
tar -xzf hadoop-3.2.1.tar.gz
sudo mv hadoop-3.2.1 /usr/local/hadoop
```

---

### 🧩 Step 4: Set Up Environment Variables

Edit system environment variables:

```bash
sudo nano /etc/environment
```

Append:

```bash
JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
PATH="/usr/local/hadoop/bin:/usr/local/hadoop/sbin:$PATH"
```

Apply changes:

```bash
source /etc/environment
```

Set Java path in Hadoop's environment script:

```bash
sudo nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Ensure it contains:

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

---

### 👤 Step 5: Create Hadoop User (Recommended)

It's good practice to avoid running Hadoop as root:

```bash
sudo adduser h-user
sudo usermod -aG sudo h-user
sudo chown -R h-user:root /usr/local/hadoop
sudo chmod -R g+rwx /usr/local/hadoop
```

---

### 🧭 Step 6: Set Hostnames and Static IP Mapping

Each machine should have a consistent hostname:
- Master: `node-master`
- Workers: `node-worker-1`, `node-worker-2`, etc.

Edit each node’s hostname:

```bash
sudo nano /etc/hostname
```

Update the `/etc/hosts` file on all nodes to map hostnames to their respective static private IPs:

```bash
sudo nano /etc/hosts
```

Example entry (replace with actual IPs of your network):

```text
<master-private-ip> node-master
<worker1-private-ip> node-worker-1
<worker2-private-ip> node-worker-2
```

> ✅ Use static IP addresses in the private network (e.g., `192.168.x.x`) assigned by your virtualization platform or cloud VPC.

---

### 🔑 Step 7: Configure Passwordless SSH Between Nodes

Switch to `h-user` and generate SSH keys:

```bash
ssh-keygen -t rsa
```

Copy the SSH key to all other nodes:

```bash
ssh-copy-id h-user@node-master
ssh-copy-id h-user@node-worker-1
ssh-copy-id h-user@node-worker-2
```

Test passwordless SSH:

```bash
pdsh -w node-worker-1,node-worker-2 hostname
```

---

### 📂 Step 8: Configure HDFS

Edit `core-site.xml`:

```xml
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://node-master:9000</value>
</property>
```

Edit `hdfs-site.xml`:

```xml
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

Edit the `workers` file to list all worker hostnames:

```text
node-worker-1
node-worker-2
```

Distribute the Hadoop configuration to all nodes:

```bash
scp -r /usr/local/hadoop/etc/hadoop/* node-worker-1:/usr/local/hadoop/etc/hadoop/
scp -r /usr/local/hadoop/etc/hadoop/* node-worker-2:/usr/local/hadoop/etc/hadoop/
```

---

### 🧹 Step 9: Format HDFS and Launch Daemons

Format the NameNode (only once on the master):

```bash
hdfs namenode -format
```

Start the HDFS services:

```bash
start-dfs.sh
```

Check service processes:

```bash
jps
```

Expected:
- Master: NameNode
- Workers: DataNode

---

### 🧠 Step 10: Configure and Start YARN

Add the following to `.bashrc`:

```bash
export HADOOP_HOME="/usr/local/hadoop"
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
```

Reload:

```bash
source ~/.bashrc
```

Edit `yarn-site.xml`:

```xml
<property>
  <name>yarn.resourcemanager.hostname</name>
  <value>node-master</value>
</property>
```

Start YARN services:

```bash
start-yarn.sh
```

Use `jps` again to confirm:
- Master: ResourceManager
- Workers: NodeManager

---

## 🌐 Web Interfaces

| Component            | Address                        |
|---------------------|---------------------------------|
| HDFS NameNode UI     | `http://node-master:9870`       |
| YARN ResourceManager | `http://node-master:8088`       |

---

## 📊 Monitoring (ResourceManager UI)

Via YARN UI, you can track:

- Node status (active/lost/unhealthy)
- Available and used containers
- Memory/CPU usage
- Last heartbeat
- Logs per node

---

## ✅ Final Cluster Validation

On all nodes, run:

```bash
jps
```

Verify expected processes:
- Master: `NameNode`, `ResourceManager`
- Workers: `DataNode`, `NodeManager`

---

## 📄 License

This project is licensed under the [GNU GPL v2.0](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)

---

## 🙌 Author

This setup was implemented manually for hands-on experience with big data infrastructure, emphasizing distributed file systems, resource allocation, and multi-node coordination using Hadoop.
