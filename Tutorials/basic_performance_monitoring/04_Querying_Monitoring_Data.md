# Setting Up a Basic Performance Monitoring Server

## Querying the Monitoring Data

### Introduction
Hopefully you followed my advise and took a break, maybe you went for a walk got some fresh air...whatever.

If so, there should have been some nice juicy data being following into the database. In this section we'll 
take a look at the data and understand more how it is structed.

### Understanding the Influx Line Protocol
The default method of storing data is to store it in InfluxDB, that's why we only needed to configure the input and nothign about the output.

Telegraf writes metrics to InfluxDB using the folowing format, known as Influx line protocol:

````
<measurement>,[tag_set] [field_set] <timestamp>
````

The measurement from Telegrafs perspective is the name of the input so from our configuration we wil have two measures; ping and big_ping.

The tag_set are static values that are added to every measurement, we defined our own called "site" and Telegraf will add a couple of it's own "host" and "url".

The field_set are the actual values we are interested in. Ping will output values for response time, result of the ping, packet_loss amongst other things.

Timestamp is the date/time that the metric was recorded, if you look at it you'll initially think this can't be right.  The timestemp is expressed in unix time
which is the number of seconds from a specific date. Don't worry we'll do something to make the display a little easier.

An example metric collection in line protocol could be:
ping,site=UK-LDN-DC01 result_code=0 1465839830100400200

Once you get used to it is pretty easier to read, more so than some other tools anyway.

### Starting the Influx Client and Running a Query
To start the influx client you can just run it using the `influx` command but hang on.  If you just run it like that you'll have to deal with those nasty
 timestamps.

Instead run it with an arument that will covert time the time to something more usable, `influx -precision rfc3339`.  You can also use the command 
`precision rfc3339` inside the influx client, if you forget.

Once the client is running switch to the appropriate database with the command `use monitoring`.

Lets check if our ping data has been getting stored.  Use the command `show measurements`, should get something like this:

````
> show measurements
name: measurements
name
----
big_ping
cpu
disk
diskio
kernel
mem
ping
processes
swap
system

````

If ping and big_ping are showing we are good to continue. Don't worry about the other measurements for now.

To check we have data for all the targets, use the command `show series from ping` and `show series from big_ping`. One line should show for each target:

````
> show series from ping, big_ping
key
---
big_ping,host=monsvr01,site=GB-LDN-DC01,url=10.0.2.1
ping,host=monsvr01,site=GB-LDN-DC01,url=1.1.1.1
ping,host=monsvr01,site=GB-LDN-DC01,url=9.9.9.9
ping,host=monsvr01,site=GB-LDN-DC01,url=www.google.co.uk

````

This is a good way to ensure all your targets are being polled without having to see all the individual measurements. Note how the measurement keys are 
returned in alphabetic order, not the order we requested them.

Ok final piece of the puzzle lets see what values are being collected that we can actually monitor.  use the command `show  field keys from ping, big_ping`:
````
> show field keys from ping,big_ping
name: big_ping
fieldKey              fieldType
--------              ---------
average_response_ms   float
maximum_response_ms   float
minimum_response_ms   float
packets_received      integer
packets_transmitted   integer
percent_packet_loss   float
result_code           integer
standard_deviation_ms float
ttl                   integer

name: ping
fieldKey              fieldType
--------              ---------
average_response_ms   float
maximum_response_ms   float
minimum_response_ms   float
packets_received      integer
packets_transmitted   integer
percent_packet_loss   float
result_code           integer
standard_deviation_ms float
ttl                   integer
````

Each of the fieldKeys is a value we can work with, for example graphing response time or counting number of hosts not responding using the result_code.

Lets put all this together and query the measurements being collected. To do this we have to use a SELECT query.

Influx uses a query style very similar to standard SQL but with it's on variances, known as InfluxQL.  If you've used MS SQL or MySQL in the past, this shouldn't look 
too different, for example:

````
select result_codem url from ping, big_ping order by time desc limit 10
````

This statement will return the 10 most recent results for both our inputs.  Hopefully your seeing a result_code of "0" (zero) which means the hosts are responding.

If you want to query for a specific time period, a query like below can be run:

````
> select result_code,url from ping,big_ping where time >= '2020-10-03T16:00:00Z' and time <= '2020-10-03T17:00:00Z' order by time desc limit 10
````


### Summary
In this section we confirmed that we could access the database and query the data being collected.

Knowing how to query the data is essential for setting up more useful visualisations and in the future alerting.  Let's move away from the CLI use 
another to the TICK stack tools known as Chrongraf to [setup up graphs](05_Building_Graphs.md)
