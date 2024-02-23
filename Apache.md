Download Cent OS ISO from here: http://centos.excellmedia.net/7.9.2009/isos/x86_64/

Choose the ISO file named `CentOS-7-x86_64-DVD-2009.iso`

Watch this video to install Cent OS: https://www.youtube.com/watch?v=AmGWCOKu0mk

Note: while watching the video, at `13:44` minutes when he selects packages do not select the following packages:
- PostgreSQL Database Server
- MariaDB Database Server
- Java platform

Open a terminal:
Click on Applications-> System Tools-> Konsole 

Run:
To switch to root user: (run this again if at any point you have to restart your PC)
`su`

Run:
`yum -y upgrade`

## Configuring the network
Before going any further, make sure that “bridge-utils” and “net-tools” are installed and available:
`yum install bridge-utils net-tools -y`

We will start by creating the bridge that Cloudstack will use for networking. Create and open /etc/sysconfig/network-scripts/ifcfg-cloudbr0 and add the following settings:

```
DEVICE=cloudbr0
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=static
IPV6INIT=no
IPV6_AUTOCONF=no
DELAY=5
IPADDR=172.16.10.2 #(or e.g. 192.168.1.2)
GATEWAY=172.16.10.1 #(or e.g. 192.168.1.1 - this would be your physical/home router)
NETMASK=255.255.255.0
DNS1=8.8.8.8
DNS2=8.8.4.4
STP=yes
USERCTL=no
NM_CONTROLLED=no
```

Save the configuration and exit. We will then edit the NIC so that it makes use of this bridge.

Open the configuration file of your NIC (e.g. /etc/sysconfig/network-scripts/ifcfg-eth0) and edit it as follows:

Note:
Interface name (eth0) used as example only. Replace eth0 with your default ethernet interface name.
```
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
NAME=eth0
DEVICE=eth0
ONBOOT=yes
BRIDGE=cloudbr0
```

Now that we have the configuration files properly set up, we need to run a few commands to start up the network:
```
systemctl disable NetworkManager; systemctl stop NetworkManager
systemctl enable network
reboot
```

##  Hostname
CloudStack requires that the hostname is properly set. If you used the default options in the installation, then your hostname is currently set to `localhost.localdomain`. To test this we will run:

`hostname --fqdn`

At this point it will likely return:
localhost

To rectify this situation - we’ll set the hostname by editing the /etc/hosts file so that it follows a similar format to this example (remember to replace the IP with your IP which might be e.g. 192.168.1.2):
```
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
<your ip address without angle brackets> srvr1.cloud.priv
```

After you’ve modified that file, go ahead and restart the network using:
`systemctl restart network`

Now recheck with the:
`hostname --fqdn`
and ensure that it return `srvr1.cloud.priv`

## SELinux
To configure SELinux to be permissive in the running system we need to run the following command:
`setenforce 0`


To ensure that it remains in that state we need to configure the file /etc/selinux/config to reflect the permissive state, as shown in this example:
```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
# disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of these two values:
# targeted - Targeted processes are protected,
# mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

## NTP
`yum -y install ntp`
`systemctl enable ntpd`
`systemctl start ntpd`

## Configuring the CloudStack Package Repository

To add the CloudStack repository, create /etc/yum.repos.d/cloudstack.repo and insert the following information:
```
[cloudstack]
name=cloudstack
baseurl=http://download.cloudstack.org/centos/$releasever/4.18/
enabled=1
gpgcheck=0
```

## NFS
`yum -y install nfs-utils`

We now need to configure NFS to serve up two different shares. This is handled in the /etc/exports file. You should ensure that it has the following content:
```
/export/secondary *(rw,async,no_root_squash,no_subtree_check)
/export/primary *(rw,async,no_root_squash,no_subtree_check)
```

We’ll go ahead and create those directories and set permissions appropriately on them with the following commands:
`mkdir -p /export/primary`
`mkdir /export/secondary`

CentOS 7.x releases use NFSv4 by default. NFSv4 requires that domain setting matches on all clients. In our case, the domain is cloud.priv, so ensure that the domain setting in /etc/idmapd.conf is uncommented and set as follows:
`Domain = cloud.priv`

Now you’ll need to add the configuration values at the bottom in the file /etc/sysconfig/nfs (or merely uncomment and set them):
```
LOCKD_TCPPORT=32803
LOCKD_UDPPORT=32769
MOUNTD_PORT=892
RQUOTAD_PORT=875
STATD_PORT=662
STATD_OUTGOING_PORT=2020
```

we need to disable the firewall, so that it will not block connections:
`systemctl stop firewalld`
`systemctl disable firewalld`

We now need to configure the nfs service to start on boot and actually start it on the host by executing the following commands:
`systemctl enable rpcbind`
`systemctl enable nfs`
`systemctl start rpcbind`
`systemctl start nfs`

## Management Server Installation

###  Database Installation and Configuration

We’ll start with installing MySQL and configuring some options to ensure it runs well with CloudStack:
`yum -y install wget`
`wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm`
`rpm -ivh mysql-community-release-el7-5.noarch.rpm`

Install by running the following command:
`yum -y install mysql-server`

With MySQL now installed we need to make a few configuration changes to /etc/my.cnf. Specifically we need to add the following options to the [mysqld] section:
```
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
```

Now that MySQL is properly configured we can start it and configure it to start on boot as follows:
`systemctl enable mysqld`
`systemctl start mysqld`

### MySQL Connector Installation
`yum -y install mysql-connector-python`


## Installation
We are now going to install the management server. We do that by executing the following command:
`yum -y install cloudstack-management`

With the application itself installed we can now setup the database, we’ll do that with the following command and options:
`cloudstack-setup-databases cloud:password@localhost --deploy-as=root`

Now that the database has been created, we can take the final step in setting up the management server by issuing the following command:
`cloudstack-setup-management`

