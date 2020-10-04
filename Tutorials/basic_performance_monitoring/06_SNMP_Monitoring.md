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
SNMPCOMMUNITY=<your community>
SNMPHOST=<your device ip>

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
of values so querying each one individually woudl be very boring.

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

Any of the numeric values (INTEGER, Counter32, Gauge32) can be used for performance monitoring and the text value (STRING) can be added as tags
 if necessary by Telegraf.

One important note, I choose to use 32-bit counters here as that is not most commonly supported.  If you are trying to monitor devices with
High Capacity (HC) Interfaces, you may need to use the `ifXTable` instead. Exactly what is best for your device depends on the model, best 
to check your vendors documentation. All that will really change is that instead of using queries such as `ifInOctets` you'll need to use
`ifHCInOctets`.  Run the same query to see for yourself.

````
$ snmpwalk -c $SNMPCOMMUNITY -v 2c $SNMPHOST ifXTable
````


### Configuring Telegraf for SNMP
TODO
