---
layout: post
title: Flume 1.5.0.1 User Guide：Flume Sources
description: Flume中重要组件source的详细介绍
categories: flume
---


##Avro Source

Listens on Avro port and receives events from external Avro client streams. When paired with the built-in Avro Sink on another (previous hop) Flume agent, it can create tiered collection topologies. Required properties are in bold.
（Avro sink与Avro source可以构成tiered collection topologies；下表中，必须属性**加黑**了）

|Property Name|Default|Description|
|--|--|--|
|**channels**|–| |	 
|**type**|	–|	The component type name, needs to be `avro`|
|**bind**|	–|	hostname or IP address to listen on|
|**port**|	–|	Port # to bind to|
|threads|	–|	Maximum number of worker threads to spawn|
|selector.type| | |	 	 
|selector.*| |	| 
|interceptors|	–|	Space-separated list of interceptors|
|interceptors.*	| | | 	 
|compression-type|	none|	This can be “none” or “deflate”. The compression-type must match the compression-type of matching `AvroSink`|
|ssl|	false|	Set this to true to enable SSL encryption. You must also specify a “keystore” and a “keystore-password”.|
|keystore|	–|	This is the path to a Java keystore file. Required for SSL.|
|keystore-password|	–	|The password for the Java keystore. Required for SSL.|
|keystore-type|	JKS	The type of the Java keystore. This can be “JKS” or “PKCS12”.|
|ipFilter|	false|	Set this to true to enable ipFiltering for netty|
|ipFilter.rules|	–|	Define N netty ipFilter pattern rules with this config.|

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = avro
	a1.sources.r1.channels = c1
	a1.sources.r1.bind = 0.0.0.0
	a1.sources.r1.port = 4141

Example of `ipFilter.rules`

ipFilter.rules defines N netty ipFilters separated by a comma(`,`) a pattern rule must be in this format.

	<`allow` or `deny`>:<`ip` or `name` for computer name>:<`pattern`> 
	allow/deny:ip/name:pattern
	
	# example
	ipFilter.rules=allow:ip:127.*,allow:name:localhost,deny:ip:*

Note that the first rule to match will apply as the example below shows from a client on the localhost（从左向右，第一个匹配出的rules生效）

	# This will Allow the client on localhost be deny clients from any other ip 
	ipFilter.rules = allow:name:localhost,deny:ip:
	
	# This will deny the client on localhost be allow clients from any other ip 
	ipFilter.rules = deny:name:localhost,allow:ip:


**notes(ningg)**：

* Avro Source提供了属性`threads`，标识能够启动并保持的worker thread的数目；
* 默认情况下，不设置`threads`属性时，worker thread的个数在`[0~infinite]`，并且默认，当worker thread空闲60s后，自动销毁，因此，如果上一级的是一个 Avro Sink，则可能发送 Avro Source 空闲时间过长，所有的worker thread被回收，因此，当上一级的agent（其中包含了 Avro Sink）接收到大量的events时，可能由于当前Avro Source中woker thread不足，导致Channel中event堆积，最终导致agent异常。
* 为避免不设置`threads`属性时，所有的Avro Source的worker thread被回收的情况，有2中解决办法：
	* 设置`threads`属性，但：`threads`取值为多少？因为worker只能启动并保持这些thread；另一个问题：如何测量/监控当前的avro source启动了几个thread？**最佳的设计**是，设置一个thread个数的下限，上线不设限制，具体thread个数根据需要进行动态的创建和销毁；
	* 自定义advanced Avro Source，实现上述**最佳的设计**，*（基于当前对flume的了解，难度很小）*；
* 上述的worker线程回收的问题，是不是也可能是Avro Sink产生的？Avro Sink与下一层的Avro Source之间什么关系？经查询利用的是Netty NIO机制，那什么是NIO？
* 在最远端的Exec Source、Netcat Source实现慢启动？
	
	# 设置 threads 属性
	ThreadPoolExecutor(nThreads, nThreads,
					   0L, TimeUnit.MILLISECONDS,
					   new LinkedBlockingQueue<Runnable>(),
					   threadFactory);

	# 使用默认 threads 属性
	ThreadPoolExecutor(0, Integer.MAX_VALUE,
					   60L, TimeUnit.SECONDS,
					   new SynchronousQueue<Runnable>(),
					   threadFactory);





									


	
##Thrift Source

Listens on Thrift port and receives events from external Thrift client streams. When paired with the built-in ThriftSink on another (previous hop) Flume agent, it can create tiered collection topologies. Required properties are in bold.
（通过Thrift sink和Thrift source，可以构成tired collection topologies；必要的属性都已加黑）

|Property Name|	Default	|Description|
|--|--|--|
|**channels**|	–	 | |
|**type**| – |	The component type name, needs to be thrift|
|**bind**| – |hostname or IP address to listen on|
|**port**| – |Port # to bind to|
|threads| –	|Maximum number of worker threads to spawn|
|selector.type	 |	| |
|selector.*	 | | |
|interceptors|	–	|Space separated list of interceptors|
|interceptors.*	| | | 	 

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = thrift
	a1.sources.r1.channels = c1
	a1.sources.r1.bind = 0.0.0.0
	a1.sources.r1.port = 4141
	
##Exec Source

Exec source runs a given Unix command on start-up and expects that process to continuously produce data on standard out (stderr is simply discarded, unless property logStdErr is set to true). If the process exits for any reason, the source also exits and will produce no further data. This means configurations such as `cat [named pipe]` or `tail -F [file]` are going to produce the desired results where as `date` will probably not - the former two commands produce streams of data where as the latter produces a single event and exits.（捕获命令的输出，并按行进行处理，当`logStdErr`设为true时，也将捕获stderr的输出；）

Required properties are in bold.

|Property Name|	Default|	Description|
|--|--|--|
|**channels**|	–|	 |
|**type**|	–	|The component type name, needs to be exec|
|**command**|	–|	The command to execute|
|shell|	–|	A shell invocation used to run the command. e.g. `/bin/sh -c`. Required only for commands relying on shell features like wildcards, back ticks, pipes etc.|
|restartThrottle|	10000|	Amount of time (in millis) to wait before attempting a restart|
|restart|	false|	Whether the executed cmd should be restarted if it dies|
|logStdErr|	false|	Whether the command’s stderr should be logged|
|batchSize|	20	|The max number of lines to read and send to the channel at a time|
|batchTimeout|3000|Amount of time(ms) to wait, if the buffer size was not reached, before to data is pushed downstream|
|selector.type|	replicating|	replicating or multiplexing|
|selector.*|	| 	Depends on the selector.type value|
|interceptors|	–|	Space-separated list of interceptors|
|interceptors.*	| |	 |

**补充参数**：通过查看源代码，可知：Exec Source还有一个设置参数 `charset`，可以设定读取信息的byte--char--byte之间映射时的编码方式（默认UTF-8），例如，原始日志文档为GBK编码，这只需添加`charset=GBK`即可解决Flume在Source端的乱码问题；不过，如果只是单纯的解决编码问题，可以在Exec Source端设置`charset`，同时，在Sink端以相同的`charset`读取channel中信息即可*（这一方式，没有干预Source，而是在Sink端进行干预，通常情况下，这一做法的难度较小；当然，实际上，在Source、Sink端，进行干预，来解决编码问题，原理上都是可以的，具体场景，具体分析）*

> **Warning**： The problem with ExecSource and other asynchronous sources is that the source can not guarantee that if there is a failure to put the event into the Channel the client knows about it. In such cases, the data will be lost. As a for instance, one of the most commonly requested features is the `tail -F [file]`-like use case where an application writes to a log file on disk and Flume tails the file, sending each line as an event. While this is possible, there’s an obvious problem; what happens if the channel fills up and Flume can’t send an event? Flume has no way of indicating to the application writing the log file that it needs to retain the log or that the event hasn’t been sent, for some reason. If this doesn’t make sense, you need only know this: Your application can never guarantee data has been received when using a unidirectional asynchronous interface such as ExecSource! As an extension of this warning - and to be completely clear - there is absolutely zero guarantee of event delivery when using this source. For stronger reliability guarantees, consider the Spooling Directory Source or direct integration with Flume via the SDK.

**notes(ningg)**：ExecSource方式，当command异常退出后，会丢失数据。解决办法：考虑Spooling Directory Source或者通过SDK直接与Flume集成。

> **Note**：You can use ExecSource to emulate TailSource from Flume 0.9x (flume og). Just use unix command tail -F /full/path/to/your/file. Parameter -F is better in this case than -f as it will also follow file rotation.（Flume 0.9x版本中，可以使用 `tail -F path`命令模仿 TailSource）

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = exec
	# follow file rotation
	a1.sources.r1.command = tail -F /var/log/secure
	a1.sources.r1.channels = c1

The ‘shell’ config is used to invoke the ‘command’ through a command shell (such as Bash or Powershell). The ‘command’ is passed as an argument to ‘shell’ for execution. This allows the ‘command’ to use features from the shell such as wildcards, back ticks, pipes, loops, conditionals etc. In the absence of the ‘shell’ config, the ‘command’ will be invoked directly. Common values for ‘shell’ : ‘/bin/sh -c’, ‘/bin/ksh -c’, ‘cmd /c’, ‘powershell -Command’, etc.（启用shell选项时，系统会将command当作参数，传送给shell执行，此时，能利用不同shell的特性）

	agent_foo.sources.tailsource-1.type = exec
	agent_foo.sources.tailsource-1.shell = /bin/bash -c
	agent_foo.sources.tailsource-1.command = for i in /path/*.txt; do cat $i; done



##JMS Source

JMS Source reads messages from a JMS destination such as a queue or topic. Being a JMS application it should work with any JMS provider but has only been tested with ActiveMQ. The JMS source provides configurable batch size, message selector, user/pass, and message to flume event converter. Note that the vendor provided JMS jars should be included in the Flume classpath using `plugins.d` directory (preferred), –classpath on command line, or via `FLUME_CLASSPATH` variable in `flume-env.sh`.
（JMS从JMS destination中读取message，例如：a queue、topic；）

Required properties are in **bold**.

|Property Name|	Default|	Description|
|--|--|--|
|**channels**|–	| |
|**type**|	–	|The component type name, needs to be jms|
|**initialContextFactory**|	–	|Inital Context Factory, e.g: org.apache.activemq.jndi.ActiveMQInitialContextFactory|
|**connectionFactory**|	–|	The JNDI name the connection factory shoulld appear as|
|**providerURL**|	–|	The JMS provider URL|
|**destinationName**|	–	|Destination name|
|**destinationType**|	–	|Destination type (queue or topic)|
|messageSelector|	–	|Message selector to use when creating the consumer|
|userName	|–	|Username for the destination/provider|
|passwordFile|	–	|File containing the password for the destination/provider|
|batchSize|	100	|Number of messages to consume in one batch|
|converter.type|	DEFAULT|	Class to use to convert messages to flume events. See below.|
|converter.*|	–	|Converter properties.|
|converter.charset|	UTF-8	|Default converter only. Charset to use when converting JMS TextMessages to byte arrays.|


**notes(ningg)**：JMS是java方面的消息队列？做几年java了，对这个我还不清楚~~

###Converter

The JMS source allows pluggable converters, though it’s likely the default converter will work for most purposes. The default converter is able to convert Bytes, Text, and Object messages to FlumeEvents. In all cases, the properties in the message are added as headers to the FlumeEvent.（默认，message的properties会转换为FlumeEvent的headers）

* BytesMessage: Bytes of message are copied to body of the FlumeEvent. Cannot convert more than 2GB of data per message.
* TextMessage: Text of message is converted to a byte array and copied to the body of the FlumeEvent. The default converter uses UTF-8 by default but this is configurable.
* ObjectMessage: Object is written out to a ByteArrayOutputStream wrapped in an ObjectOutputStream and the resulting array is copied to the body of the FlumeEvent.

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = jms
	a1.sources.r1.channels = c1
	a1.sources.r1.initialContextFactory = org.apache.activemq.jndi.ActiveMQInitialContextFactory
	a1.sources.r1.connectionFactory = GenericConnectionFactory
	a1.sources.r1.providerURL = tcp://mqserver:61616
	a1.sources.r1.destinationName = BUSINESS_DATA
	a1.sources.r1.destinationType = QUEUE

##Spooling Directory Source

This source lets you ingest data by placing files to be ingested into a “spooling” directory on disk. This source will watch the specified directory for new files, and will parse events out of new files as they appear. The event parsing logic is pluggable. After a given file has been fully read into the channel, it is renamed to indicate completion (or optionally deleted).
（设定一个`spooling`目录，Spooling Directory Source将会监控这一目录中的new files，并且解析其中新增的内容——生成event；解析file的逻辑，是可以配置的；当一个file全部读入channel后，通常会rename/delete这个file，来标识此文件已经被处理）

Unlike the Exec source, this source is reliable and will not miss data, even if Flume is restarted or killed. In exchange for this reliability, only immutable, uniquely-named files must be dropped into the spooling directory. Flume tries to detect these problem conditions and will fail loudly if they are violated:
（Spooling Directory Source是可靠的source，即使flume重启，也不会丢失数据；但有一个要求：spooling directory目录下的文件要求immutable、uniquely-named；Flume会检查这一约束条件）

1. If a file is written to after being placed into the spooling directory, Flume will print an error to its log file and stop processing.（spooling directory下的file，如果被改动，则Flume就不干了，他会在log文件中print error并且终止process）
1. If a file name is reused at a later time, Flume will print an error to its log file and stop processing.（如果file name被reuse，则Flume也会终止处理）

To avoid the above issues, it may be useful to add a unique identifier (such as a timestamp) to log file names when they are moved into the spooling directory.
（为了解决上面的问题，通常在file name中添加timestamp）

Despite the reliability guarantees of this source, there are still cases in which events may be duplicated if certain downstream failures occur. This is consistent with the guarantees offered by other Flume components.
（这一source有可靠性保证，不过可能引发event重复发送问题；其他有可靠性保证的source，也会有类似情况）

|Property Name|	Default|	Description|
|--|--|--|
|**channels**|	–	 |  |
|**type**|	–	|The component type name, needs to be `spooldir`.|
|**spoolDir**|	–	|The directory from which to read files from.|
|fileSuffix|	.COMPLETED|	Suffix to append to completely ingested files|
|deletePolicy|	`never`|	When to delete completed files: `never` or `immediate`|
|fileHeader|	false|	Whether to add a header storing the absolute path filename.|
|fileHeaderKey|	file|	Header key to use when appending absolute path filename to event header.|
|basenameHeader|	false|	Whether to add a header storing the basename of the file.|
|basenameHeaderKey|	basename|	Header Key to use when appending basename of file to event header.|
|ignorePattern|	^$	|Regular expression specifying which files to ignore (skip)|
|trackerDir|	.flumespool|	Directory to store metadata related to processing of files. If this path is not an absolute path, then it is interpreted as relative to the spoolDir.|
|consumeOrder|	oldest	|In which order files in the spooling directory will be consumed `oldest`, `youngest` and `random`. In case of `oldest` and `youngest`, the last modified time of the files will be used to compare the files. In case of a tie, the file with smallest laxicographical order will be consumed first. In case of `random` any file will be picked randomly. When using `oldest` and `youngest` the whole directory will be scanned to pick the oldest/youngest file, which might be slow if there are a large number of files, while using random may cause old files to be consumed very late if new files keep coming in the spooling directory.|
|maxBackoff	|4000	|The maximum time (in millis) to wait between consecutive attempts to write to the channel(s) if the channel is full. The source will start at a low backoff and increase it exponentially each time the channel throws a ChannelException, upto the value specified by this parameter.|
|batchSize	|100|	Granularity at which to batch transfer to the channel|
|inputCharset|	UTF-8|	Character set used by deserializers that treat the input file as text.|
|decodeErrorPolicy|	`FAIL`|	What to do when we see a non-decodable character in the input file. `FAIL`: Throw an exception and fail to parse the file. `REPLACE`: Replace the unparseable character with the “replacement character” char, typically Unicode `U+FFFD`. `IGNORE`: Drop the unparseable character sequence.|
|deserializer|	`LINE`|	Specify the deserializer used to parse the file into events. Defaults to parsing each line as an event. The class specified must implement `EventDeserializer.Builder`.|
|deserializer.*	| 	|Varies per event deserializer.*(设置每个deseralizer的实现类，对应的配置参数)*|
|bufferMaxLines|	–	|(Obselete) This option is now ignored.|
|bufferMaxLineLength|	5000|	(Deprecated) Maximum length of a line in the commit buffer. Use `deserializer.maxLineLength` instead.|
|selector.type|	`replicating`|	`replicating` or `multiplexing`|
|selector.*	| 	|Depends on the `selector.type` value|
|interceptors|	–	|Space-separated list of interceptors|
|interceptors.*	|  |  |

**notes(ningg)**：关于Spooling Directory Source的配置参数，几点：

* `fileHeader`、`basenameHeader`设定Event Header（`Map<Stirng, String>`）中是否添加两个key：默认为`file`和`basename`；
* `trackerDir`：什么用？是处理file所需的metadata？当前查看，在处理每个文件过程中都会产生metadata，这个metadata上保存什么内容？用于保证：处理单个文件时，程序重启，不会产生数据重发或者丢失吗？
* 


**notes(ningg)**：上述配置中提到向Flume event添加Header的情况，那一个event到底是什么构成的呢？Header、Key、EventBody？这些的用途是什么？

**RE**：Event，`org.apache.flume.Event`实际为`Interface`，其向外提供两个属性：

* Headers：`Map<String, String>`，即，一组key-value对；
* Body：`byte[]`，即，字节数组；

**额外补充**：关于Body对应的`byte[]`，有两点要强调一下：

* 通常原始信息是`String`类型，将其转换为`byte[]`，通常需要指定`charset`，常用方法`String.getBytes(String charset)`；
* 具体如何将`byte[]`转换为人可以识别的`char[]`，其中需要设定`charset`，常用方法`new String(byte[], String charset)`；
* 如果上述过程中指定的`charset`不合适，会产生乱码；



Example for an agent named agent-1:

	agent-1.channels = ch-1
	agent-1.sources = src-1
	
	agent-1.sources.src-1.type = spooldir
	agent-1.sources.src-1.channels = ch-1
	agent-1.sources.src-1.spoolDir = /var/log/apache/flumeSpool
	agent-1.sources.src-1.fileHeader = true

##Twitter 1% firehose Source (experimental)

Warning This source is hightly experimental and may change between minor versions of Flume. Use at your own risk.

Experimental source that connects via Streaming API to the 1% sample twitter firehose, continously downloads tweets, converts them to Avro format and sends Avro events to a downstream Flume sink. Requires the consumer and access tokens and secrets of a Twitter developer account. Required properties are in bold.

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be org.apache.flume.source.twitter.TwitterSource
consumerKey	–	OAuth consumer key
consumerSecret	–	OAuth consumer secret
accessToken	–	OAuth access token
accessTokenSecret	–	OAuth toekn secret
maxBatchSize	1000	Maximum number of twitter messages to put in a single batch
maxBatchDurationMillis	1000	Maximum number of milliseconds to wait before closing a batch

Example for agent named a1:

a1.sources = r1
a1.channels = c1
a1.sources.r1.type = org.apache.flume.source.twitter.TwitterSource
a1.sources.r1.channels = c1
a1.sources.r1.consumerKey = YOUR_TWITTER_CONSUMER_KEY
a1.sources.r1.consumerSecret = YOUR_TWITTER_CONSUMER_SECRET
a1.sources.r1.accessToken = YOUR_TWITTER_ACCESS_TOKEN
a1.sources.r1.accessTokenSecret = YOUR_TWITTER_ACCESS_TOKEN_SECRET
a1.sources.r1.maxBatchSize = 10
a1.sources.r1.maxBatchDurationMillis = 200

###Event Deserializers

The following event deserializers ship with Flume.

####LINE

This deserializer generates one event per line of text input.

Property Name	Default	Description
deserializer.maxLineLength	2048	Maximum number of characters to include in a single event. If a line exceeds this length, it is truncated, and the remaining characters on the line will appear in a subsequent event.
deserializer.outputCharset	UTF-8	Charset to use for encoding events put into the channel.

####AVRO

This deserializer is able to read an Avro container file, and it generates one event per Avro record in the file. Each event is annotated with a header that indicates the schema used. The body of the event is the binary Avro record data, not including the schema or the rest of the container file elements.

Note that if the spool directory source must retry putting one of these events onto a channel (for example, because the channel is full), then it will reset and retry from the most recent Avro container file sync point. To reduce potential event duplication in such a failure scenario, write sync markers more frequently in your Avro input files.

Property Name	Default	Description
deserializer.schemaType	HASH	How the schema is represented. By default, or when the value HASH is specified, the Avro schema is hashed and the hash is stored in every event in the event header “flume.avro.schema.hash”. If LITERAL is specified, the JSON-encoded schema itself is stored in every event in the event header “flume.avro.schema.literal”. Using LITERAL mode is relatively inefficient compared to HASH mode.

####BlobDeserializer

This deserializer reads a Binary Large Object (BLOB) per event, typically one BLOB per file. For example a PDF or JPG file. Note that this approach is not suitable for very large objects because the entire BLOB is buffered in RAM.

Property Name	Default	Description
deserializer	–	The FQCN of this class: org.apache.flume.sink.solr.morphline.BlobDeserializer$Builder
deserializer.maxBlobLength	100000000	The maximum number of bytes to read and buffer for a given request



##NetCat Source

A netcat-like source that listens on a given port and turns each line of text into an event. Acts like `nc -k -l [host] [port]`. In other words, it opens a specified port and listens for data. The expectation is that the supplied data is newline separated text. Each line of text is turned into a Flume event and sent via the connected channel.
（NetCat Source监听`[host] [port]`，并将数据按行构造为Flume event）

Required properties are in **bold**.

|Property Name|	Default|	Description|
|--|--|--|
|**channels**|	– |  |
|**type**|	–|	The component type name, needs to be `netcat`|
|**bind**|	–|	Host name or IP address to bind to|
|**port**|	–|	Port # to bind to|
|max-line-length|	512	|Max line length per event body (in bytes)|
|ack-every-event|	true	|Respond with an “OK” for every event received|
|selector.type|	replicating|	replicating or multiplexing|
|selector.*|	 	|Depends on the selector.type value|
|interceptors|	–|	Space-separated list of interceptors|
|interceptors.*| |	 	 |

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = netcat
	a1.sources.r1.bind = 0.0.0.0
	a1.sources.r1.port = 6666
	a1.sources.r1.channels = c1

##Sequence Generator Source


A simple sequence generator that continuously generates events with a counter that starts from 0 and increments by 1. Useful mainly for testing. Required properties are in bold.
（持续不断的generate event，从0开始、每次加1，主要用于testing）

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be seq
selector.type	 	replicating or multiplexing
selector.*	replicating	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors
interceptors.*	 	 
batchSize	1	 

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = seq
	a1.sources.r1.channels = c1


##Syslog Sources

Reads syslog data and generate Flume events. The UDP source treats an entire message as a single event. The TCP sources create a new event for each string of characters separated by a newline (‘n’).
（读取`syslog`中数据，并构造Flume event；UDP source，将整个message构造成一个event；TCP source将收到的每行数据都单独构造一个event）

Required properties are in bold.

###Syslog TCP Source

The original, tried-and-true syslog TCP source.
（原始的、可靠的syslog TCP source）

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be syslogtcp
host	–	Host name or IP address to bind to
port	–	Port # to bind to
eventSize	2500	Maximum size of a single event line, in bytes
keepFields	false	Setting this to true will preserve the Priority, Timestamp and Hostname in the body of the event.
selector.type	 	replicating or multiplexing
selector.*	replicating	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors
interceptors.*	
 	 
For example, a syslog TCP source for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = syslogtcp
	a1.sources.r1.port = 5140
	a1.sources.r1.host = localhost
	a1.sources.r1.channels = c1


###Multiport Syslog TCP Source

This is a newer, faster, multi-port capable version of the Syslog TCP source. Note that the ports configuration setting has replaced port. Multi-port capability means that it can listen on many ports at once in an efficient manner. This source uses the Apache Mina library to do that. Provides support for RFC-3164 and many common RFC-5424 formatted messages. Also provides the capability to configure the character set used on a per-port basis.
（监听多端口的数据，效率很高；这是通过Apache Mina lib来实现的，当前支持RFC-3164和RFC-5424中规定的message格式）

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be multiport_syslogtcp
host	–	Host name or IP address to bind to.
ports	–	Space-separated list (one or more) of ports to bind to.
eventSize	2500	Maximum size of a single event line, in bytes.
keepFields	false	Setting this to true will preserve the Priority, Timestamp and Hostname in the body of the event.
portHeader	–	If specified, the port number will be stored in the header of each event using the header name specified here. This allows for interceptors and channel selectors to customize routing logic based on the incoming port.
charset.default	UTF-8	Default character set used while parsing syslog events into strings.
charset.port.<port>	–	Character set is configurable on a per-port basis.
batchSize	100	Maximum number of events to attempt to process per request loop. Using the default is usually fine.
readBufferSize	1024	Size of the internal Mina read buffer. Provided for performance tuning. Using the default is usually fine.
numProcessors	(auto-detected)	Number of processors available on the system for use while processing messages. Default is to auto-detect # of CPUs using the Java Runtime API. Mina will spawn 2 request-processing threads per detected CPU, which is often reasonable.
selector.type	replicating	replicating, multiplexing, or custom
selector.*	–	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors.
interceptors.*	 
	 
For example, a multiport syslog TCP source for agent named a1:


	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = multiport_syslogtcp
	a1.sources.r1.channels = c1
	a1.sources.r1.host = 0.0.0.0
	a1.sources.r1.ports = 10001 10002 10003
	a1.sources.r1.portHeader = port


###Syslog UDP Source

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be syslogudp
host	–	Host name or IP address to bind to
port	–	Port # to bind to
keepFields	false	Setting this to true will preserve the Priority, Timestamp and Hostname in the body of the event.
selector.type	 	replicating or multiplexing
selector.*	replicating	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors
interceptors.*	
 	 
For example, a syslog UDP source for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = syslogudp
	a1.sources.r1.port = 5140
	a1.sources.r1.host = localhost
	a1.sources.r1.channels = c1


##HTTP Source

A source which accepts Flume Events by HTTP POST and GET. GET should be used for experimentation only. HTTP requests are converted into flume events by a pluggable “handler” which must implement the HTTPSourceHandler interface. This handler takes a HttpServletRequest and returns a list of flume events. All events handled from one Http request are committed to the channel in one transaction, thus allowing for increased efficiency on channels like the file channel. If the handler throws an exception, this source will return a HTTP status of 400. If the channel is full, or the source is unable to append events to the channel, the source will return a HTTP 503 - Temporarily unavailable status.
（`GET`方式当前只用于experiment；利用`handler`将request转换为Flume events，同一个HTTP请求对应的event构成一个transaction）

All events sent in one post request are considered to be one batch and inserted into the channel in one transaction.
（单个request中的所有events都被最为one batch，并且以one transaction的形式插入到channel中）

Property Name	Default	Description
type	 	The component type name, needs to be http
port	–	The port the source should bind to.
bind	0.0.0.0	The hostname or IP address to listen on
handler	org.apache.flume.source.http.JSONHandler	The FQCN of the handler class.
handler.*	–	Config parameters for the handler
selector.type	replicating	replicating or multiplexing
selector.*	 	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors
interceptors.*	 	 
enableSSL	false	Set the property true, to enable SSL
keystore	 	Location of the keystore includng keystore file name
keystorePassword Keystore password

For example, a http source for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = http
	a1.sources.r1.port = 5140
	a1.sources.r1.channels = c1
	a1.sources.r1.handler = org.example.rest.RestHandler
	a1.sources.r1.handler.nickname = random props


###JSONHandler

A handler is provided out of the box which can handle events represented in JSON format, and supports UTF-8, UTF-16 and UTF-32 character sets. The handler accepts an array of events (even if there is only one event, the event has to be sent in an array) and converts them to a Flume event based on the encoding specified in the request. If no encoding is specified, UTF-8 is assumed. The JSON handler supports UTF-8, UTF-16 and UTF-32. Events are represented as follows.
（处理JSON字符串，并生成Flume event，要求JSON字符串是`UTF-8`、`UTF-16`、`UTF-32`编码格式）

	[{
	  "headers" : {
				 "timestamp" : "434324343",
				 "host" : "random_host.example.com"
				 },
	  "body" : "random_body"
	  },
	  {
	  "headers" : {
				 "namenode" : "namenode.example.com",
				 "datanode" : "random_datanode.example.com"
				 },
	  "body" : "really_random_body"
	  }]

To set the charset, the request must have content type specified as `application/json; charset=UTF-8` (replace `UTF-8` with `UTF-16` or `UTF-32` as required).
（request通过`content type`来设置charset，默认是`UTF-8`编码）

One way to create an event in the format expected by this handler is to use `JSONEvent` provided in the Flume SDK and use Google Gson to create the JSON string using the `Gson#fromJson(Object, Type)` method. The type token to pass as the 2nd argument of this method for list of events can be created by:
（利用JSON字符串来创建event时，通常要用到Flume SDK提供的`JSONEvent`以及`Google Gson#fromJson`）

	Type type = new TypeToken<List<JSONEvent>>() {}.getType();


###BlobHandler

By default HTTPSource splits JSON input into Flume events. As an alternative, BlobHandler is a handler for HTTPSource that returns an event that contains the request parameters as well as the Binary Large Object (BLOB) uploaded with this request. For example a PDF or JPG file. Note that this approach is not suitable for very large objects because it buffers up the entire BLOB in RAM.
（默认情况下，HTTPSource将JSON字符串分割为多个Flume event；BlobHandler能够保证将所有的request parameters甚至是BLOB的文件，不进行分割，构造为一个event；特别注意：BlobHandler不适合大文件，因为会占用过多的RAM）

Property Name	Default	Description
handler	–	The FQCN of this class: org.apache.flume.sink.solr.morphline.BlobHandler
handler.maxBlobLength	100000000	The maximum number of bytes to read and buffer for a given request

##Legacy Sources

**notes(ningg)**：Legacy Sources，目标，将`Flume 0.9.x`输出的event format，转换为`Flume 1.x`能够接收和处理的event format。

The legacy sources allow a Flume 1.x agent to receive events from Flume 0.9.4 agents. It accepts events in the Flume 0.9.4 format, converts them to the Flume 1.0 format, and stores them in the connected channel. The 0.9.4 event properties like timestamp, pri, host, nanos, etc get converted to 1.x event header attributes. The legacy source supports both Avro and Thrift RPC connections. To use this bridge between two Flume versions, you need to start a Flume 1.x agent with the avroLegacy or thriftLegacy source. The 0.9.4 agent should have the agent Sink pointing to the host/port of the 1.x agent.

Note The reliability semantics of Flume 1.x are different from that of Flume 0.9.x. The E2E or DFO mode of a Flume 0.9.x agent will not be supported by the legacy source. The only supported 0.9.x mode is the best effort, though the reliability setting of the 1.x flow will be applicable to the events once they are saved into the Flume 1.x channel by the legacy source.
Required properties are in bold.

###Avro Legacy Source

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be org.apache.flume.source.avroLegacy.AvroLegacySource
host	–	The hostname or IP address to bind to
port	–	The port # to listen on
selector.type	 	replicating or multiplexing
selector.*	replicating	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors
interceptors.*	 	 

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = org.apache.flume.source.avroLegacy.AvroLegacySource
	a1.sources.r1.host = 0.0.0.0
	a1.sources.r1.bind = 6666
	a1.sources.r1.channels = c1

###Thrift Legacy Source

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be org.apache.flume.source.thriftLegacy.ThriftLegacySource
host	–	The hostname or IP address to bind to
port	–	The port # to listen on
selector.type	 	replicating or multiplexing
selector.*	replicating	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors
interceptors.*	 	 

Example for agent named a1:

a1.sources = r1
a1.channels = c1
a1.sources.r1.type = org.apache.flume.source.thriftLegacy.ThriftLegacySource
a1.sources.r1.host = 0.0.0.0
a1.sources.r1.bind = 6666
a1.sources.r1.channels = c1

##Custom Source

A custom source is your own implementation of the Source interface. A custom source’s class and its dependencies must be included in the agent’s classpath when starting the Flume agent. The type of the custom source is its FQCN.
（通过实现Source接口，可以自定义一个Source；要求：源文件和依赖的jar包需要添加到`classpath`）

**notes(ningg)**：FQCN？什么含义？

Property Name	Default	Description
channels	–	 
type	–	The component type name, needs to be your FQCN
selector.type	 	replicating or multiplexing
selector.*	replicating	Depends on the selector.type value
interceptors	–	Space-separated list of interceptors
interceptors.*	 	 

Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = org.example.MySource
	a1.sources.r1.channels = c1

##Scribe Source

Scribe is another type of ingest system. To adopt existing Scribe ingest system, Flume should use ScribeSource based on Thrift with compatible transfering protocol. For deployment of Scribe please follow the guide from Facebook. Required properties are in bold.

**notes(ningg)**：scribe？什么含义？

Property Name	Default	Description
type	–	The component type name, needs to be `org.apache.flume.source.scribe.ScribeSource`
port	1499	Port that Scribe should be connected
workerThreads	5	Handing threads number in Thrift
selector.type	 	 
selector.*


Example for agent named a1:

	a1.sources = r1
	a1.channels = c1
	a1.sources.r1.type = org.apache.flume.source.scribe.ScribeSource
	a1.sources.r1.port = 1463
	a1.sources.r1.workerThreads = 5
	a1.sources.r1.channels = c1






##参考来源

* [Flume User Guide 1.5.0.1](http://flume.apache.org/FlumeUserGuide.html)







[NingG]:    http://ningg.github.com  "NingG"