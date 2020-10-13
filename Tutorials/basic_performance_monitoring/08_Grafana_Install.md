# Setting Up a Basic Performance Monitoring Server

## Install Grafana

### Introduction
TODO
* Chronograf built for data exploration and designing queries
* Chrongraf only supports InfluxDB
* Flexible
* Plugins
* Multiple dashboards
* Multple datasources
* Customisable time ranges

### Security Note
This installation of Grafana we will be doing in this section should in no way be accessible from any untrusted network.  

It will not be setup for authentication or encrypted traffic.

### Setting Up Repositories
Grafana maintain a set of pre-built packages for Debian-based distributions.  This makes installation easy as well as 
allowing for simple updates to deal with bug fixes and security issues.

First, lets install the HTTPS transport so we can get packages over a secure connection

```
$ apt install -y apt-transport-https
```

Next we need to add the Grafana GPG key so the operating system trusts the source of the packages:

```
$ wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
OK
```

The simple OK response indicates the key was installed correctly.

Last thing, we need to tell Ubuntu where to get the packages from, we just need to add the package repo to apt:

```
$ echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list 
```

The command should echo back the source path without error.


### Installing Packages
Now the repository is setup we can get on with installing Grafana.

First, update the package sources
```
sudo apt update
```

You should see *packages.grafana.com* as one of the sources.

And next install Grafana:
```
sudo apt -y install grafana
```

### Basic Grafana Configuration
For a basic setup the default configuration for Grafana should be sufficient.

By default is will run on TCP port 3000 and will use an SQlite database for its own configuration.

If you do wish to change anything, the configuration can be found in `/etc/grafana/grafana.ini`

### Starting the Server
By now Grafana should be installed but it doens't start by default.  Use the command `systemctl status grafana-server` to confirm.

If it shows `inactive (dead)` then it's not running. Inform SystemD of the new configuration:
```
$ sudo systemctl daemon-reload
```

```
$ sudo systemctl start grafana-server
```

Check the status of the service again with `systemctl status grafana-server`. You should see it as `active (running)` and in the logs an entry
for "HTTP Server Listen" with address ending in "3000".

### Setting up port forwarding
If you have been following this tutorial, we setup the server in an isolated network so it's not possible to access Grafana directly. You will need 
to know the IP address of the server which can be found with the "ip add" command.

Below are the instructions for configuring VirtualBox to expose the Grafana Server, "S.S.S.S" is the servers real IP:
* In the VirtualBox Manager, select File > Preferences.
* Click Network and select the NAT network you VM is connected to and then click *Edit* (3rd icon down on the right).
* The *Network CIDR* should match with the network the S.S.S.X IP address of your VM is on. Click *Port Forwarding*.
* In the Port Forwarding Rules windows, click "Add" (top icon on the right).
* Name the rule as appropriate, I'm going with "MONSVR01 - Grafana".
* Protocol should be TCP and Host IP should be something on the local host (e.g. 127.0.0.1).
* Host Port will be the port you connect to from your host machines web browser. Unless you have anything already on the default port, enter 3000.
* Guest IP will be S.S.S.S (one you obtained from `ip add` earlier), Guest Port will be default 3000.
* Click OK to save the new rule.
* Click OK to exit the *NAT Network Details* window.
* Finally, click OK to exit the *Virtualbox - Preferences* window.

We should now be all set to see Grafana for the first time.

### Accessing the Web Interface
To access the Grafana web interface, open your web browser and goto http://localhost:3000

You will then be prompted to login (Default username/password is admin/admin)

If Grafana detets you are using the default username/password it will ask to change the password at every login attempt.  Up to you if you change it at the time or not 
(don't forget the new password if you do).

Grafana presents a default *Home* dashboard with some useful information.  Feel free to explore but leave setting up data sources and dashboards until the end of this section.


### Adding Our Influx Data Source
If you left the server running for a while after the last section, there should be a good set of data for us to display in Grafana. 

Let's add our InfluxDB data source to Grafana as folows:
* On the left-hand tool bar, hover over the *Configuration* icon (small cog) and click *Data Sources*
* Click *Add data source*
* Select *InfluxDB* from the available options
* Enter a unique name for the data source (e.g. InfluxDB - Monitoring)
* Enter the URL *http://localhost:8086*
* Enter the name of your *Database* (e.g. monitoring). This will be used as the default unless specified in a query.
* Click *Save & Test*

If all is setup correct, you should get a positive green message *Data source is working*. Click Back


### Setting up a scratchpad dashboard
I always find it's helpful to have a scratchpad dashboard setup where I can experiment so I can try out new visualisations without breaking any used in production.

On the left sidebar, hover over *Add* (plus icon) and then click Dashboard By default, the dashboard is very unimaginatively named *New Dashboard*, lets's change this.  

Click the dashboard settings icon (cog in the top right corner), change the name to *Scratchpad* and then select the *Go Back* icon (left arrow in the left corner)

The dashboard is not that interesting to start, lets add some simple single counters

Click *Add New Panel*


The *Stat* panel allows us to show a single statistic which can be useful for giving an overview of the environment.  In our case we going to set up a crewd counter for
displaying how many hosts are reporting as up or down (based on the ping `result_code`).


Under Panel > Settings, enter the *Panel Title* of `Hosts Responding`. Next select the *Stat* panel in the *Visualisation* section in the right sidebar.  You should et a message for `No data`. Let's write our query.

Underneath the main display, the *Query* tab should already be selected. Click the button to toggle *Text Editor Mode*. You should now see a *SELECT* query, delete it and enter the 
following query instead:
```
select count(host_status) from (SELECT last(result_code) as host_status FROM "monitoring"."autogen"."ping" 
where time > now() -5m and result_code = 0  group by url)
```

As long as you have a least one host that is up you should now get a number in the main display. This query is by no means perfect but for now it gets the job done.  One problem you will
have is when no hosts up it will return to *No Data* because query returns no results (a *Null* result).  Try this now by changing the query to read `result_code = -1`, if you click out 
of the query the display will refresh to *No Data*.

Before you save the query, let's fix this problem so it shows the number zero instead of *No Data*.  In the right-side bar:
* Click the Field tab
* Under *Standard Options*, look for the *No Value* option and enter the number 0 in the box. 

As soon as you click out of the box, the display should now show 0 instead of 'No Data'.  Reset the query back to `result_code = 0` before continuing.

It's probably useful if the dashboard also refreshes, you can change this be selecting a value from the drop down in 
the far-right corner. As checks are only done every 5 minutes, there's not much point in setting it any less than that.

We now need to save the dashboard
* Click *Apply* in the top right corner to return to the main dashboard
* Click the *Save* button (old fashioned floppy disk icon)
* When prompted for a name called it *Scratchpad* or whatever takes your fancy

As a test, I'd like you now to create a second *Stat* panel but this time for hosts that are not responding (or another error).  
The query is the same but you need to use a fitler of `result_code <> 0` otherwsie all the steps above are the same (TIP: There
is a copy option if you're feeling lazy). Don't forget to save after adding the second panel.

It should be noted that this query is not 100% robust.  Depending on when the checks are done and the dashboard is refreshed,
you may see inconsistent results (such as hosts counted in both up and down). This should only be a problem if you are very
intently watching the dashboard, it still works to give you a good overview of the health of your environment.

In later sections we'll add more advanced features (such as data aggregation) to fix this problem.

### Conclusion
In this section we successfully installed Grafana to allow us to setup much more powerful dashboards than can be done 
with Chronograf.

We also confirmed we can take data in from InfluxDB and use InfluxQL queries to display that data graphically.

In the next section we will carry on using Grafana and try out some of it's more advanced queries to make setting up 
our dashboards more efficient.

### Reference

**Grafana Documentation**

https://grafana.com/docs/grafana/latest/
