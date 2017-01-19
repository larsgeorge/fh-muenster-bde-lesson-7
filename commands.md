## Kommandos

Wie immer sind die folgenden Befehle in den Hadoop VMs auszufuehren. Fuer Flume wird dazu die Cloudera Quickstart VM genutzt (also sind leichte Abwandlungen fuer die Hortonworks Sandbox VM vorzunehmen).

### Spark

Starten der Shell in der VM:

```
[cloudera@quickstart fh-muenster-bde-lesson-6]$ spark-shell
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/lib/zookeeper/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/lib/flume-ng/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 1.6.0
      /_/

Using Scala version 2.10.5 (Java HotSpot(TM) 64-Bit Server VM, Java 1.7.0_67)
Type in expressions to have them evaluated.
Type :help for more information.
17/01/19 01:49:30 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Spark context available as sc (master = local[*], app id = local-1484819373040).
17/01/19 01:49:48 WARN metastore.ObjectStore: Version information not found in metastore. hive.metastore.schema.verification is not enabled so recording the schema version 1.1.0
17/01/19 01:49:49 WARN metastore.ObjectStore: Failed to get database default, returning NoSuchObjectException
17/01/19 01:49:51 WARN shortcircuit.DomainSocketFactory: The short-circuit local reads feature cannot be used because libhadoop cannot be loaded.
SQL context available as sqlContext.
```

Den Kontext kann man nun mit der Variablen `sc` ansprechen:

```
scala> sc
res0: org.apache.spark.SparkContext = org.apache.spark.SparkContext@4bdb8813
```

Nun kann man die Scala Tests durchfuehren. 

```
scala> sc.parallelize(Array(1,2,3))
res2: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:28

...
```

Hier ein Beispiel fuer den Word Count der Buecher aus den vorherigen Uebungen:

```
scala> val books = sc.textFile("hdfs://quickstart.cloudera:8020/user/cloudera/books")
books: org.apache.spark.rdd.RDD[String] = hdfs://quickstart.cloudera:8020/user/cloudera/books MapPartitionsRDD[2] at textFile at <console>:27

scala> val counts = books.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
counts: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[5] at reduceByKey at <console>:29

scala> counts.take(10)
res3: Array[(String, Int)] = Array((celebrant,1), (mattered,6), (elm-trees.,1), (rises.,3), (bone,55), (Hats,1), (Never,,6), (showy,2), ("Valentine.",1), (Robespierre,4))
```

`flatMap` dient dazu, die Arrays pro Zeile aufzuloesen (siehe unten), denn `RDD.map` auf einer Textdatei liest jede Zeile ebenfalls als Array Element. Damit haben wir dann ein Array von Arrays. Wir wollen aber eine flache Liste haben.

```
scala> "hallo welt, guten tag".split(" ")
res4: Array[String] = Array(hallo, welt,, guten, tag)
```

Weiterhin sind auch nicht Alphabetzeichen wieder mit in den zerlegten Woertern. Es muss also noch nachgeholfen werden:

```
scala> "\"Hello, World\"".replaceAll("""\W""","")
res7: String = HelloWorld

scala> val counts = books.flatMap(line => line.split(" ")).map(word => (word.toLowerCase().replaceAll("""\W""",""), 1)).reduceByKey(_ + _)
counts: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[8] at reduceByKey at <console>:29

scala> counts.take(10)
res8: Array[(String, Int)] = Array((celebrant,1), (blavin,1), (mattered,7), (cosmopolitan,5), (bone,89), (stickyback,1), (fatmaking,1), (showy,4), (agier,1), (alavatar,1))
```

Jetzt koennen wir noch die Ergebnisse sortieren nach Auftreten der Woerter:

```
scala> val sorted = counts.sortBy(x => -x._2)
sorted: org.apache.spark.rdd.RDD[(String, Int)] = MapPartitionsRDD[29] at sortBy at <console>:31

scala> sorted.take(10)
res10: Array[(String, Int)] = Array(("",301912), (the,147448), (and,75259), (of,72477), (to,57255), (a,51123), (in,40781), (he,30436), (that,26552), (i,26489))
```

### Flume

Fuer Flume auf der Cloudera VM ist zu beachten, das dort der Agent Name als `tier1` vorgegeben ist:

```
[cloudera@quickstart conf]$ sudo grep -r "tier1" /etc
/etc/default/flume-ng-agent:FLUME_AGENT_NAME=tier1
```
Deshalb ist ein einfaches Beispiel wie folgt zu generieren. In dem Verzeichnis `/etc/flume-ng/conf/` muss die `flume.conf` editiert werden, damit sie folgenden Inhalt hat:

```
[cloudera@quickstart conf]$ cd /etc/flume-ng/conf
[cloudera@quickstart conf]$ vi flume.conf
[cloudera@quickstart conf]$ cat flume.conf
tier1.sources = seqGenSrc
tier1.channels = memoryChannel
tier1.sinks = loggerSink

tier1.sources.seqGenSrc.type = seq
tier1.sources.seqGenSrc.channels = memoryChannel

tier1.sinks.loggerSink.type = logger
tier1.sinks.loggerSink.channel = memoryChannel

tier1.channels.memoryChannel.type = memory
tier1.channels.memoryChannel.capacity = 100
```

Es wird eine _Sequence Generator_ Quelle gestart, welche bei "1" anfaengt und immer weiter zaehlt. Der Sink ist ein Test Sink welcher in der Flume Log Datei die Nachrichten verbatim ausgibt. Der Kanal ist einfach gehalten und nutzt nur den Hauptspeicher. Dann muss Flume noch gestartet werden:

```
[cloudera@quickstart conf]$ sudo service flume-ng-agent start
```

Das started alles und Flume generiert Zahlen und logged diese wie erwartet:

```
[cloudera@quickstart conf]$ tail -f /var/log/flume-ng/flume.log
...
19 Jan 2017 05:00:38,106 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{} body: 33 34 31 31 31 35                               341115 }
19 Jan 2017 05:00:38,106 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{} body: 33 34 31 31 31 36                               341116 }
19 Jan 2017 05:00:38,106 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{} body: 33 34 31 31 31 37                               341117 }
19 Jan 2017 05:00:38,107 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{} body: 33 34 31 31 31 38                               341118 }
^C
```

Am besten den Flume Agent Prozess stoppen, denn sonst laeuft die Logdatei voll.

```
[cloudera@quickstart conf]$ sudo service flume-ng-agent stop
Stopped Flume NG agent daemon:                             [  OK  ]
```

Ein zweites Beispiel nutzt den Linux Syslog Daemon. Zuerst aber muss wieder die `flume.conf` angepasst werden:

```
[cloudera@quickstart conf]$ vi flume.conf
[cloudera@quickstart conf]$ cat flume.conf
tier1.sources = syslogSrc
tier1.channels = memoryChannel
tier1.sinks = loggerSink

tier1.sources.syslogSrc.type = syslogtcp
tier1.sources.syslogSrc.host = 127.0.0.1
tier1.sources.syslogSrc.port = 5140
tier1.sources.syslogSrc.channels = memoryChannel

tier1.sinks.loggerSink.type = logger
tier1.sinks.loggerSink.channel = memoryChannel

tier1.channels.memoryChannel.type = memory
tier1.channels.memoryChannel.capacity = 100

[cloudera@quickstart conf]$ cat flume.conf 
tier1.sources = syslogSrc
tier1.channels = memoryChannel
tier1.sinks = loggerSink

tier1.sources.syslogSrc.type = syslogtcp
tier1.sources.syslogSrc.host = 127.0.0.1
tier1.sources.syslogSrc.port = 5140
tier1.sources.syslogSrc.channels = memoryChannel

tier1.sinks.loggerSink.type = logger
tier1.sinks.loggerSink.channel = memoryChannel

tier1.channels.memoryChannel.type = memory
tier1.channels.memoryChannel.capacity = 100
```

Auch der Syslogd Prozess muss angepasst werden, hier in der Cloudera VM mit CentOS (RSyslog):

```
[cloudera@quickstart conf]$ sudo vi /etc/rsyslog.conf 
[cloudera@quickstart conf]$ sudo cat /etc/rsyslog.conf 
# rsyslog v5 configuration file
...

# Diese Zeile am Ende hinzufuegen. Zwei "@" bedeutet TCP.
*.*	@@127.0.0.1:5140
```

Dann den Syslog Prozess neu starten:

```
[cloudera@quickstart conf]$ sudo service rsyslog restart
Shutting down system logger:                               [  OK  ]
Starting system logger:                                    [  OK  ]
```

Und im Anschluss Flume neu starten:

```
[cloudera@quickstart conf]$ sudo service flume-ng-agent restart
Flume agent is not running                                 [  OK  ]
Starting Flume NG agent daemon (flume-ng-agent):           [  OK  ]
```

Das `logger` Kommando erlaubt eine Nachricht an den Syslog Daemon zu schicken:

```
[cloudera@quickstart fh-muenster-bde-lesson-6]$ logger "This is a test"
```

Und das sollte dann in dem System Log auftauchen:

```
[cloudera@quickstart fh-muenster-bde-lesson-6]$ sudo tail -f /var/log/messages
...
Jan 19 05:19:10 quickstart kernel: Kernel logging (proc) stopped.
Jan 19 05:19:10 quickstart rsyslogd: [origin software="rsyslogd" swVersion="5.8.10" x-pid="1976" x-info="http://www.rsyslog.com"] exiting on signal 15.
Jan 19 05:19:10 quickstart kernel: imklog 5.8.10, log source = /proc/kmsg started.
Jan 19 05:19:10 quickstart rsyslogd: [origin software="rsyslogd" swVersion="5.8.10" x-pid="3630" x-info="http://www.rsyslog.com"] start
Jan 19 05:19:23 quickstart cloudera: This is a test
```

Aber auch im Flume Log:

```
[cloudera@quickstart conf]$ tail -f /var/log/flume-ng/flume.log
19 Jan 2017 05:10:57,410 INFO  [conf-file-poller-0] (org.apache.flume.node.AbstractConfigurationProvider.getConfiguration:114)  - Channel memoryChannel connected to [syslogSrc, loggerSink]
19 Jan 2017 05:10:57,421 INFO  [conf-file-poller-0] (org.apache.flume.node.Application.startAllComponents:138)  - Starting new configuration:{ sourceRunners:{syslogSrc=EventDrivenSourceRunner: { source:org.apache.flume.source.SyslogTcpSource{name:syslogSrc,state:IDLE} }} sinkRunners:{loggerSink=SinkRunner: { policy:org.apache.flume.sink.DefaultSinkProcessor@1fa68b9c counterGroup:{ name:null counters:{} } }} channels:{memoryChannel=org.apache.flume.channel.MemoryChannel{name: memoryChannel}} }
19 Jan 2017 05:10:57,440 INFO  [conf-file-poller-0] (org.apache.flume.node.Application.startAllComponents:145)  - Starting Channel memoryChannel
19 Jan 2017 05:10:57,597 INFO  [lifecycleSupervisor-1-0] (org.apache.flume.instrumentation.MonitoredCounterGroup.register:120)  - Monitored counter group for type: CHANNEL, name: memoryChannel: Successfully registered new MBean.
19 Jan 2017 05:10:57,598 INFO  [lifecycleSupervisor-1-0] (org.apache.flume.instrumentation.MonitoredCounterGroup.start:96)  - Component type: CHANNEL, name: memoryChannel started
19 Jan 2017 05:10:57,602 INFO  [conf-file-poller-0] (org.apache.flume.node.Application.startAllComponents:173)  - Starting Sink loggerSink
19 Jan 2017 05:10:57,604 INFO  [conf-file-poller-0] (org.apache.flume.node.Application.startAllComponents:184)  - Starting Source syslogSrc
19 Jan 2017 05:10:57,726 INFO  [lifecycleSupervisor-1-2] (org.apache.flume.source.SyslogTcpSource.start:119)  - Syslog TCP Source starting...
19 Jan 2017 05:19:11,798 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484831950000, Severity=6, host=quickstart, Facility=0, priority=6} body: 6B 65 72 6E 65 6C 3A 20 69 6D 6B 6C 6F 67 20 35 kernel: imklog 5 }
19 Jan 2017 05:19:11,799 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484831950000, Severity=6, host=quickstart, Facility=5, priority=46} body: 72 73 79 73 6C 6F 67 64 3A 20 5B 6F 72 69 67 69 rsyslogd: [origi }

19 Jan 2017 05:19:23,021 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484831963000, Severity=5, host=quickstart, Facility=1, priority=13} body: 63 6C 6F 75 64 65 72 61 3A 20 54 68 69 73 20 69 cloudera: This i }
19 Jan 2017 05:20:01,750 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484832001000, Severity=6, host=quickstart, Facility=9, priority=78} body: 43 52 4F 4E 44 5B 33 36 36 33 5D 3A 20 28 72 6F CROND[3663]: (ro }
19 Jan 2017 05:28:48,923 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484832528000, Severity=4, host=quickstart, Facility=2, priority=20} body: 70 6F 73 74 66 69 78 2F 70 69 63 6B 75 70 5B 33 postfix/pickup[3 }
19 Jan 2017 05:28:48,923 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484832528000, Severity=4, host=quickstart, Facility=2, priority=20} body: 70 6F 73 74 66 69 78 2F 70 69 63 6B 75 70 5B 33 postfix/pickup[3 }
19 Jan 2017 05:30:01,868 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484832601000, Severity=6, host=quickstart, Facility=9, priority=78} body: 43 52 4F 4E 44 5B 33 39 39 33 5D 3A 20 28 72 6F CROND[3993]: (ro }
19 Jan 2017 05:40:01,950 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484833201000, Severity=6, host=quickstart, Facility=9, priority=78} body: 43 52 4F 4E 44 5B 34 33 31 31 5D 3A 20 28 72 6F CROND[4311]: (ro }
19 Jan 2017 05:50:01,100 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:94)  - Event: { headers:{timestamp=1484833801000, Severity=6, host=quickstart, Facility=9, priority=78} body: 43 52 4F 4E 44 5B 34 36 33 38 5D 3A 20 28 72 6F CROND[4638]: (ro }
```

Man kann hier auch gut die automatisch gesetzten Header sehen, z. B.:

```
headers:{timestamp=1484833801000, Severity=6, host=quickstart, Facility=9, priority=78
```

Diese kennen wir aus der letzten Vorlesung, wo wir die Syslog Daten mit Morphlines geparsed haben.



