# Timely Server

## Starting Timely

> Note: The Timely server requires a Java 8 runtime. Timely utilizes iterators for Apache Accumulo, so your Accumulo instance will need to be run with Java 8 also.

If you are just starting out with Timely and want to see what it can do, then start up the standalone server using the `bin/timely-standalone.sh` script. This will start up the necessary HDFS, Accumulo, and Timely processes on your local machine. You can use the `bin/insert-test-data*.sh` scripts to push same fake data to the Timely server.

<aside class="warning">
The standalone server will not save your metric data across restarts.
</aside>

To deploy Timely with a running Accumulo instance you will need to modify the `conf/timely.properties file appropriately. Then copy the `lib/timely-server.jar` file to your Accumulo tablet servers. Finally, launch Timely using the `bin/timely-server.sh` script.

## Configuration

The `NUM_SERVER_THREADS` variable in the `timely-server.sh` script controls how many threads are used in the Netty event group for TCP and HTTP operations. The TCP and HTTP groups use a different event group, so if you set the value to 8, then you will have 8 threads for TCP operations and 8 threads for HTTP operations. The properties file in the `conf` directory supports the following properties:

> Note: Each thread in the Timely server that is used for processing TCP put operations has its own BatchWriter. Each BatchWriter honors the `timely.write.latency` and `timely.write.threads` configuration property, but the buffer size for each BatchWriter is `timely.write.buffer.size` divided by the number of threads. For example, if you have 8 threads processing put operations and the following settings, then you will have 8 BatchWriters each using 2 threads with a 30s latency and a maximum buffer size of 128M: timely.write.latency=30s, timely.write.threads=2 timely.write.buffer.size=1G

> Note: The `timely.scanner.threads` property is used for BatchScanners on a per query basis. If you set this to 32 and have 8 threads processing HTTP operations, then you might have 256 threads concurrently querying your tablet servers. Be sure to set your ulimits appropriately.

> Note: The Timely server contains an object called the meta cache, which is a cache of the keys that would be inserted into the meta table if they did not already exist. This cache is used to reduce the insert load of duplicate keys into the meta table and to serve up data to the `/api/metrics` endpoint. The meta cache is an object that supports eviction based on last access time and can be tuned with the `timely.meta.cache.*` properties.

Property | Description | Default Value
---------|-------------|--------------
timely.ip | The ip address where the Timely server is running |
timely.port.put | The port that will be used for processing put requests |
timely.port.query | The port that will be used for processing query requests |
timely.port.websocket | The port that will be used for processing web socket requests |
timely.instance\_name | The name of your Accumulo instance |
timely.zookeepers |  The list of Zookeepers for your Accumulo instance |
timely.username | The username that Timely will use to connect to Accumulo |
timely.password | The password of the Accumulo user |
timely.table | The name of the metrics table | timely.metrics
timely.meta | The name of the meta table | timely.meta
timely.write.latency | The Accumulo BatchWriter latency | 5s
timely.write.threads | The Accumulo BatchWriter number of threads | [default](https://github.com/apache/accumulo/blob/master/core/src/main/java/org/apache/accumulo/core/client/BatchWriterConfig.java#L49)
timely.write.buffer.size | The Accumulo BatchWriter buffer size | [default](https://github.com/apache/accumulo/blob/master/core/src/main/java/org/apache/accumulo/core/client/BatchWriterConfig.java#L40)
timely.metric.age.off.days | The number of days to keep metrics | 7
timely.cors.allow.any.origin | Allow any origin in cross origin requests (true/false) | false
timely.cors.allow.null.origin | Allow null origins in cross origin requests (true/false) | false
timely.cors.allowed.origins | List of allowed origins in cross origin requests (can be null or comma separated list)
timely.cors.allowed.methods | List of allowed methods for cross origin requests | <ul><li>DELETE</li><li>GET</li><li>HEAD</li><li>OPTIONS</li><li>PUT</li><li>POST</li></ul>
timely.cors.allowed.headers | Comma separated list of allowed HTTP headers for cross origin requests | content-type
timely.cors.allow.credentials | Allow credentials to be passed in cross origin requests (true/false) | true
timely.scanner.threads | Number of BatchScanner threads to be used in a query | 4
timely.metrics.report.tags.ignored | Comma separated list of tags which will not be shown in the /api/metrics response |
timely.meta.cache.expiration.minutes | Number of minutes after which unaccessed meta information will be purged from the meta cache | 60
timely.meta.cache.initial.capacity | Initial capacity of the meta cache | 2000
timely.meta.cache.max.capacity | Maximum capacity of the meta cache | 10000
timely.ssl.certificate.file | Public certificate to use for the Timely server (x509 pem format) |
timely.ssl.key.file | Private key to use for the Timely server (in pkcs8 format) |
timely.ssl.key.pass | Password to the private key |
timely.ssl.use.generated.keypair | Use a generated certificate/key pair - useful for testing | false
timely.ssl.trust.store.file | Certificate trust store (a concatenated list of trusted CA x509 pem certificates) |
timely.ssl.use.openssl | Use OpenSSL (vs JDK SSL) | true
timely.ssl.use.ciphers | List of allowed SSL ciphers | see Configuration.java
timely.session.max.age | Setting for max age of session cookie (in seconds) | 86400
timely.http.host | Address for the Timely server, used for the session cookie domain |
timely.allow.anonymous.access | Allow anonymous access | false
timely.visibility.cache.expiration.minutes | Column Visibility Cache Expiration (minutes) | 60
timely.visibility.cache.initial.capacity | Column Visibility Cache Initial Capacity | 2000
timely.visibility.cache.max.capacity | Column Visibility Cache Max Capacity | 10000
timely.web.socket.timeout | Number of seconds with no client ping response before closing subscription | 60
timely.ws.subscription.lag | Number of seconds that subscriptions should lag to account for latency | 120
timely.http.redirect.path | Path to use for HTTP to HTTPS redirect | /secure-me
timely.hsts.max.age | HTTP Strict Transport Security max age (in seconds) | 604800

## Data Storage

Metrics sent to Timely are stored in two Accumulo tables, `meta` and `metrics`.

### Meta Table Format

Row | ColumnFamily | ColumnQualifier | Value
----|--------------|-----------------|------
m:metric | | |
t:metric | tagKey | |
v:metric | tagKey | tagValue |

### Metric Table Format

Row | ColumnFamily | ColumnQualifier | Value
----|--------------|-----------------|------
metric\timestamp | tagKey=tagValue | tagKey=tagValue,tagKey=tagValue,... | metricValue

### Storage Example

As an example, if you sent the following metric to the put api: `sys.cpu.user 1447879348291 2.0 rack=r001 host=r001n01 instance=0` it would get stored in the following manner in the `meta` table:

Row | ColumnFamily | ColumnQualifier | Value
----|--------------|-----------------|------
m:sys.cpu.user | | |
t:sys.cpu.user | host | | 
t:sys.cpu.user | instance | | 
t:sys.cpu.user | rack | | 
v:sys.cpu.user | host | r001n01 | 
v:sys.cpu.user | instance | 0 | 
v:sys.cpu.user | rack | r001 | 

and in the following manner in the `metrics` table

> Note: Not shown in the examples is the encoding of the row and value. You can apply the Timely formatter to your metrics table using the Accumulo shell command: `config -t timely.metrics -s table.formatter=timely.util.TimelyMetricsFormatter`

Row | ColumnFamily | ColumnQualifier | Value
----|--------------|-----------------|------
sys.cpu.user\1447879348291 | host=r001n01 | instance=0,rack=r001    | 2.0
sys.cpu.user\1447879348291 | instance=0   | host=r001n01,rack=r001  | 2.0
sys.cpu.user\1447879348291 | rack=r001    | host=r001n01,instance=0 | 2.0

<aside class="notice">
Timely stores your metric N times in the metrics table, where N is the number of tags in your metric data.
</aside>

## Accumulo Configuration

You can generate split points for the metrics table after sending data to Timely for a short amount of time. The `bin/get-metric-split-points.sh` script will print out a set of split points based on the metric names in the meta table. Yuo can use this output in the Accumulo `addsplits` command.

You can lower the `table.scan.max.memory` property on your metrics table. This will send data back from the Tablet Servers to the Timely server at a faster rate.

If you don't mind losing some metric data in the event of an Accumulo tablet server death, you can set the `table.walog.enabled` property to false and the `table.durability` property to none on your metrics table. This should speed up ingest a little.