# Basic Performance Monitoring Server
## Setting Up SNMP Monitoring

### Installation
First install the SNMP utilities, just for testing

````
sudo apt install -y snmp
````

SNMP mibs are not available by default for license reason but we still need to allow them to be loaded in the config

````
sudo sed -i 's/mibs :/# mibs :/g' /etc/snmp/snmp.conf
````

By defaul the SNMP MIBs onces downloaded are available from /usr/share/snmp/mibs. Lets download the default MIBS:

````
sudo apt install snmp-mibs-downloader
`````

A long stream of files will be listed on the screen as each MIB is downloaded.  Whilst optional downloading the MIBS allows us to specify
values by names instead of having to know the OID. 

Let's test this is working by checking if the snmp tools can translae a MIB name to it's corresponding OID:
````
$ snmptranslate -On -IR ifTable
.1.3.6.1.2.1.2.2
````

The ifTable is an industry standard MIB that almost all network devices will provide statistics on. For further info, past the OID 
(remove the beginning period ".") into the link below:

http://www.oid-info.com/

You should be able to do this for any industry standard MIB/OID you need more info on.

### Verifying connectivity to a device
As stated in the introduction we won't be covering how to configure SNMP v2 or v3 hear for an specific device so I'm going to assume 
you've done that yoursef already.

Firstly store your SNMP community and Device IP in variables, that way the commands be can run exactly as shown

```
SNMPCOMMUNITY=<your community>
SNMPHOST=<your device ip>
````

Let's see if we can obtain some basic information about our network device
````
$ snmpget -c $SNMPCOMMUNITY -v 2c $SNMPHOST sysDescr.0
SNMPv2-MIB::sysDescr.0 = STRING: RouterOS RBmAP2nD
````

In this case, I'm querying one of my labs Mikrotik routers. You should get something similar regardles of it's Cisco, Juniper or whatever you're
using.

### Obtaining Information on Network Interfaces
The `snmpget` command is used to obtain a single value from the SNMP agent running on the network device. Querying Network interfaces needs
something a little more complex.

Do you recall the ifTable mentioned earlier?

The ifTable gives information on all the network interfaces on the device. If you have a high density switch, this could be many hundreds 
of values so querying each one individually would be very boring.

The SNMP utilities however give us another useful command which allows us to automatically traverse the table and show all possible values. Be 
warned the output from this command could be very long, recommend outputting to a file if you want to read it.


````
$ snmpwalk -c $SNMPCOMMUNITY -v 2c $SNMPHOST ifTable > /tmp/snmp_interfaces.txt
````

You should not get any output from this command as we wrote it all to a file. To display it use `less /tmp/snmp_interfaces.txt`. In my case
the device only has four interfaces, here's the output from the first two:
````
IF-MIB::ifIndex.1 = INTEGER: 1
IF-MIB::ifIndex.2 = INTEGER: 2
IF-MIB::ifDescr.1 = STRING: wlan1
IF-MIB::ifDescr.2 = STRING: ether1
IF-MIB::ifType.1 = INTEGER: ieee80211(71)
IF-MIB::ifType.2 = INTEGER: ethernetCsmacd(6)
IF-MIB::ifMtu.1 = INTEGER: 1500
IF-MIB::ifMtu.2 = INTEGER: 1500
IF-MIB::ifSpeed.1 = Gauge32: 50000000
IF-MIB::ifSpeed.2 = Gauge32: 0
IF-MIB::ifPhysAddress.1 = STRING: 64:d1:54:4f:72:86
IF-MIB::ifPhysAddress.2 = STRING: 64:d1:54:4f:72:84
IF-MIB::ifAdminStatus.1 = INTEGER: up(1)
IF-MIB::ifAdminStatus.2 = INTEGER: up(1)
IF-MIB::ifOperStatus.1 = INTEGER: up(1)
IF-MIB::ifOperStatus.2 = INTEGER: down(2)
IF-MIB::ifLastChange.1 = Timeticks: (0) 0:00:00.00
IF-MIB::ifLastChange.2 = Timeticks: (0) 0:00:00.00
IF-MIB::ifInOctets.1 = Counter32: 18027406
IF-MIB::ifInOctets.2 = Counter32: 217063
IF-MIB::ifInUcastPkts.1 = Counter32: 17852
IF-MIB::ifInUcastPkts.2 = Counter32: 1202
IF-MIB::ifInNUcastPkts.1 = Counter32: 0
IF-MIB::ifInNUcastPkts.2 = Counter32: 0
IF-MIB::ifInDiscards.1 = Counter32: 0
IF-MIB::ifInDiscards.2 = Counter32: 0
IF-MIB::ifInErrors.1 = Counter32: 0
IF-MIB::ifInErrors.2 = Counter32: 0
IF-MIB::ifInUnknownProtos.1 = Counter32: 0
IF-MIB::ifInUnknownProtos.2 = Counter32: 0
IF-MIB::ifOutOctets.1 = Counter32: 1945058
IF-MIB::ifOutOctets.2 = Counter32: 66634
IF-MIB::ifOutUcastPkts.1 = Counter32: 9339
IF-MIB::ifOutUcastPkts.2 = Counter32: 371
IF-MIB::ifOutNUcastPkts.1 = Counter32: 0
IF-MIB::ifOutNUcastPkts.2 = Counter32: 0
IF-MIB::ifOutDiscards.1 = Counter32: 0
IF-MIB::ifOutDiscards.2 = Counter32: 0
IF-MIB::ifOutErrors.1 = Counter32: 0
IF-MIB::ifOutErrors.2 = Counter32: 0
IF-MIB::ifOutQLen.1 = Gauge32: 0
IF-MIB::ifOutQLen.2 = Gauge32: 0
````


One important note, I choose to use 32-bit counters here as that is not most commonly supported.  If you are trying to monitor devices with
High Capacity (HC) Interfaces, you may need to use the `ifXTable` instead. Exactly what is best for your device depends on the model, best 
to check your vendors documentation. All that will really change is that instead of using queries such as `ifInOctets` you'll need to use
`ifHCInOctets`.  Run the same query to see for yourself.

````
$ snmpwalk -c $SNMPCOMMUNITY -v 2c $SNMPHOST ifXTable
````

### Understanding SNMP data types
This is not an SNMP tutorial but in understanding what and how to graph a brief explanation may help.

Any of the numeric values (INTEGER, Counter32, Gauge32) can be used for performance monitoring.

INTEGER is a single value which can be positive or negative. Often it wil be mapped to a specific description value, for example with 
ifAdminStatus indicating if a port has been enabled a value of 1 means enable. and 0 means disabled. You could use this for example to only 
store values in the database for interfaces that are up.

Gauge32 is a single positive value within a a defined range.  This is only visible as single type in our output for `ifSpeed` or `ifHighSpeed` 
and tells us the speed at which an interface is running (for example 100 is 100Mbps). Gauge32 will never wrap around. You may also see this for 
values representing percentage (e.g. RAM or CPU Usage).

The final numeric type we have is Counter32 (or Counter64 for High Capacity), is a non-negative value which when the maximum is reached will wrap around. We
see this type a lot in our output as many of the values could go on forever if the counters are not reset.  In the majority of cases we are
not concerned by the raw value but the rate of change between queries.  For example we may want to know Packets (Pkts) or Bytes (Octets) per 
second. SNMP does not calculate that for us automatically, we will need to do some work in the queries so the graphs are correct.

Finally we have non-numeric or text values (STRING) canot be processed direclty by Telegrad or InfluxDB but can be added as tags if necessary by Telegraf to give more context
to the values.  For example adding the `ifName` as a tag means you query details on specific interfaces later.


### Configuring Telegraf for SNMP
Because of the multiple values setting up SNMP in telegraf isn't as simple as our last plugin. Lets get it configured and we can work through 
it step-by-step.

Open you your text editor of choice (e.g. nano) and createa new file in the Telegraf config directory.

````
sudo nano /etc/telegraf/telegraf.d/snmp.conf
````

Now paste all of the following
````
[[inputs.snmp]]
  agents = [ "<DEVICEIP>:161" ]
  version = 2
  community = "<COMMUNITY>"

  [[inputs.snmp.field]]
    name = "hostname"
    oid = "RFC1213-MIB::sysName.0"
    is_tag = true

  [[inputs.snmp.field]]
    name = "uptime"
    oid = "DISMAN-EXPRESSION-MIB::sysUpTimeInstance"

  # IF-MIB::ifTable contains counters on input and output traffic as well as errors and discards.
  [[inputs.snmp.table]]
    name = "interface"
    inherit_tags = [ "hostname" ]
    oid = "IF-MIB::ifTable"

    # Interface tag - used to identify interface in metrics database
    [[inputs.snmp.table.field]]
      name = "ifDescr"
      oid = "IF-MIB::ifDescr"
      is_tag = true

  # IF-MIB::ifXTable contains newer High Capacity (HC) counters that do not overflow as fast for a few of the ifTable counters
  [[inputs.snmp.table]]
    name = "interface"
    inherit_tags = [ "hostname" ]
    oid = "IF-MIB::ifXTable"

    # Interface tag - used to identify interface in metrics database
    [[inputs.snmp.table.field]]
      name = "ifDescr"
      oid = "IF-MIB::ifDescr"
      is_tag = true

  # EtherLike-MIB::dot3StatsTable contains detailed ethernet-level information about what kind of errors have been logged on an interface (such as FCS error, frame too long, etc)
  [[inputs.snmp.table]]
    name = "interface"
    inherit_tags = [ "hostname" ]
    oid = "EtherLike-MIB::dot3StatsTable"

    # Interface tag - used to identify interface in metrics database
    [[inputs.snmp.table.field]]
      name = "ifDescr"
      oid = "IF-MIB::ifDescr"
      is_tag = true
````

You will then need to change the `<SNMPHOST>` and `<SNMPCOMMUNITY>` values to match what we you set in the variables earlier.

Save the file (Ctrl+O in nano) and then exit (CTRL+X in nano).


Now restart Telegraf and check that it has restart sucessfuly

````
sudo systemctl stop telegraf
sudo systemctl start telegraf
systemctl status telegraf
````

In the status output you should now see that it is `active` and there is a new `snmp` loaded input.

Give Telegraf a few minutes to run the queries then run start up the influxdb client

````
influx -precision rfc3339 -database monitoring
````

Lets see if we have any new data
````
show measurements
````

The ones we are interested in are snmp and interface. The snmp measurement just records the uptime of the host so lets concentrate on interface.

To see what values we've got use
````
show field keys from interface
````

All of the values we were shown earlier should now be listed.  Lets get a report on the status of the interfaces

````
select ifOperStatus, hostname, ifDescr from interface
````

All the hosts and their interfaces should be listed. Note unlike SQL it is only possible ot order the results by time with Influx at the moment.

Maybe we only care about interfaces that are currently up, let's see if there are any errors
````
select hostname, ifDesr, ifInErrors, ifOutErrors from interface where ifOperStatus = 1
````


### Calcuating Rates wth Influx
TODO
