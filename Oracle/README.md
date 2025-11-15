# 🧩 Deploying OSB 12c in LXC + Oracle DB in Docker (Separate LXC) + Win11 JDeveloper

## 📘 Overview

This guide shows how to install:

-   **Oracle Service Bus 12c (OSB)** inside a **Proxmox LXC**
-   **Oracle Database 19c** running in **Docker inside a separate LXC**
-   **Windows 11 VM** connecting through **JDeveloper 12c**

A realistic, lightweight SOA/OSB development environment.

## 🧱 Architecture Diagram
![alt text](<diagrams/diagram - deployment.png>)

![alt text](<diagrams/diagram - deployment 001.png>)

## 📦 Requirements

### LXC OSB 12c

Oracle Linux (Oracle Linux 7) is technically free to download and use, but support and updates require a subscription, which may be a concern for some setups. For OSB 12c, you have a few alternative OS options that are certified or commonly used.

Debian LXC can work for a lab or dev environment, but it’s not supported for production. If your goal is learning, testing, or PoC, Debian is fine. For production, stick to Oracle Linux 7 or RHEL 7.

| Resource | Requirement                          |
| -------- | ------------------------------------ |
| CPU      | 4 vCPU                               |
| Memory   | **12 GB recommended**                |
| Disk     | **60 GB**                            |
| OS       | debian-12-standard                   |
| Notes    | Requires WebLogic + OSB installation |


### LXC Oracle DB (Docker)

| Resource | Requirement                    |
| -------- | ------------------------------ |
| CPU      | 2 vCPU                         |
| Memory   | **8 GB minimum**               |
| Disk     | **40 GB**                      |
| OS       | Ubuntu 22.04 or Oracle Linux 8 |
| Notes    | Runs Oracle DB 19c in Docker   |


### Windows 11 VM

| Resource | Requirement    |
| -------- | -------------- |
| CPU      | 2 vCPU         |
| Memory   | 8 GB           |
| Disk     | 60 GB          |
| Notes    | JDeveloper 12c |


## ⚙️ Creating LXCs in Proxmox

### Create OSB LXC:

``` bash
pct create 200 local:vztmpl/pct create 210 local:vztmpl/debian-12-standard_20231011_amd64.tar.xz \
  --hostname osb-debian --cores 4 --memory 12288 --swap 1024 --rootfs local-lvm:60 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.20/24,gw=192.168.0.1 \
  --features nesting=1
pct start 210

```

### Create DB LXC:

``` bash
pct create 210 local:vztmpl/debian-12-standard_20231011_amd64.tar.xz \
  --hostname orcl-debian --cores 4 --memory 16384 --swap 2048 --rootfs local-lvm:80 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.122.21/24,gw=192.168.122.1 \
  --features nesting=1
pct start 210

```

Start containers:

``` bash
pct start 200
pct start 201
```

## 🐋 Setup Oracle DB (Docker) in LXC #201

``` bash
apt update && apt install -y docker.io docker-compose
systemctl enable --now docker
```

### Create Volume

``` bash
mkdir /home/avictoria/oracle-data
docker volume create --name oracle-data \
    --opt type=none \
    --opt device=/home/avictoria/oracle-data \
    --opt o=bind       
```

### Run Oracle DB

docker image: https://hub.docker.com/r/gvenzl/oracle-xe
``` bash

docker pull gvenzl/oracle-xe
docker run -d --name oracle-db \
  -p 1521:1521 -p 5500:5500 \
  -e ORACLE_PASSWORD=Welcome123 \
  -e ORACLE_CHARACTERSET=AL32UTF8 \
  -e "ORACLE_ALLOW_REMOTE=true" \
  -v oracle-data:/opt/oracle/oradata \
  gvenzl/oracle-xe
```
check: https://github.com/lxc/incus/issues/2623
``` bash
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: open sysctl net.ipv4.ip_unprivileged_port_start file: reopen fd 8: permission denied
```

Default connection info for gvenzl/oracle-xe
| Parameter | Value                       |
| --------- | --------------------------- |
| Host      | LXC IP (e.g., 192.168.0.31) |
| Port      | 1521                        |
| Service   | `XE`                        |
| User      | `SYSTEM`                    |
| Password  | `Welcome123`                |


## 🧰 Prepare OSB LXC

Install Required Packages
``` bash
apt update
apt install -y libaio1 ksh unzip wget net-tools xvfb curl gnupg lsb-release
``` 

### Install Python 2 (Required by RCU)

Debian 12 removed Python 2, so we use the legacy Bullseye repo:
``` bash
echo "deb http://deb.debian.org/debian bullseye main" > /etc/apt/sources.list.d/oldbullseye.list
apt update
apt install -y python2
rm /etc/apt/sources.list.d/oldbullseye.list
apt update
```
### Install Java 8 (Required by OSB 12c)
Upload jdk-8u331-linux-x64.tar.gz to LXC /opt/java using pct push or SCP:

``` bash
mkdir -p /opt/java
# pct push 210 jdk-8u331-linux-x64.tar.gz /opt/java/


scp fmw_12.2.1.4.0_infrastructure.jar root@192.168.0.20:/opt/oracle/
scp fmw_12.2.1.4.0_soa.jar root@192.168.0.20:/opt/oracle/
scp fmw_12.2.1.4.0_b2b.jar root@192.168.0.20:/opt/oracle/
scp fmw_12.2.1.4.0_osb.jar root@192.168.0.20:/opt/oracle/


scp jdk-8u331-linux-x64.tar.gz root@192.168.0.20:/opt/java/
```
Extract and set environment:
``` bash
cd /opt/java
tar -xzf jdk-8u331-linux-x64.tar.gz
export JAVA_HOME=/opt/java/jdk1.8.0_331
export PATH=$JAVA_HOME/bin:$PATH
java -version
# Should show Java 1.8.x
```

## Use Silent Mode (fully automated install, no GUI)

Oracle installers support silent XML response files.

Infrastructure Response `silent-infra.rsp`
Silent Install Response File (OSB + WebLogic 12.2.1.4)
``` bash
[ENGINE]
Response File Version=1.0.0.0.0
[GENERIC]
ORACLE_HOME=/home/avictoria/oracle/middleware
INSTALL_TYPE=Fusion Middleware Infrastructure
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
```
SOA Suite Response `silent-soa.rsp`
``` bash
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
ORACLE_HOME=/home/avictoria/oracle/middleware
INSTALL_TYPE=SOA Suite
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
```
OSB Response `silent-osb.rsp` 
``` bash
[ENGINE]
Response File Version=1.0.0.0.0

[GENERIC]
ORACLE_HOME=/home/avictoria/oracle/middleware
INSTALL_TYPE=Service Bus
DECLINE_SECURITY_UPDATES=true
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
```
## Create Oracle Inventory Pointer
Here is the complete oraInst.loc file you need for a non-root / user-level install using your user.

This file tells the installer where to place the Oracle Inventory when you are not root.
``` bash
mkdir -p /home/avictoria/oraInventory
```
If oraInst.loc does not exist, create it:

``` bash
inventory_loc=/home/avictoria/oraInventory
inst_group=avictoria
```

🏃 How to Run (Silent Install Commands)

## 🧩 Install WebLogic Infrastructure

``` bash
java -jar fmw_12.2.1.4.0_infrastructure.jar \
  -silent \
  -responseFile /home/avictoria/fmw_12214/silent-infra.rsp \
  -invPtrLoc /home/avictoria/oraInventory/oraInst.loc
```

## 🧩 Install SOA

``` bash
java -jar fmw_12.2.1.4.0_soa.jar \
  -silent \
  -responseFile /home/avictoria/fmw_12214/silent-soa.rsp \
  -invPtrLoc /home/avictoria/oraInventory/oraInst.loc
```
## 🧩 Install OSB

``` bash
java -jar fmw_12.2.1.4.0_osb.jar \
  -silent \
  -responseFile /home/avictoria/fmw_12214/silent-osb.rsp \
  -invPtrLoc /home/avictoria/oraInventory/oraInst.loc
```

## ⚙️ RCU — Schema Creation for SOA + OSB

Access the Oracle XE container
``` bash
docker exec -it oracle-xe bash
```
Connect to the pluggable database (XEPDB1) as SYSDBA
``` bash
sqlplus sys/Welcome123@XEPDB1 as sysdba
```
Create a single tablespace SOAOSB for all schemas
``` bash
CREATE TABLESPACE SOAOSB 
DATAFILE '/opt/oracle/oradata/XE/soaosb.dbf' 
SIZE 500M 
AUTOEXTEND ON;

SELECT tablespace_name, status, contents FROM dba_tablespaces;

``` 
From the OSB LXC
``` bash
cd /home/avictoria/oracle/middleware/oracle_common/bin

bash ./rcu -silent -createRepository \
  -databaseType ORACLE \
  -connectString 192.168.0.31:1521/XEPDB1 \
  -dbUser sys -dbRole SYSDBA \
  -schemaPrefix SOAOSB \
  -useSamePasswordForAllSchemaUsers true \
  -component WLS \
  -component MDS \
  -component SOAINFRA \
  -component ESS \
  -component OPSS \
  -component IAU \
  -component IAU_APPEND \
  -component IAU_VIEWER \
  -component STB \
  -component UCSUMS \
  -tablespace SOAOSB

# dropRepository
bash ./rcu -silent -dropRepository \
  -databaseType ORACLE \
  -connectString 192.168.0.31:1521/XEPDB1 \
  -dbUser sys -dbRole SYSDBA \
  -schemaPrefix SOAOSB \
  -component WLS \
  -component MDS \
  -component SOAINFRA \
  -component ESS \
  -component OPSS \
  -component IAU \
  -component IAU_APPEND \
  -component IAU_VIEWER \
  -component STB \
  -component UCSUMS \
  -f
```

### Database connection:

    Host: 192.168.0.31
    Port: 1521
    Service: XEPDB1
    User: SOAOSB_SOAINFRA
    Password: Welcome123
    Role: Normal

## ⚙️ Create OSB Domain

Create Domain (SOA + OSB Combined Domain)

WLS Configuration Wizard silent config:

`/home/avictoria/oracle/middleware/oracle_common/common/bin/` file: `create-domain.py`
``` python
print('=== Loading base WebLogic template ===')

readTemplate('/home/avictoria/oracle/middleware/wlserver/common/templates/wls/wls.jar')

print('=== Adding JRF template ===')
addTemplate('/home/avictoria/oracle/middleware/oracle_common/common/templates/wls/oracle.jrf_template.jar')

print('=== Adding SOA template ===')
addTemplate('/home/avictoria/oracle/middleware/soa/common/templates/wls/oracle.soa_template.jar')

print('=== Adding OSB template ===')
addTemplate('/home/avictoria/oracle/middleware/osb/common/templates/wls/oracle.osb_template.jar')

# --- Domain options ---
print('=== Domain Options ===')
setOption('DomainName', 'soa_domain')
setOption('JavaHome', '/opt/java/jdk1.8.0_331')
setOption('OverwriteDomain', 'true')

# --- Admin user ---
print('=== Configuring User ===')
cd('/Security/base_domain/User/weblogic')
cmo.setPassword('Welcome123')

print('=== Configuring RCU datasources ===')

def configureDS(dsName, schema):
    try:
        cd('/JDBCSystemResource/' + dsName + '/JdbcResource/' + dsName + '/JDBCDriverParams/NO_NAME_0')
        cmo.setUrl('jdbc:oracle:thin:@//192.168.0.31:1521/XEPDB1')
        cmo.setPasswordEncrypted('Welcome123')
        cd('Properties/NO_NAME_0/Property/user')
        cmo.setValue(schema)
        print("Configured DS: " + dsName)
    except:
        print("WARNING: datasource not found: " + dsName)

configureDS('LocalSvcTblDataSource', 'SOAOSB_STB')
configureDS('opss-data-source', 'SOAOSB_OPSS')
configureDS('opss-audit-DBDS', 'SOAOSB_IAU')
configureDS('SOADataSource', 'SOAOSB_SOAINFRA')
configureDS('EDNDataSource', 'SOAOSB_SOAINFRA')
configureDS('OraSDPMDataSource', 'SOAOSB_UCSUMS')
configureDS('mds-owsm', 'SOAOSB_MDS')
configureDS('mds-soa', 'SOAOSB_MDS')

print('=== Setting AdminServer ports ===')

cd('/Servers/AdminServer')
set('ListenAddress', '')
set('ListenPort', 7001)

print('=== Writing domain ===')

writeDomain('/home/avictoria/oracle/middleware/user_projects/domains/soa_domain')
closeTemplate()

exit()
```

``` bash
/home/avictoria/oracle/middleware/oracle_common/common/bin/wlst.sh create_domain.py
```

## Verify the domain structure

`ls -l /home/avictoria/oracle/middleware/user_projects/domains/`
You should see your domain folder (e.g., soa_domain).


## 🚀 Start WebLogic

``` bash
cd /home/avictoria/oracle/middleware/user_projects/domains/soa_domain/bin
./startWebLogic.sh
```
If you run it in LXC without GUI, use nohup or screen to keep it running:
``` bash
nohup ./startWebLogic.sh &> ~/AdminServer.out &
tail -f ~/AdminServer.out
```

tail -f /home/avictoria/oracle/middleware/user_projects/domains/soa_domain/servers/AdminServer/logs/AdminServer.log

tail -f /home/avictoria/oracle/middleware/user_projects/domains/soa_domain/servers/AdminServer/logs/AdminServer.log

## 🧠 Connect from Windows 11 JDeveloper

    Host: 192.168.0.20
    Port: 7001
    User: weblogic

