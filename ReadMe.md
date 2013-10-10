#Forwarding log events to HDFS using Flume
This use-case processes(filters) http log events recieved from apache web server(`httpd`), it does the following:

* Reads logs from webservers using **exec** source
* Filters the log events based on the `status_code`(200, 404, 503) received using **interceptors**, which add status code to the header of the flume event
* Based on the events `status_code` header, the events are redirected to different channels using **multiplexing channel selector**, if no matching status codes are found agnet will fall back to default channel.
* Sinks picks up events form the assigned channel and forwards them to respective directories in hdfs using **hdfs-sink**

##About Flume
Flume is a distributed service for efficiently collecting, aggregating, and moving large amounts of data to a centralised data store. It's architecture is based on streaming data flows and it uses a simple extensible data model that allows for online analytic application. It is robust and fault tolerant with tuneable reliability mechanisms and many failover and recovery mechanisms.

A unit of data in Flume is called an event, and events flow through one or more Flume agents to reach their destination. An event has a byte payload and an optional set of string attributes. An agent is a Java process that hosts the components through which events flow. The components are a combination of sources, channels, and sinks.

A Flume source consumes events delivered to it by an external source. When a source receives an event, it stores it into one or more Flume channels. A channel is a passive store that keeps the event until it's consumed by a Flume sink. The sink removes the event from the channel and puts it into an external repository (i.e. HDFS or HBase) or forwards it to the source of the next agent in the flow. The source and sink within a given agent run asynchronously, with the events staged in the channel.

Flume agents can be chained together to form multi-hop flows. This allows flows to fan-out and fan-in, and for contextual routing and backup routes to be configured.

For more information, see the [Apache Flume User Guide](http://flume.apache.org/FlumeUserGuide.html).

###Flume Interceptors
Interceptors are part of Flume's extensibility model. They allow events to be inspected as they pass between a source and a channel, and the developer is free to modify or drop events as required. Interceptors can be chained together to form a processing pipeline.

Interceptors are classes that implement the `org.apache.flume.interceptor.Interceptor` interface and they are defined as part of a source's configuration. 

* Built-in interceptors allow adding headers such as timestamps, hostname, static markers. 
* Custom interceptors can inspect event payload to create specific headers where necessary.

For more information, see [Flume Interceptors](http://flume.apache.org/FlumeUserGuide.html#flume-interceptors).

###Flume Channel Selectors
Channel Selector facilitates the selection of one or more channels from all configured channels, based on the preset criteria.

Built-in Channel Selectors:

* **Replicating**: for duplicating the events
* **Multiplexing**: for routing based on the event headers (added by interceptors)

###Flume Sink Processors
Sink Processor is responsible for invoking one sink from a specified group of sinks. Sink processors can be used to provide load balancing capabilities over all sinks inside the group or to achieve fail over from one sink to another in case of temporal failure.

Built-in Sink Porcessors:

* **Load Balancing Sink Processor**: provides the ability to load-balance flow over multiple sinks. It maintains an indexed list of active sinks on which the load must be distributed.  Implementation supports distributing load using either via `round_robin` or `random` selection mechanisms.
* **Failover Sink Processor**: maintains a prioritized list of sinks, guaranteeing that so long as one is available events will be processed (delivered).
* **Default Sink Processor**: accepts only a single sink, user is not forced to create processor (sink group) for single sinks. Instead user can follow the `source -> channel -> sink` pattern.

##Testing it out
###Get some logs
Use http-events generator from [cloudwick-labs](https://github.com/cloudwicklabs/datagenerators) to generate logs to a path, follow these instructions to do so:

```
cd ~ && git clone --recursive https://github.com/cloudwicklabs/datagenerators.git
cd datagenerators/http_events
mkdir /var/logs.flume
ruby random_log_gen.rb -f /var/logs.flume/apache.log
```

Create hdfs dir for flume events storage:

```
hadoop fs -mkdir /flume
hadoop fs -chown [USERNAME] /flume
```
where, `USERNAME` is the user who is running the flume agent

Make sure to change the `NAMNODE_HOST` to your fqdn of namenode in the flume configuration file `webserver.conf` and finally start the flume like so:

```
/usr/lib/flume-ng/bin/flume-ng agent \
  -c /etc/flume-ng/conf/ \
  -f /etc/flume-ng/conf/webserver.conf \
  -n webserver \
  -Dflume.root.logger=DEBUG,console &> logs/flume.log
```