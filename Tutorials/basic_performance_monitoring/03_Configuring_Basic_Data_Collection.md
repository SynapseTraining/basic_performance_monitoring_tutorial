# Setting Up a Basic Performance Monitoring Server

## Configuring Basic Data Collection

### Introduction

Before we get ahead of ourselves lets try some of basic monitoring.

If every environment you want to know if one of your systems is down or maybe that all important website your business relies on.  One of the common ways to check
this is using ICMP echos, most often called pings.

The program we will use to carry out this monitoring is called Telegraf and forms part of InfluxDatas TICK stack.  It has a number of built in collectors 
(called inputs) as we all built-in support for a number of ways of storing the data, one being InfluxDB (the I in TICK).

### Notes of what to ping
If you plan to purely monitor things within your own environment this section probably doesn't matter but if you plan to monitor external devices (such as 
google.com) or you work in a secure environment with devices such as IPS sensors, expect to get yourself blocked if you send too many requests.

We wil help prevent this in a small way by changing the check interval from the default of every 10 seconds to every 5 minutes but some sites will still see 
this as an attack.

## Setting defaults
The default location for the Telegraf's config is /etc/telegraf.

The main 'telegraf.conf' file stores default settings which control the operation of the agent as a whole as we all for all inputs and outputs.

For individual inputs and outputs, their config can be stored as individual files inside the telegraf.d/ directory which is what we will be doing.

Side note, with Telegraf you're not limited to a single agent.  Having a seperate main .conf file for each agent allows multple agents to be run if 
you find a single agent is not enough.  This is out of scope for this tutorial.

There is a few things we want to change globally:
1. Set a global tag which defines the location of our monitoring agent (Telegraf)
1. Change the check interval at which Telegraf polls each device/endpoint
1. Set the InfluxDB database name to use (default: telegraf)

Open up the config file in a a text editor (e.g. nano):

````
sudo nano /etc/telegraf/telegraf.conf
````

Find the "global_tags" section indicated by square brackets, underneath this add a tag for "site" and a useful name for location your agent is in. Example:

````
[global_tags]
  site = "GB-LDN-DC01"
````

In my case I'm planning for a multi-national setup so I've stuck to a convent of "<country>-<city>-<datacentre>", you may need a different naming convention.


Next lets change the polling interval to something less frequent.  In the "agent" section, find the setting for "interval" and change it to 5 minutes:

````
[agent]
  interval = "5m"
````

Lastly, we need to change the default database.  This isn't strictly necessary but I like to define the purpose for the metrics, later you may find this useful
if you want to have different databases for server, network and maybe applications. To allow a bit more control, we're going to also define a tag which will 
allow the database the be set on individual inputs.

Find the "outpus.influxdb" section and add the following

````
[[outputs.influxdb]]
  database = "monitoring"
  database_tag = "odb"
  exclude_database_tag = true
````

Save the new configuration (if you're using nano, use Ctrl+O) and then exit (Ctrl+X in nano).

## Setting up Ping monitoring
Lets setup some network checks, we'll define these in a new file just to keep things more managable.

Create the new file in the `/etc/telegraf/telegraf.d` directory, if using nano `sudo nano /etc/telegraf/telegraf.d/pings.conf`.

You will need to specify what to ping, in my case I'll be monitoring the router nearest the monitoring server as well as a few google endpoints. 
Modify the config below as needed.

````
# Public Ping Checks
[[inputs.ping]]
  urls = ["1.1.1.1","9.9.9.9","www.google.co.uk"]
  count = 5
  timeout = 1.0
  deadline = 10
  method = "native"


# Private Ping Checks
[[inputs.ping]]
  name_override = "big_ping"
  # Hosts to check
  urls = ["10.0.2.1"]
  method = "exec"
  # Settings used as if running ping manualy
  arguments = ["-M", "do", "-w", "10", "-W", "1", "-c", "5", "-s", "1472"]

````

The first plugin uses Telegraf's built-in method for sending pings and will send a small amount of data to each endpoint (url) 5 times. 
If each ping (echo) doesn't receive a reply (echo reply) within 1 second, it is stopped and will be considered lost. If the entire ping runs 
for more than 10 seconds, the entire ping will be cancelled.

The second plugin uses the linux built-in ping command as we want to send larger pings on our internal network but this isn't natively supported by the Telegraf 
plugin as yet. We also tell ping not fragment the pings either, really giving our network a test.

For more information what all of these options do, see the ping input plugin documentation at the link below:
https://github.com/influxdata/telegraf/blob/master/plugins/inputs/ping/README.md


Save the file as before (CTRL+O, followed by CTRL+X in nano).

## Activating the new configuration

Because the configuration has been changed we need to reload the agent, use the command `sudo systemctl telegraf stop` to stop the agent

Check that is has finished flushing data and stopped with the commend `sudo systemctl telegraf status`.  It should show "inactive"

Restart the agent with `sudo systemctl telegraf start`.

And then finally check the status of the agent once more.  It should be "active" and the logs should show the new site tag as well as interval.


## Summary
By now we will built our basic monitoring server and got it collecting some data.

Now would be a good time to take a break as it'll take some time for us to have anything useful to proceed in the next steps. Take an hour or better 
yet leave it running overnight so that when we come back there we have something more to work with.

When you are ready, proceed to [Querying the Monitoring Data](04_Querying_Monitoring_Data.md).
