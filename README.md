# Spark running into Spark Standalone cluster with History Server

Apache Spark is an open-source, distributed processing system used for big data workloads.

In this demo, a Spark container uses a Spark Standalone cluster as a resource management and job scheduling technology to perform distributed data processing.

This Docker image contains Spark binaries prebuilt and uploaded in Docker Hub.


## Start Swarm cluster

1. start swarm mode in node1
```shell
$ docker swarm init --advertise-addr <IP node1>
$ docker swarm join-token worker  # issue a token to add a node as worker to swarm
```

2. add 3 more workers in swarm cluster (node2, node3, node4)
```shell
$ docker swarm join --token <token> <IP nodeN>:2377
```

3. label each node to anchor each container in swarm cluster
```shell
docker node update --label-add hostlabel=hdpmst node1
docker node update --label-add hostlabel=hdp1 node2
docker node update --label-add hostlabel=hdp2 node3
docker node update --label-add hostlabel=hdp3 node4
```

4. create an external "overlay" network in swarm to link the 2 stacks (hdp and spk)
```shell
docker network create --driver overlay mynet
```

5. start a spark standalone cluster
```shell
$ docker stack deploy -c docker-compose.yml spk
$ docker service ls
ID             NAME          MODE         REPLICAS   IMAGE                             PORTS
9sjce1afjaq2   spk_hdpmst    replicated   1/1        mkenjis/ubhdp_img:latest          
d8zste0gyx96   spk_spk1      replicated   1/1        mkenjis/ubspkcluster_img:latest   
ekptsevz8x7g   spk_spk2      replicated   1/1        mkenjis/ubspkcluster_img:latest   
riqot03ma5h1   spk_spk_mst   replicated   1/1        mkenjis/ubspkcluster_img:latest   *:4040->4040/tcp, *:8080->8080/tcp, *:18080->18080/tcp```

6. in hadoop master node, create a spark-logs HDFS directory
```shell
$ docker container exec -it <spk_hdpmst ID> bash
$ hdfs dfs -mkdir /spark-logs
```

7. in spark master node, copy core-site.xml and hdfs-site.xml from hadoop master into $SPARK_HOME/conf
```shell
$ scp root@hdpmst:/usr/local/hadoop-2.7.3/etc/hadoop/core-site.xml .
Warning: Permanently added 'hdpmst,10.0.1.11' (ECDSA) to the list of known hosts.
core-site.xml                                                              100%  137    78.1KB/s   00:00    
$ scp root@hdpmst:/usr/local/hadoop-2.7.3/etc/hadoop/hdfs-site.xml .
hdfs-site.xml                                                              100%  310   292.6KB/s   00:00
```

8. copy *.xml files to spk1 and spk2
```shell
$ scp *.xml root@spk1:/usr/local/spark-2.3.2-bin-hadoop2.7/conf
core-site.xml                                                              100%  137    93.1KB/s   00:00    
hdfs-site.xml                                                              100%  310   174.1KB/s   00:00    
$ scp *.xml root@spk2:/usr/local/spark-2.3.2-bin-hadoop2.7/conf
core-site.xml                                                              100%  137   135.3KB/s   00:00    
hdfs-site.xml                                                              100%  310   275.8KB/s   00:00                                                              100%  285    59.8KB/s   00:00
```

9. edit spark-defaults.conf and add following lines
```shell
$ vi spark-defaults.conf

spark.eventLog.enabled true
spark.eventLog.dir  hdfs://hdpmst:9000/spark-logs
spark.history.fs.logDirectory  hdfs://hdpmst:9000/spark-logs

```

10. start spark history server and access port 18080 as shown
```shell
$ start-history-server.sh
starting org.apache.spark.deploy.history.HistoryServer, logging to /usr/local/spark-2.3.2-bin-hadoop2.7/logs/spark--org.apache.spark.deploy.history.HistoryServer-1-939f7f90538d.out
$ tail -f /usr/local/spark-2.3.2-bin-hadoop2.7/logs/spark--org.apache.spark.deploy.history.HistoryServer-1-939f7f90538d.out
2023-09-24 15:45:59 INFO  Server:351 - jetty-9.3.z-SNAPSHOT, build timestamp: unknown, git hash: unknown
2023-09-24 15:45:59 INFO  Server:419 - Started @4138ms
2023-09-24 15:45:59 INFO  AbstractConnector:278 - Started ServerConnector@3c1e23ff{HTTP/1.1,[http/1.1]}{0.0.0.0:18080}
2023-09-24 15:45:59 INFO  Utils:54 - Successfully started service on port 18080.
2023-09-24 15:45:59 INFO  ContextHandler:781 - Started o.s.j.s.ServletContextHandler@654b72c0{/,null,AVAILABLE,@Spark}
2023-09-24 15:45:59 INFO  ContextHandler:781 - Started o.s.j.s.ServletContextHandler@5aa6202e{/json,null,AVAILABLE,@Spark}
2023-09-24 15:45:59 INFO  ContextHandler:781 - Started o.s.j.s.ServletContextHandler@3af9aa66{/api,null,AVAILABLE,@Spark}
2023-09-24 15:45:59 INFO  ContextHandler:781 - Started o.s.j.s.ServletContextHandler@6826c41e{/static,null,AVAILABLE,@Spark}
2023-09-24 15:45:59 INFO  ContextHandler:781 - Started o.s.j.s.ServletContextHandler@72889280{/history,null,AVAILABLE,@Spark}
2023-09-24 15:45:59 INFO  HistoryServer:54 - Bound HistoryServer to 0.0.0.0, and started at http://939f7f90538d:18080
```

11. start spark-shell in spark cluster mode
```shell
$ spark-shell --master spark://<spk_hostname>:7077
2021-12-13 15:09:50 WARN  NativeCodeLoader:62 - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://464730a41833:4040
Spark context available as 'sc' (master = spark://464730a41833:7077, app id = app-20211213150958-0000).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.2
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181)
Type in expressions to have them evaluated.
Type :help for more information.

scala> 
```
