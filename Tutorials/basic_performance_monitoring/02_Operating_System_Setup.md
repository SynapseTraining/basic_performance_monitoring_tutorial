# Setting Up a Basic Performance Monitoring Server

# Setting up the operating system and packages

## Important Note ##
This part of the tutorial will not be taking account of any security concerns such as authentication or encryption. Both during and after completing this 
tutorial do not expose any poart of the system onto a public or other untrusted network. 

Later tutorials will cover how to secure the system, for now lets just get it working.

## Environment Setup
1. Install VirtualBox
2. Install Git

## Basic Installation
1. Download Ubuntu 20.04.1 LTS ISO
1. Install manually
1. Install SSH
1. Install Security Updates
1. Reload

# Update the server
Although security updates would have been applied during setup, general bug fix ones wil not.
1. Login as root (sudo -i)
1. Update the local package repo (apt update)
1. Update all packages without prompting (apt upgrade -y)
1. Install useful tools (apt install <package-name>)
.. net-tools
1. Reload the server (reboot)

# Setup port forwards
Managing the server from the Virtualbox session is very frustrating, lets setup up the VirtualBox instance so that we can directly login to the VM:
1. Port Forward for xx22 to the IP of the VM (ip add) on TCP port 22
1. Port forwarding for 8888 to the IP of the VM (ip add) on TCP Port 8888

# Setup the package repositories
For the tutorial we will mainly be using packages provided by InfluxData.  We need to install their repo to get the latest packages
and also any future updates.

First we need to add the Influx Repo key so packages are trusted, a simple OK response indicates success:
```
curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
```

Next, read in the release versions for Ubuntu:
```
source /etc/lsb-release
```

Now we need to add the InfluxData repo as a package source:
```
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```

Lets just verify that the package path was added correctly using the command `cat /etc/sources.list.d/influxdb.list`. Output should show:
```
deb https://repos.influxdata.com/ubuntu focal stable
```

Once again update the cached package lists (`sudo apt update`). You should note a couple of "repos.influxdata.com" entries.

Assuming no error, lets install the tools we need:
```
apt install -y telegraf influxdb chronograf
```

If the installation succeeeds the telegraf and chronograf should be started automatically, verify with `systemctl status` followed by either `telegraf` or `chronograf`.  
active (running) indicates the service is up.

InfluxDB is not started automatically, run the command below to start it and check its status:
```
systemctl start influxdb
systemctl status influxdb
```

Once they are all running very the ports are listing:
```
netstat -atn | egrep "(8888|8086)"
``` 
The two TCP entries should show as LISTEN

## Summary
In this section we have now sucessfully setup the basic server and can now get onto the more fun bits of making it do something. 
Continue on to [Configuring Data Collection](03_Configuring_Basic_Data_Collection.md)

# Reference
## InfluxData
**Package Dowloads**

https://portal.influxdata.com/downloads/

**Network Requirements**
* Chronograf web interface port - TCP 8888
* InfluxDB Default port - TCP 8086
