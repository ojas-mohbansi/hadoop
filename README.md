# Apache Hadoop 3.5.0 on WSL 2

Step-by-step configuration, startup, verification, and shell aliases.

This guide follows the WSL 2 methodology contained in the supplied "Installing Apache Hadoop on Windows" PDF. The native Windows `winutils.exe`/`hadoop.dll` route and the Python/PySide6 application material are intentionally excluded. The practical target is a single-node Hadoop installation inside WSL 2 Ubuntu, with repeatable start/stop/status aliases.

## Table of contents

1. [Methodology used](#1-methodology-used)
2. [Install WSL 2 and Ubuntu](#2-1-install-wsl-2-and-ubuntu)
3. [Install Java 17 and basic tools](#3-2-install-java-17-and-basic-tools)
4. [Download Hadoop 3.5.0](#4-3-download-hadoop-350)
5. [Install Hadoop under /usr/local/hadoop](#5-4-install-hadoop-under-usrlocalhadoop)
6. [Add the Hadoop environment to .bashrc](#6-5-add-the-hadoop-environment-to-bashrc)
7. [Configure passwordless SSH for localhost](#7-6-configure-passwordless-ssh-for-localhost)
8. [Create HDFS data directories](#8-7-create-hdfs-data-directories)
9. [Configure Hadoop XML files](#9-8-configure-hadoop-xml-files)
10. [Format the NameNode once](#10-9-format-the-namenode-once)
11. [Start the Hadoop cluster](#11-10-start-the-hadoop-cluster)
12. [Verify the Hadoop web interfaces](#12-11-verify-the-hadoop-web-interfaces)
13. [Add one-word Hadoop aliases to .bashrc](#13-12-add-one-word-hadoop-aliases-to-bashrc)
14. [Optional PowerShell shortcuts to WSL aliases](#14-13-optional-powershell-shortcuts-to-wsl-aliases)
15. [Routine startup after reboot](#15-14-routine-startup-after-reboot)
16. [Clean shutdown and stale PID troubleshooting](#16-15-clean-shutdown-and-stale-pid-troubleshooting)
17. [Final checklist](#17-16-final-checklist)
18. [Scope note](#scope-note)

---

## 1. Methodology used

The supplied procedure evolves from a native Windows installation toward WSL 2 as the recommended approach. The WSL path keeps Hadoop inside a normal Linux environment, avoiding Windows-specific helper binaries and Windows path configuration. The later setup in the source uses Hadoop 3.5.0 with OpenJDK 17, installs Hadoop under `/usr/local/hadoop`, configures the four Hadoop XML files, formats the NameNode once, starts HDFS and YARN, verifies with `jps`, and exposes the web interfaces on localhost.

## 2. 1. Install WSL 2 and Ubuntu

Perform the Windows-side installation from an elevated PowerShell window.

1. Open **PowerShell as Administrator**.
2. Run:

```powershell
wsl --install
```

3. Restart Windows if prompted.
4. Launch **Ubuntu** from the Start menu.
5. Create your Linux username and password when Ubuntu asks for them.

**Result.** You should now have an Ubuntu shell running under WSL 2. All remaining Hadoop commands in this document are executed inside that Linux terminal unless a step explicitly says **PowerShell**.

## 3. 2. Install Java 17 and basic tools

The source document records a Java-version mismatch with Hadoop 3.5.0 and then switches the environment to OpenJDK 17. Follow that working configuration for the WSL installation.

Inside Ubuntu:

```bash
sudo apt update
sudo apt install openjdk-17-jdk wget gnupg -y
```

Check Java:

```bash
java -version
```

Locate the Java installation directory:

```bash
readlink -f /usr/bin/java | sed "s:/bin/java::"
```

For the source setup, this resolves to a path such as:

```text
/usr/lib/jvm/java-17-openjdk-amd64
```

## 4. 3. Download Hadoop 3.5.0

Create a temporary working directory and download the official Hadoop 3.5.0 release archive.

```bash
mkdir -p ~/hadoop-setup
cd ~/hadoop-setup

wget https://dlcdn.apache.org/hadoop/common/hadoop-3.5.0/hadoop-3.5.0.tar.gz
```

For the verification workflow used in the source, also download the SHA-512 file, the PGP signature, and the Apache release keys:

```bash
wget https://downloads.apache.org/hadoop/common/hadoop-3.5.0/hadoop-3.5.0.tar.gz.sha512
wget https://downloads.apache.org/hadoop/common/hadoop-3.5.0/hadoop-3.5.0.tar.gz.asc
wget https://downloads.apache.org/hadoop/common/KEYS
```

Verify the archive checksum:

```bash
sha512sum hadoop-3.5.0.tar.gz
cat hadoop-3.5.0.tar.gz.sha512
```

The two SHA-512 values should match exactly.

Verify the PGP signature:

```bash
gpg --import KEYS
gpg --verify hadoop-3.5.0.tar.gz.asc hadoop-3.5.0.tar.gz
```

## 5. 4. Install Hadoop under /usr/local/hadoop

Extract the archive and move it into the standard location used by the source setup.

```bash
cd ~/hadoop-setup
tar -xvzf hadoop-3.5.0.tar.gz
sudo mv hadoop-3.5.0 /usr/local/hadoop
sudo chown -R $USER:$USER /usr/local/hadoop
```

Confirm the binary can execute:

```bash
/usr/local/hadoop/bin/hadoop version
```

A successful result should report Hadoop 3.5.0 and its build metadata.

Remove the temporary download directory after verification:

```bash
cd ~
rm -rf ~/hadoop-setup
```

## 6. 5. Add the Hadoop environment to .bashrc

Open the shell profile:

```bash
nano ~/.bashrc
```

Append the following block. The Java path should match the path returned by the lookup command above.

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPPED=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

Save the file and reload it:

```bash
source ~/.bashrc
```

Verify the environment:

```bash
echo $JAVA_HOME
echo $HADOOP_HOME
hadoop version
```

## 7. 6. Configure passwordless SSH for localhost

The source procedure uses local OpenSSH so that Hadoop scripts can manage the local daemons.

Install the server:

```bash
sudo apt install openssh-server -y
sudo service ssh start
```

Generate an RSA key without a passphrase:

```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
```

Authorize the key and set permissions:

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Test localhost SSH:

```bash
ssh localhost
```

Type `yes` if prompted. Once the connection opens, run `exit` to return to the main shell.

## 8. 7. Create HDFS data directories

Create the local directories used by the NameNode and DataNode.

```bash
mkdir -p /usr/local/hadoop/data/namenode
mkdir -p /usr/local/hadoop/data/datanode
```

## 9. 8. Configure Hadoop XML files

All four files live in `/usr/local/hadoop/etc/hadoop`. The source methodology configures a single-node cluster as follows.

### 9.1 core-site.xml

Open:

```bash
nano /usr/local/hadoop/etc/hadoop/core-site.xml
```

Use:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

### 9.2 hdfs-site.xml

Use:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///usr/local/hadoop/data/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///usr/local/hadoop/data/datanode</value>
    </property>
</configuration>
```

### 9.3 mapred-site.xml

Use:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
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
</configuration>
```

### 9.4 yarn-site.xml

Use:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

### 9.5 Set Java inside hadoop-env.sh

Open:

```bash
nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Ensure it contains:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

## 10. 9. Format the NameNode once

This operation initializes the HDFS metadata. The source explicitly warns to perform it only once for a fresh cluster because repeating it can remove existing cluster metadata.

```bash
hdfs namenode -format
```

## 11. 10. Start the Hadoop cluster

Start HDFS first:

```bash
start-dfs.sh
```

Then start YARN:

```bash
start-yarn.sh
```

Verify the Java daemons:

```bash
jps
```

For the single-node configuration, the expected processes are:

- NameNode
- DataNode
- SecondaryNameNode
- ResourceManager
- NodeManager
- Jps

## 12. 11. Verify the Hadoop web interfaces

From Windows Edge or Chrome, use:

| Service | URL |
|---|---|
| HDFS NameNode UI | http://localhost:9870 |
| YARN ResourceManager UI | http://localhost:8088 |

## 13. 12. Add one-word Hadoop aliases to .bashrc

This is the main repeatability improvement in the source document: combine the HDFS and YARN startup commands into one shell alias, do the same for shutdown, and keep `jps` as the status check.

Open the profile:

```bash
nano ~/.bashrc
```

Append:

```bash
alias start-hadoop='start-dfs.sh && start-yarn.sh'
alias stop-hadoop='stop-yarn.sh && stop-dfs.sh'
alias hadoop-status='jps'
```

Reload:

```bash
source ~/.bashrc
```

Verify the aliases:

```bash
start-hadoop
hadoop-status
```

To stop the cluster cleanly:

```bash
stop-hadoop
hadoop-status
```

## 14. 13. Optional PowerShell shortcuts to WSL aliases

The source also shows how to invoke the Linux aliases directly from a Windows PowerShell session. These are Windows PowerShell functions, not replacements for the Linux aliases.

Open the PowerShell profile:

```powershell
notepad $PROFILE
```

Add:

```powershell
function Start-Hadoop { wsl bash -ic "start-hadoop" }
function Stop-Hadoop  { wsl bash -ic "stop-hadoop" }
function Hadoop-Jps   { wsl bash -ic "hadoop-status" }
```

Save the file and reload it:

```powershell
. $PROFILE
```

Then these commands can be used directly in PowerShell:

```powershell
Start-Hadoop
Hadoop-Jps
Stop-Hadoop
```

## 15. 14. Routine startup after reboot

Once the installation is complete, the recurring workflow is intentionally short:

```bash
# Inside WSL Ubuntu
source ~/.bashrc
start-hadoop
hadoop-status
```

Or from Windows PowerShell:

```powershell
Start-Hadoop
Hadoop-Jps
```

## 16. 15. Clean shutdown and stale PID troubleshooting

Always stop Hadoop before closing the environment:

```bash
stop-hadoop
```

If an unexpected reboot leaves stale Hadoop PID files and the next startup reports existing process locks, the source suggests removing orphaned PID files:

```bash
rm -f /tmp/hadoop-*.pid
```

Then retry:

```bash
start-hadoop
hadoop-status
```

## 17. 16. Final checklist

- [ ] WSL 2 and Ubuntu are installed.
- [ ] Java 17 is installed and `JAVA_HOME` points to the Java 17 directory.
- [ ] Hadoop 3.5.0 is installed at `/usr/local/hadoop`.
- [ ] Hadoop's `bin` and `sbin` directories are on `PATH`.
- [ ] Passwordless localhost SSH works.
- [ ] HDFS data directories exist.
- [ ] The four Hadoop XML files are configured for the single-node setup.
- [ ] `hdfs namenode -format` has been run once for the fresh cluster.
- [ ] `start-hadoop` starts HDFS and YARN.
- [ ] `hadoop-status` reports the expected daemons.
- [ ] `stop-hadoop` shuts the cluster down cleanly.
- [ ] The NameNode and ResourceManager UIs open at ports 9870 and 8088.

## Scope note

This document deliberately omits the native Windows `winutils.exe`/`hadoop.dll` installation route, native Windows environment-variable editing, and the Python/PySide6 desktop-controller/application-development material present later in the supplied PDF. The operational path here is WSL 2 Ubuntu plus Hadoop shell configuration and aliases.

---

Made with code and caffeine by [ALICE LLM](https://ollama.com/S00K/alice) — developed by [S00K](https://github.com/Stylish00killer)
