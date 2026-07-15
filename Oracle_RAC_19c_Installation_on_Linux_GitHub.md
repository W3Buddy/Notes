# Oracle RAC 19c Installation on Linux – Complete Step-by-Step Guide
> **Production-ready SOP** converted to GitHub Markdown.
Oracle RAC 19c Installation on Linux – Complete Step-by-Step Guide
A complete production-ready SOP for installing Oracle Real Application Clusters (RAC) 19c on Linux from scratch. Covers cluster planning, shared storage setup, Grid Infrastructure installation, ASM diskgroup creation, RAC database creation, services configuration, and full post-installation validation — with real commands, expected outputs, and consultant-level notes for both standard OFA and enterprise custom path conventions.
1. Document Info
2. RAC Architecture — Understand Before You Start
> **Note:** What is Oracle RAC? Oracle RAC (Real Application Clusters) allows multiple servers (nodes) to run the same Oracle database simultaneously. All nodes share the same database files stored on shared storage. If one node fails, the other nodes continue serving the database — providing High Availability. RAC also scales performance by distributing the workload across nodes.
> **Note:** Why is RAC complex? Unlike a standalone database where everything is on one server, RAC requires precise coordination between servers — shared storage, private network for inter-node communication, cluster software, shared voting disks, and synchronized configuration across all nodes. Every component must be correctly configured or the cluster will not form.
Key RAC Components You Must Know
RAC Network Requirements Summary
> **Warning:** IMPORTANT: The private interconnect network must be a dedicated, low-latency network. Never share the interconnect with public traffic. In real environments this is either a dedicated NIC pair, a dedicated VLAN, or InfiniBand. A slow or shared interconnect is the number one cause of RAC performance problems.
3. Path Conventions
> **Note:** Confirm which convention your environment uses before starting. All examples use Convention A. Substitute Convention B paths where applicable.
> **Warning:** IMPORTANT: Grid Infrastructure has its OWN separate ORACLE_HOME (Grid Home). It must NEVER be the same directory as the database ORACLE_HOME. Installing GI into the DB home or vice versa will corrupt both installations.
4. Pre-Installation Planning
> **Note:** Why plan before touching anything? RAC installation involves DNS entries, IP addresses, hostnames, and network configuration that must all be consistent and correct BEFORE the first installer is run. Changing these after GI is installed is painful and sometimes requires a full reinstall.
## 4.1 — Plan and Document Your IP Address Layout
> **Note:** Why? Every IP in RAC has a specific role. You must plan all IPs before touching any server. Share this with your network team to get the IPs allocated and configured.
Example 2-node RAC IP layout — fill this in for your environment:
> **Warning:** IMPORTANT: SCAN IPs must be registered in DNS — NOT in /etc/hosts. Oracle Grid Infrastructure specifically checks that SCAN resolves via DNS and not via hosts file. If SCAN is in /etc/hosts only, GI installation will fail prerequisite checks.
> **Note:** Why 3 SCAN IPs? Oracle recommends 3 SCAN IPs for load balancing and availability. With 3 SCAN IPs, client connections are round-robin distributed. You can use 1 SCAN IP in lab/test environments but never in production.
## 4.2 — Plan Shared Storage Layout
> **Note:** Why? All nodes must see the exact same shared disks with the exact same device paths. Plan the diskgroup layout before storage is presented to the servers.
Recommended diskgroup layout:
> **Note:** Why separate +OCR diskgroup? OCR and Voting disks are critical cluster files. Keeping them in a separate diskgroup with NORMAL or HIGH redundancy protects the cluster even if the DATA diskgroup has issues. Never put OCR/Voting disks in the same diskgroup as database files in production.
5. Pre-Installation Checks — Both Nodes
> **Warning:** IMPORTANT: Every check in this section must be performed on ALL nodes unless stated otherwise. A configuration difference between nodes is the most common cause of RAC installation failures. Always run checks on node1 AND node2 and compare results.
## 5.1 — Check OS Version and Architecture on All Nodes
> **Note:** Why? Both nodes must run identical OS versions. A mismatch in kernel version or OS release between nodes causes clusterware instability.
```bash
# Run on BOTH nodes and compare output — must be identical
cat /etc/os-release
uname -r
uname -m
What to look for: Exact same output on both nodes. Any difference must be resolved before proceeding.
```
## 5.2 — Check and Configure /etc/hosts on All Nodes
> **Note:** Why? /etc/hosts must have entries for all public, private, and VIP hostnames on all nodes. Private network hostnames go here (not DNS). Public and VIP hostnames can be in DNS or hosts file — but must be resolvable.
```bash
# Check current /etc/hosts on both nodes
cat /etc/hosts
Add the following entries to /etc/hosts on ALL nodes (as root):
# -------------------------------------------------------
# Oracle RAC /etc/hosts entries
# -------------------------------------------------------
```
# Public IPs
###### 192.168.1.11    racnode1.company.com      racnode1
###### 192.168.1.12    racnode2.company.com      racnode2
# Virtual IPs (VIP) — managed by Clusterware
###### 192.168.1.13    racnode1-vip.company.com  racnode1-vip
###### 192.168.1.14    racnode2-vip.company.com  racnode2-vip
# Private Interconnect IPs — used for Cache Fusion only
###### 192.168.10.11   racnode1-priv.company.com racnode1-priv
###### 192.168.10.12   racnode2-priv.company.com racnode2-priv
# NOTE: DO NOT add SCAN entries here — SCAN must resolve via DNS only
```bash
# Verify hostname resolution works from both nodes
# Run on node1 — ping all hostnames
ping -c 2 racnode1
ping -c 2 racnode2
ping -c 2 racnode1-vip
ping -c 2 racnode2-vip
ping -c 2 racnode1-priv
ping -c 2 racnode2-priv
```
# Verify SCAN resolves via DNS (not hosts file)
nslookup rac-scan
# Must return 1-3 IPs and must NOT come from /etc/hosts
## 5.3 — Check Network Interfaces on All Nodes
> **Note:** Why? Grid Infrastructure needs to know which interface is public and which is private. It detects this during installation based on subnet. Both nodes must have identical interface names and subnet assignments.
```bash
# Check all network interfaces and their IP addresses
ip addr show
```
# Check interface names (must be same on both nodes)
ip link show
# Verify public interface is up
ping -c 3 192.168.1.11    # node1 public from node2
ping -c 3 192.168.1.12    # node2 public from node1
# Verify private interconnect is up and working
ping -c 3 192.168.10.11   # node1 private from node2
ping -c 3 192.168.10.12   # node2 private from node1
# Check interconnect latency — should be sub-millisecond
ping -c 10 192.168.10.12 | tail -2
> **Warning:** IMPORTANT: If private interconnect ping shows latency above 1ms consistently, investigate the network. Poor interconnect performance will cause Cache Fusion bottlenecks and cluster instability in production.
## 5.4 — Check Disk Space on All Nodes
```bash
# Check all filesystems — run on both nodes
df -hP
```
# Minimum requirements:
# /u01 (or /oracle) — 15GB+ for GI Home + DB Home + diag files
# /tmp — 1GB+
# /home/oracle — 1GB+ (for oracle user home)
df -hP /tmp
free -m
## 5.5 — Check Required OS Packages on All Nodes
> **Note:** Why? RAC requires additional packages beyond standalone installation — specifically for cluster communication and shared storage.
```bash
# Run on BOTH nodes
rpm -q binutils \
        compat-libcap1 \
        compat-libstdc++-33 \
        gcc \
        gcc-c++ \
        glibc \
        glibc-devel \
        ksh \
        libaio \
        libaio-devel \
        libgcc \
        libstdc++ \
        libstdc++-devel \
        libXi \
        libXtst \
        make \
        sysstat \
        elfutils-libelf-devel \
        nfs-utils \
        smartmontools \
        net-tools \
        bind-utils \
        unzip 2>&1 | grep "not installed"
```
```bash
# Install missing packages on BOTH nodes (as root)
yum install -y binutils compat-libcap1 compat-libstdc++-33 gcc gcc-c++ \
glibc glibc-devel ksh libaio libaio-devel libgcc libstdc++ libstdc++-devel \
libXi libXtst make sysstat elfutils-libelf-devel nfs-utils \
smartmontools net-tools bind-utils unzip
```
## 5.6 — Check and Set Kernel Parameters on All Nodes
> **Note:** Why? Same as standalone but with additional parameters needed for cluster communication and shared memory across nodes.
```bash
# Check current values — run on BOTH nodes
/sbin/sysctl -a | grep -E \
"kernel.shmall|kernel.shmmax|kernel.shmmni|\
kernel.sem|fs.file-max|fs.aio-max-nr|\
net.ipv4.ip_local_port_range|\
net.core.rmem_default|net.core.rmem_max|\
net.core.wmem_default|net.core.wmem_max|\
net.ipv4.conf.all.rp_filter|\
net.ipv4.conf.default.rp_filter"
Add to /etc/sysctl.conf on ALL nodes (as root):
# -------------------------------------------------------
# Oracle 19c RAC Required Kernel Parameters
# Added by DBA on <date>
# -------------------------------------------------------
```
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
fs.file-max = 6815744
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
# RAC specific — disable reverse path filtering on interconnect
# Without this, interconnect packets may be dropped by the kernel
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
```bash
# Apply on BOTH nodes
/sbin/sysctl -p
```
# Verify on BOTH nodes
/sbin/sysctl -a | grep -E "kernel.shmall|net.ipv4.conf.all.rp_filter"
## 5.7 — Check and Set OS User Limits on All Nodes
> **Note:** Why? Same limits as standalone. Must be configured identically on all nodes.
Add to /etc/security/limits.conf on ALL nodes (as root):
# -------------------------------------------------------
# Oracle 19c RAC User Resource Limits
# -------------------------------------------------------
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc     2047
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   soft   memlock   3145728
oracle   hard   memlock   3145728
grid     soft   nofile    1024
grid     hard   nofile    65536
grid     soft   nproc     2047
grid     hard   nproc     16384
grid     soft   stack     10240
grid     hard   stack     32768
grid     soft   memlock   3145728
grid     hard   memlock   3145728
> **Note:** Note: In RAC, we also set limits for the grid OS user (who owns the Grid Infrastructure installation) in addition to the oracle user.
## 5.8 — Check Shared Storage Visibility from All Nodes
> **Note:** Why? All shared disks must be visible from ALL nodes with the same device paths. If node1 sees a disk as /dev/sdb but node2 sees the same disk as /dev/sdd, ASM will fail or worse — both nodes will try to write to different disks thinking they are the same.
```bash
# Check block devices on BOTH nodes
lsblk
```
# List all disk devices
ls -l /dev/sd*
# Confirm same disks visible — sizes must match
fdisk -l 2>/dev/null | grep "^Disk /dev/sd"
# Verify device paths are identical on both nodes
# Run this on node1 then node2 and compare
ls -l /dev/sd* | awk '{print $5, $6, $10}'
> **Warning:** IMPORTANT: If device paths differ between nodes, you must work with your storage team to present the LUNs with consistent device names. Use udev rules or ASMLib/ASMFD to assign consistent names to shared disks across all nodes.
## 5.9 — Configure SSH User Equivalence Between Nodes
> **Note:** Why? Oracle Grid Infrastructure installer runs scripts on all nodes simultaneously. It uses SSH to execute commands on remote nodes without a password prompt. Without passwordless SSH between nodes (for both oracle and grid users), the installer will hang waiting for a password.
```bash
# Run on NODE 1 as oracle user
su - oracle
```
# Step 1 — Generate SSH key pair on node1
ssh-keygen -t rsa -b 2048
# Press Enter for all prompts (no passphrase)
# Step 2 — Copy public key to node2
ssh-copy-id oracle@racnode2
# Step 3 — Test passwordless SSH from node1 to node2
ssh racnode2 hostname
ssh racnode2 date
# Step 4 — Test passwordless SSH from node1 to itself (required by GI)
ssh racnode1 hostname
```bash
# Repeat on NODE 2 as oracle user
su - oracle
ssh-keygen -t rsa -b 2048
ssh-copy-id oracle@racnode1
ssh racnode1 hostname
ssh racnode2 hostname
```
```bash
# Repeat the entire process above for the grid user on BOTH nodes
# Grid user also needs passwordless SSH for GI installer
su - grid
ssh-keygen -t rsa -b 2048
ssh-copy-id grid@racnode1
ssh-copy-id grid@racnode2
ssh racnode1 hostname
ssh racnode2 hostname
```
> **Warning:** IMPORTANT: Test SSH in BOTH directions from BOTH nodes. The installer tests all combinations. If any single combination requires a password, the installer will fail.
## 5.10 — Check Time Synchronization on All Nodes
> **Note:** Why? Oracle Clusterware requires all nodes to have synchronized clocks. A time difference of more than a few seconds between nodes causes Clusterware to evict nodes from the cluster (a deliberate protection mechanism against split-brain). NTP or Chrony must be configured and working on all nodes.
```bash
# Check time synchronization service status (run on ALL nodes)
systemctl status chronyd
# OR
systemctl status ntpd
```
# Check current time on all nodes — must be within seconds of each other
date
# Check NTP/Chrony sync status
chronyc tracking
# OR
ntpq -p
# Check time difference between nodes
# From node1 — compare date with node2
ssh racnode2 date
> **Warning:** IMPORTANT: If time is not synchronized, configure and start chrony before proceeding. GI will explicitly check time synchronization during installation.
```bash
# Configure chrony if not running (as root on ALL nodes)
systemctl enable chronyd
systemctl start chronyd
chronyc makestep    # Force immediate time sync
chronyc tracking    # Verify synchronized
```
## 5.11 — Verify SCAN DNS Resolution
> **Note:** Why? GI installer explicitly checks that SCAN resolves via DNS. This is a hard requirement. If it fails, the installer stops.
```bash
# Verify SCAN resolves to 1-3 IPs via DNS (run on ALL nodes)
nslookup rac-scan
# OR
host rac-scan
# OR
dig rac-scan
```
# Must return IP addresses and must NOT use /etc/hosts
# Also verify forward and reverse resolution
nslookup 192.168.1.15
nslookup 192.168.1.16
nslookup 192.168.1.17
What correct nslookup rac-scan output looks like:
Server:   192.168.1.1
Address:  192.168.1.1#53
Name:     rac-scan.company.com
Address:  192.168.1.15
Name:     rac-scan.company.com
Address:  192.168.1.16
Name:     rac-scan.company.com
Address:  192.168.1.17
## 5.12 — Run Oracle Cluster Verification Utility (cluvfy)
> **Note:** Why? cluvfy is Oracle's own pre-installation checker for RAC. It checks everything in one command — OS packages, kernel parameters, SSH equivalence, network configuration, disk visibility, and more. It is the most comprehensive pre-check available. Always run it before starting GI installation.
```bash
# cluvfy is inside the Grid Infrastructure software
# First unzip GI software (as grid user) to Grid Home
su - grid
cd /u01/app/19.3.0/grid
unzip -q /stage/19c/LINUX.X64_193000_grid_home.zip
```
# Run pre-installation checks for all nodes
$ORACLE_HOME/runcluvfy.sh stage -pre crsinst \
    -n racnode1,racnode2 \
    -fixup \
    -verbose
> **Note:** What -fixup does: Generates a fixup script that automatically corrects many of the issues it finds. Review and run the fixup script as root on all nodes.
```bash
# After cluvfy completes — review the fixup script if generated
# Run as root on ALL nodes
/tmp/CVU_<date>/runfixup.sh
```
> **Warning:** IMPORTANT: Do NOT proceed with GI installation until cluvfy shows no FAILED checks. WARNING items can sometimes be ignored (cluvfy output explains each one) but FAILED items must be fixed.
6. Pre-Installation Configuration — Both Nodes
> **Warning:** IMPORTANT: Every configuration step must be performed on ALL nodes unless explicitly stated. Always confirm by checking both nodes after each step.
## 6.1 — Create OS Groups on All Nodes
> **Note:** Why? Same groups as standalone plus additional groups for Grid Infrastructure. GIDs must be identical across all nodes.
```bash
# Run as root on ALL nodes
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
groupadd -g 54324 backupdba
groupadd -g 54325 dgdba
groupadd -g 54326 kmdba
groupadd -g 54327 racdba
groupadd -g 54328 asmadmin    # ASM administrator group
groupadd -g 54329 asmdba      # ASM SYSDBA group
groupadd -g 54330 asmoper     # ASM SYSOPER group
```
# Verify all groups on BOTH nodes — output must be identical
grep -E "oinstall|dba|oper|backupdba|dgdba|kmdba|racdba|asmadmin|asmdba|asmoper" /etc/group
## 6.2 — Create oracle and grid OS Users on All Nodes
> **Note:** Why two users? In RAC, Oracle best practice separates the Grid Infrastructure owner (grid) from the database software owner (oracle). This provides privilege separation — the grid user owns Clusterware and ASM, the oracle user owns the database software. UIDs must be identical across all nodes.
```bash
# Run as root on ALL nodes
```
# Create grid user — owns Grid Infrastructure and ASM
useradd -u 54320 \
        -g oinstall \
        -G dba,asmadmin,asmdba,asmoper \
        -d /home/grid \
        -s /bin/bash \
        grid
passwd grid
# Create oracle user — owns database software
useradd -u 54321 \
        -g oinstall \
        -G dba,oper,backupdba,dgdba,kmdba,racdba,asmadmin,asmdba \
        -d /home/oracle \
        -s /bin/bash \
        oracle
passwd oracle
# Verify both users on BOTH nodes
id grid
id oracle
## 6.3 — Create Directory Structure on All Nodes
> **Note:** Why? Grid Home and DB Home directories must exist on all nodes with correct ownership. These are local directories — not shared. Each node has its own copy of the software.
```bash
# Run as root on ALL nodes
```
# Convention A
mkdir -p /u01/app/19.3.0/grid                        # Grid Infrastructure Home
mkdir -p /u01/app/oracle/product/19.3.0/dbhome_1     # Database Home
mkdir -p /u01/app/oraInventory                         # Oracle Inventory
mkdir -p /u01/app/grid                                 # Grid Base
# Set ownership
chown -R grid:oinstall    /u01/app/19.3.0
chown -R grid:oinstall    /u01/app/grid
chown -R grid:oinstall    /u01/app/oraInventory
chown -R oracle:oinstall  /u01/app/oracle
# Set permissions
chmod -R 775 /u01/app
```bash
# Convention B
mkdir -p /oracle/GRID/19.31
mkdir -p /oracle/RDBMS/19.31
mkdir -p /oracle/oraInventory
mkdir -p /oracle/grid
```
chown -R grid:oinstall    /oracle/GRID
chown -R grid:oinstall    /oracle/grid
chown -R grid:oinstall    /oracle/oraInventory
chown -R oracle:oinstall  /oracle/RDBMS
```bash
# Verify on ALL nodes
ls -ld /u01/app/19.3.0/grid
ls -ld /u01/app/oracle/product/19.3.0/dbhome_1
ls -ld /u01/app/oraInventory
```
## 6.4 — Configure Environment Profiles on All Nodes
For grid user — add to ~/.bash_profile on ALL nodes:
```bash
su - grid
vi ~/.bash_profile
```
```bash
# -------------------------------------------------------
# Grid Infrastructure Environment — grid user
# -------------------------------------------------------
export ORACLE_BASE=/u01/app/grid               # Grid Base (separate from oracle base)
export ORACLE_HOME=/u01/app/19.3.0/grid        # Grid Infrastructure Home
export ORACLE_SID=+ASM                          # ASM instance SID
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
export PATH=$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```
alias asmcli='asmcmd'
alias alert='tail -500f $ORACLE_BASE/diag/asm/+asm/+ASM/trace/alert_+ASM.log'
For oracle user — add to ~/.bash_profile on ALL nodes:
```bash
su - oracle
vi ~/.bash_profile
```
```bash
# -------------------------------------------------------
# Oracle Database Environment — oracle user
# -------------------------------------------------------
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=RACDB1                        # Node1: RACDB1, Node2: RACDB2
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
export PATH=$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```
alias sqlp='sqlplus / as sysdba'
alias alert='tail -500f $ORACLE_BASE/diag/rdbms/*/*/trace/alert_*.log'
> **Note:** Note: ORACLE_SID is node-specific in RAC. Node1 uses RACDB1, node2 uses RACDB2. The DB_NAME is RACDB (shared). Each instance name = DB_NAME + node_number.
## 6.5 — Configure Shared Disk Permissions and udev Rules
> **Note:** Why? ASM needs to read and write directly to the raw shared disks. By default Linux sets disk permissions to root only. We must change permissions so the grid and oracle users can access the disks. udev rules make these permissions persistent across reboots.
```bash
# Step 1 — Identify the shared disks (run as root)
# These are the disks your storage team presented for ASM
lsblk | grep -v sda    # sda is usually the OS disk — exclude it
```
# Step 2 — Check WWIDs for persistent naming (important for SAN disks)
/usr/lib/udev/scsi_id --whitelisted --replace-whitespace --device=/dev/sdb
/usr/lib/udev/scsi_id --whitelisted --replace-whitespace --device=/dev/sdc
# Note the WWID output for each disk
# Step 3 — Create udev rules file for persistent disk naming and permissions
vi /etc/udev/rules.d/99-oracle-asmdevices.rules
Add one line per shared disk (use your actual WWIDs):
# Oracle ASM Disk udev Rules
# Format: KERNEL==device, SUBSYSTEM==block, ENV{ID_SERIAL}==WWID, SYMLINK+=name, OWNER=grid, GROUP=asmadmin, MODE=0660
# OCR/Vote diskgroup disks
KERNEL=="sdb", SUBSYSTEM=="block", ENV{ID_SERIAL}=="<WWID_DISK1>", SYMLINK+="oracleasm/ocr1", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sdc", SUBSYSTEM=="block", ENV{ID_SERIAL}=="<WWID_DISK2>", SYMLINK+="oracleasm/ocr2", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sdd", SUBSYSTEM=="block", ENV{ID_SERIAL}=="<WWID_DISK3>", SYMLINK+="oracleasm/ocr3", OWNER="grid", GROUP="asmadmin", MODE="0660"
# DATA diskgroup disks
KERNEL=="sde", SUBSYSTEM=="block", ENV{ID_SERIAL}=="<WWID_DISK4>", SYMLINK+="oracleasm/data1", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sdf", SUBSYSTEM=="block", ENV{ID_SERIAL}=="<WWID_DISK5>", SYMLINK+="oracleasm/data2", OWNER="grid", GROUP="asmadmin", MODE="0660"
# FRA diskgroup disks
KERNEL=="sdg", SUBSYSTEM=="block", ENV{ID_SERIAL}=="<WWID_DISK6>", SYMLINK+="oracleasm/fra1", OWNER="grid", GROUP="asmadmin", MODE="0660"
```bash
# Apply the udev rules immediately (as root on ALL nodes)
udevadm control --reload-rules
udevadm trigger
```
# Verify the symlinks and permissions were created
ls -l /dev/oracleasm/
# Expected output: all disks listed with owner=grid, group=asmadmin, mode=660
> **Note:** Alternative — Use ASM Filter Driver (ASMFD): ASMFD is Oracle's recommended approach in 19c instead of raw udev rules. It provides additional protection by filtering out non-ASM I/O to ASM disks. If your environment uses ASMFD, configure it during GI installation using the asmcmd afd_label command. ASMFD setup is covered in the ASMFD section of MOS Doc ID 2292840.1.
7. Grid Infrastructure Installation
> **Warning:** IMPORTANT: Grid Infrastructure is installed ONLY FROM NODE 1. The installer automatically connects to other nodes via SSH and installs/configures on all nodes simultaneously. Never run the GI installer simultaneously on multiple nodes.
## 7.1 — Unzip Grid Infrastructure Software on Node 1
```bash
# As grid user on NODE 1 ONLY
su - grid
```
# GI software must be unzipped INTO Grid Home (same 19c rule as DB software)
cd $ORACLE_HOME     # /u01/app/19.3.0/grid
# Unzip GI software directly into Grid Home
unzip -q /stage/19c/LINUX.X64_193000_grid_home.zip
# Verify key files are present
ls $ORACLE_HOME/gridSetup.sh      # GI installer (different from runInstaller)
ls $ORACLE_HOME/install/response/
> **Warning:** IMPORTANT: Grid Infrastructure uses gridSetup.sh as the installer, not runInstaller. This is different from the database installer.
## 7.2 — Run Grid Infrastructure Installation (Silent)
> **Note:** Why silent? Same reasons as standalone — no GUI dependency, repeatable, documentable. In production RAC environments GUI is almost never available.
```bash
# As grid user on NODE 1 ONLY
su - grid
cd $ORACLE_HOME
```
# Silent GI installation for a 2-node cluster
./gridSetup.sh -silent \
  -responseFile $ORACLE_HOME/install/response/gridsetup.rsp \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  oracle.install.option=CRS_CONFIG \
  ORACLE_BASE=/u01/app/grid \
  oracle.install.asm.OSDBA=asmdba \
  oracle.install.asm.OSOPER=asmoper \
  oracle.install.asm.OSASM=asmadmin \
  oracle.install.crs.config.scanName=rac-scan \
  oracle.install.crs.config.scanPort=1521 \
  oracle.install.crs.config.ClusterName=rac-cluster \
  oracle.install.crs.config.clusterNodes=racnode1:racnode1-vip,racnode2:racnode2-vip \
  oracle.install.crs.config.networkInterfaceList=eth0:192.168.1.0:1,eth1:192.168.10.0:5 \
  oracle.install.crs.config.storageOption=FLEX_ASM_STORAGE \
  oracle.install.crs.config.sharedFileSystemStorage.diskDriveList=/dev/oracleasm/ocr1,/dev/oracleasm/ocr2,/dev/oracleasm/ocr3 \
  oracle.install.crs.config.useIPMI=false \
  oracle.install.asm.diskGroup.name=OCR \
  oracle.install.asm.diskGroup.redundancy=NORMAL \
  oracle.install.asm.diskGroup.AUSize=4 \
  oracle.install.asm.diskGroup.disks=/dev/oracleasm/ocr1,/dev/oracleasm/ocr2,/dev/oracleasm/ocr3 \
  oracle.install.asm.diskGroup.diskDiscoveryString=/dev/oracleasm/* \
  oracle.install.asm.monitorPassword=Oracle_123 \
  oracle.install.crs.rootconfig.executeRootScript=false \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
  DECLINE_SECURITY_UPDATES=true
> **Note:** Key parameters explained:
oracle.install.crs.config.networkInterfaceList → format is interface:subnet:role where role 1=public, 5=private/ASM, 3=do not use
oracle.install.crs.config.clusterNodes → format is hostname:vip-hostname for each node
oracle.install.crs.config.scanName → must match exactly what DNS resolves
> **Note:** Monitor GI installation log in a separate terminal:
```bash
tail -100f /tmp/GridSetupActions*/gridSetupActions*.log
```
## 7.3 — Run Root Scripts on All Nodes (CRITICAL)
> **Warning:** IMPORTANT: When the installer prompts, you must run root scripts on ALL nodes. The order is critical — run on node1 FIRST, wait for it to complete, THEN run on node2. Running simultaneously or in wrong order will corrupt the cluster configuration.
```bash
# TERMINAL 1 — NODE 1 — as root
# Step 1 — Run orainstRoot.sh on node1 first
/u01/app/oraInventory/orainstRoot.sh
```
# Step 2 — Run root.sh on node1
# This starts Clusterware on node1 — takes 5-15 minutes
/u01/app/19.3.0/grid/root.sh
> **Note:** Wait for node1 root.sh to complete fully before touching node2.
```bash
# TERMINAL 2 — NODE 2 — as root (only after node1 root.sh completes)
# Step 3 — Run orainstRoot.sh on node2
/u01/app/oraInventory/orainstRoot.sh
```
# Step 4 — Run root.sh on node2
# This joins node2 to the cluster formed by node1
/u01/app/19.3.0/grid/root.sh
Expected output from root.sh on each node:
Performing root user operation.
The following environment variables are set as:
    ORACLE_OWNER= grid
    ORACLE_HOME=  /u01/app/19.3.0/grid
Copying dbhome to /usr/local/bin ...
Creating /etc/oratab file...
Entries will be added to the /etc/oratab file as needed...
CRS-4402: The CSS daemon was successfully initialized...
CRS-4000: Command Start completed successfully...
> **Note:** After both root.sh scripts complete, go back to the installer terminal and click OK (GUI) or let silent install continue.
## 7.4 — Verify Cluster Formation After GI Installation
```bash
# As grid user on any node
su - grid
```
# Check cluster status — all nodes must show online
$ORACLE_HOME/bin/crsctl check cluster -all
# Check all CRS resources
$ORACLE_HOME/bin/crsctl stat res -t
# Check cluster nodes
$ORACLE_HOME/bin/olsnodes -n -i -s
# Check SCAN and VIP resources
$ORACLE_HOME/bin/srvctl status scan
$ORACLE_HOME/bin/srvctl status vip -n racnode1
$ORACLE_HOME/bin/srvctl status vip -n racnode2
# Check ASM is running on all nodes
$ORACLE_HOME/bin/srvctl status asm
Expected output from crsctl check cluster -all:
**************************************************************
racnode1:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
racnode2:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
## 7.5 — Create Additional ASM Diskgroups (DATA and FRA)
> **Note:** Why? The GI installer only creates the OCR diskgroup. We must separately create the DATA and FRA diskgroups that the database will use.
```bash
# As grid user — connect to ASM instance
su - grid
sqlplus / as sysasm
```
```sql
-- Create +DATA diskgroup for database files
CREATE DISKGROUP DATA
    EXTERNAL REDUNDANCY
    DISK '/dev/oracleasm/data1',
         '/dev/oracleasm/data2'
    ATTRIBUTE 'AU_SIZE'          = '4M',
              'COMPATIBLE.ASM'   = '19.0',
              'COMPATIBLE.RDBMS' = '19.0';
```
-- Create +FRA diskgroup for recovery files
CREATE DISKGROUP FRA
    EXTERNAL REDUNDANCY
    DISK '/dev/oracleasm/fra1'
    ATTRIBUTE 'AU_SIZE'          = '4M',
              'COMPATIBLE.ASM'   = '19.0',
              'COMPATIBLE.RDBMS' = '19.0';
-- Verify all diskgroups are mounted
set linesize 200
set pagesize 50
col name     for a12
col state    for a12
col type     for a10
col total_gb for 99999.99
col free_gb  for 99999.99
SELECT name, state, type,
       total_mb/1024 total_gb,
       free_mb/1024  free_gb
FROM   v$asm_diskgroup
ORDER BY name;
```bash
# Register new diskgroups with CRS so they auto-mount on startup
su - grid
$ORACLE_HOME/bin/srvctl add diskgroup -diskgroup DATA -oraclehome $ORACLE_HOME
$ORACLE_HOME/bin/srvctl add diskgroup -diskgroup FRA  -oraclehome $ORACLE_HOME
$ORACLE_HOME/bin/srvctl start diskgroup -diskgroup DATA
$ORACLE_HOME/bin/srvctl start diskgroup -diskgroup FRA
$ORACLE_HOME/bin/srvctl status diskgroup -diskgroup DATA
$ORACLE_HOME/bin/srvctl status diskgroup -diskgroup FRA
```
8. Oracle Database Software Installation on RAC
> **Warning:** IMPORTANT: Database software installation is run ONLY FROM NODE 1 as the oracle user. The installer deploys software to all nodes automatically via SSH.
## 8.1 — Unzip Oracle Database Software on Node 1
```bash
# As oracle user on NODE 1 ONLY
su - oracle
```
# Unzip DB software into DB Oracle Home
cd $ORACLE_HOME    # /u01/app/oracle/product/19.3.0/dbhome_1
unzip -q /stage/19c/LINUX.X64_193000_db_home.zip
# Verify
ls $ORACLE_HOME/runInstaller
## 8.2 — Install Oracle Database Software (Silent)
```bash
# As oracle user on NODE 1 ONLY
su - oracle
cd $ORACLE_HOME
```
# Install DB software for RAC (INSTALL_DB_SWONLY — no DB creation yet)
./runInstaller -silent \
  -responseFile $ORACLE_HOME/install/response/db_install.rsp \
  oracle.install.option=INSTALL_DB_SWONLY \
  UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  ORACLE_HOME=$ORACLE_HOME \
  ORACLE_BASE=/u01/app/oracle \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba \
  oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba \
  oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba \
  oracle.install.db.OSRACDBA_GROUP=racdba \
  oracle.install.db.isRACOneInstall=false \
  oracle.install.db.rac.serverpoolCardinality=0 \
  oracle.install.db.config.starterdb.type=GENERAL_PURPOSE \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
  DECLINE_SECURITY_UPDATES=true
```bash
# Monitor installation
tail -100f /tmp/InstallActions*/installActions*.log
```
### 8.3 — Run root.sh for DB Home on All Nodes
```bash
# As root on NODE 1 first
/u01/app/oracle/product/19.3.0/dbhome_1/root.sh
```
# Then as root on NODE 2
/u01/app/oracle/product/19.3.0/dbhome_1/root.sh
> **Note:** Unlike GI root.sh, the DB home root.sh can be run on both nodes in parallel — it is safe to do so.
9. RAC Database Creation via DBCA
> **Note:** Why DBCA for RAC? DBCA handles all the RAC-specific database creation steps — creating the database on shared ASM storage, configuring multiple instances (one per node), setting up the cluster database registry, and configuring SCAN listeners.
## 9.1 — Create RAC Database (Silent DBCA)
```bash
# As oracle user on NODE 1 ONLY
su - oracle
```
dbca -silent \
  -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbName RACDB \
  -sid RACDB \
  -responseFile NO_VALUE \
  -characterSet AL32UTF8 \
  -sysPassword Oracle_123 \
  -systemPassword Oracle_123 \
  -createAsContainerDatabase false \
  -databaseType MULTIPURPOSE \
  -automaticMemoryManagement false \
  -totalMemory 4096 \
  -storageType ASM \
  -diskGroupName DATA \
  -recoveryAreaDestination +FRA \
  -recoveryAreaSize 51200 \
  -redoLogFileSize 200 \
  -emConfiguration NONE \
  -nodelist racnode1,racnode2 \
  -databaseConfigType RAC \
  -enableArchive true \
  -archiveLogDest +FRA \
  -ignorePreReqs
> **Note:** Key RAC-specific parameters:
-nodelist racnode1,racnode2 → creates database instances on both nodes
-databaseConfigType RAC → creates RAC database (not standalone)
-storageType ASM → use ASM for storage (required for RAC)
-diskGroupName DATA → primary diskgroup for datafiles
-recoveryAreaDestination +FRA → FRA diskgroup for recovery files
```bash
# Monitor DBCA progress
tail -100f /u01/app/oracle/cfgtoollogs/dbca/RACDB/RACDB*.log
```
## 9.2 — Configure RAC Database Services
> **Note:** Why services? In RAC, applications should connect via a named service rather than directly to an instance. Services provide load balancing (connections distributed across nodes) and High Availability (if a node fails, the service starts on surviving nodes automatically).
```bash
# As oracle user — create an application service
su - oracle
```
# Add a service that runs on both nodes (preferred on node1, available on node2)
$ORACLE_HOME/bin/srvctl add service \
    -db RACDB \
    -service RACDB_APP \
    -preferred racnode1 \
    -available racnode2 \
    -failovertype SELECT \
    -failovermethod BASIC \
    -failoverretry 3 \
    -failoverdelay 3
# Start the service
$ORACLE_HOME/bin/srvctl start service -db RACDB -service RACDB_APP
# Verify service status
$ORACLE_HOME/bin/srvctl status service -db RACDB -service RACDB_APP
# List all services
$ORACLE_HOME/bin/srvctl config service -db RACDB
10. Post-Installation Checks
> **Warning:** IMPORTANT: Post-checks for RAC are more extensive than standalone. You must verify cluster health, all node connectivity, ASM, database, services, and listeners. Do not sign off on a RAC installation without completing every check below.
## 10.1 — Check Cluster Status
```bash
# As grid user
su - grid
```
# Full cluster status
$ORACLE_HOME/bin/crsctl check cluster -all
# All CRS resources — everything should be ONLINE
$ORACLE_HOME/bin/crsctl stat res -t
# Cluster nodes list with status
$ORACLE_HOME/bin/olsnodes -n -i -s -t
## 10.2 — Check All CRS Resources Are Online
```bash
su - grid
```
# Check all resources — look for any OFFLINE entries
$ORACLE_HOME/bin/crsctl stat res -t | grep -E "OFFLINE|INTERMEDIATE|UNKNOWN"
# Check specific resources
$ORACLE_HOME/bin/srvctl status scan
$ORACLE_HOME/bin/srvctl status scan_listener
$ORACLE_HOME/bin/srvctl status listener
$ORACLE_HOME/bin/srvctl status asm
$ORACLE_HOME/bin/srvctl status database -db RACDB
$ORACLE_HOME/bin/srvctl status service  -db RACDB
What to look for: No resource should show OFFLINE except for intentionally stopped ones.
## 10.3 — Check Database Instances on All Nodes
```sql
-- Connect from node1
sqlplus / as sysdba
```
set linesize 200
set pagesize 100
col inst_id       for 999
col instance_name for a15
col host_name     for a20
col status        for a12
col version       for a15
col db_name       for a12
col open_mode     for a15
-- Shows all instances across all RAC nodes
SELECT inst_id, instance_name, host_name,
       version, status, database_status
FROM   gv$instance
ORDER BY inst_id;
-- Check database open mode across all instances
SELECT inst_id, name, open_mode, log_mode
FROM   gv$database
ORDER BY inst_id;
What to look for: All instances must show OPEN and ACTIVE.
## 10.4 — Check ASM Diskgroups on All Nodes
```bash
su - grid
sqlplus / as sysasm
```
set linesize 200
set pagesize 100
col inst_id      for 999
col name         for a12
col state        for a12
col total_gb     for 99999.99
col free_gb      for 99999.99
col offline_disks for 999
-- Shows diskgroup status on ALL nodes (gv$ view)
SELECT inst_id, name, state,
       total_mb/1024  total_gb,
       free_mb/1024   free_gb,
       offline_disks
FROM   gv$asm_diskgroup
ORDER BY name, inst_id;
## 10.5 — Check All Datafiles Are Online
```sql
sqlplus / as sysdba
```
set linesize 200
set pagesize 100
col file#           for 999
col tablespace_name for a25
col status          for a12
col name            for a70
SELECT file#, tablespace_name, status, name
FROM   v$datafile
ORDER BY file#;
## 10.6 — Check SCAN Listener and Services
```bash
su - grid
```
# Check SCAN listener status
$ORACLE_HOME/bin/srvctl status scan_listener
# Check services registered with SCAN listener
lsnrctl status LISTENER_SCAN1
lsnrctl status LISTENER_SCAN2
lsnrctl status LISTENER_SCAN3
# Test SCAN connectivity from a client
sqlplus system/Oracle_123@rac-scan:1521/RACDB_APP
## 10.7 — Check Voting Disks
> **Note:** Why? Voting disks are critical for cluster integrity. If the majority of voting disks become unavailable, the cluster shuts down all nodes (to prevent data corruption). Must always have an odd number and majority available.
```bash
su - grid
```
# List all voting disks and their paths
$ORACLE_HOME/bin/crsctl query css votedisk
Expected output:
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   <uid>                            (/dev/oracleasm/ocr1) [OCR]
 2. ONLINE   <uid>                            (/dev/oracleasm/ocr2) [OCR]
 3. ONLINE   <uid>                            (/dev/oracleasm/ocr3) [OCR]
Located 3 voting disk(s).
## 10.8 — Check OCR Integrity
> **Note:** Why? OCR stores the entire cluster configuration. If OCR is corrupted, the cluster cannot start. Verify its integrity after installation.
```bash
su - grid
```
# Check OCR integrity
$ORACLE_HOME/bin/ocrcheck
# Show OCR configuration
$ORACLE_HOME/bin/ocrcheck -config
# List OCR backup status
$ORACLE_HOME/bin/ocrconfig -showbackup
Expected output from ocrcheck:
Oracle Registry Integrity check successful!
Device/File Name    : +OCR
Device/File integrity check succeeded
Overall system integrity check succeeded
## 10.9 — Check Interconnect Configuration
```bash
su - grid
```
# Show interface configuration used by Clusterware
$ORACLE_HOME/bin/oifcfg getif
# Check cluster interconnect latency
$ORACLE_HOME/bin/cluvfy comp nodecon -n racnode1,racnode2 -verbose
## 10.10 — Check Alert Logs on All Nodes
```bash
# Check CRS alert log (Grid/Cluster alert log)
# Run on ALL nodes
tail -200 /u01/app/grid/diag/crs/racnode1/crs/trace/alert.log \
     | grep -E "ORA-|Error|WARNING|EVICTED|RECONFIG"
```
# Check ASM alert log on all nodes
tail -200 /u01/app/oracle/diag/asm/+asm/+ASM1/trace/alert_+ASM1.log \
     | grep -E "ORA-|Error|WARNING"
# Check DB alert log on all nodes
tail -200 /u01/app/oracle/diag/rdbms/racdb/RACDB1/trace/alert_RACDB1.log \
     | grep -E "ORA-|Error|WARNING"
## 10.11 — Run cluvfy Post-Installation Check
> **Note:** Why? Run cluvfy again after installation to confirm the installed cluster meets all requirements. This is your final validation.
```bash
su - grid
```
# Run post-installation verification
$ORACLE_HOME/runcluvfy.sh stage -post crsinst \
    -n racnode1,racnode2 \
    -verbose
What to look for: All checks must show PASSED. Any FAILED item must be investigated and resolved.
## 10.12 — Check Invalid Objects
```sql
sqlplus / as sysdba
```
set linesize 180
set pagesize 100
col owner       for a20
col object_name for a45
col object_type for a25
col status      for a10
SELECT owner, object_name, object_type, status
FROM   dba_objects
WHERE  status = 'INVALID'
ORDER BY owner, object_type, object_name;
SELECT COUNT(*) invalid_count
FROM   dba_objects
WHERE  status = 'INVALID';
```sql
-- Recompile if needed
@?/rdbms/admin/utlrp.sql
```
SELECT COUNT(*) FROM dba_objects WHERE status = 'INVALID';
## 10.13 — Final Component Registry Check
```sql
set linesize 200
set pagesize 100
col comp_name for a50
col version   for a15
col status    for a12
```
SELECT comp_name, version, status
FROM   dba_registry
ORDER BY comp_name;
11. srvctl — Essential RAC Management Commands
> **Note:** What is srvctl? srvctl (Server Control utility) is the primary tool for managing RAC resources — starting, stopping, and checking the status of databases, instances, listeners, services, ASM, and more. Learn these commands — you will use them every day in a RAC environment.
```bash
su - oracle   # or grid for cluster-level resources
```
# -------------------------------------------------------
# Database Management
# -------------------------------------------------------
# Start entire RAC database (all instances on all nodes)
srvctl start database -db RACDB
# Stop entire RAC database
srvctl stop database -db RACDB -stopoption immediate
# Start specific instance on specific node
srvctl start instance -db RACDB -instance RACDB1
# Stop specific instance
srvctl stop instance -db RACDB -instance RACDB1 -stopoption immediate
# Check database status across all nodes
srvctl status database -db RACDB
# Show database configuration
srvctl config database -db RACDB
# -------------------------------------------------------
# Instance Management
# -------------------------------------------------------
# Check instance status
srvctl status instance -db RACDB -instance RACDB1
# -------------------------------------------------------
# Listener Management
# -------------------------------------------------------
# Start listeners on all nodes
srvctl start listener
# Stop listeners on all nodes
srvctl stop listener
# Check listener status
srvctl status listener
# Check SCAN listener
srvctl status scan_listener
srvctl config scan_listener
# -------------------------------------------------------
# Service Management
# -------------------------------------------------------
# Start a service
srvctl start service -db RACDB -service RACDB_APP
# Stop a service
srvctl stop service -db RACDB -service RACDB_APP
# Relocate service from one node to another
srvctl relocate service -db RACDB -service RACDB_APP \
    -oldinst RACDB1 -newinst RACDB2
# Check service status
srvctl status service -db RACDB
# -------------------------------------------------------
# ASM Management
# -------------------------------------------------------
# Check ASM status on all nodes
srvctl status asm
# Start ASM on all nodes
srvctl start asm
# Stop ASM on all nodes
srvctl stop asm
# -------------------------------------------------------
# Node Management
# -------------------------------------------------------
# Check cluster node status
olsnodes -n -s -i -t
# Check VIP status for each node
srvctl status vip -n racnode1
srvctl status vip -n racnode2
# Check SCAN
srvctl status scan
srvctl config scan
12. Quick Reference Card
This SOP covers everything you need to install Oracle RAC 19c on Linux from scratch without referring to any other source. Always complete the IP planning table before touching any server, always run cluvfy before and after installation, and always verify all CRS resources are online before handing over the environment.
