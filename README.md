# Zabbix-Monitoring-Kafka
Zabbix-Monitoring Kafka集群 Brokers服务,Kafka Consumer Monitoring


#### Preface:

The company let me deal with Hadoop and related services monitoring, alarm here mainly kafka cluster services. Here I also read a few kafka related articles, good text posted out:

Infoq kafka entry understanding
Introduction to kafka working principle
Several of our main features in Kafka are very satisfying to our needs: scalability, data partitioning, low latency, and the ability to handle a large number of different consumers.

And here I want to help the BI team to achieve Kafka full control. Two points:

   1. Monitor Kafka Brokers service
   2. Monitor Kafka Lag count
   
For Kafka monitoring, there are ready-made open source software, and in our company also used for some time, there are two options. Our company uses the third option.

Kafka three monitoring tools
table of Contents

```
  1, Kafka Web Conslole
  2, Kafka Manager
  3, KafkaOffsetMonitor
```


* First you have to install zabbix-java-gataway
 
 ```bash
    yum install -y zabbix-java-gataway
  ```
  
*  Configuring zabbix-java-gataway
    mcedit /etc/zabbix/zabbix_java_gateway.conf
Uncoment and set **START_POLLERS=10**

*  Configuring zabbix-server
  
   ```bash
  mcedit /etc/zabbix/zabbix_server.conf
  ```
Uncoment and set to **StartJavaPollers=5**
Change IP for **JavaGateway=IP_address_java_gateway**


*  Restart zabbix-server

```
/etc/init.d/zabbix-java-gataway restart
```

*   Add to autorun zabbix-java-gataway
   
```   
   chkconfig --level 345 zabbix-java-gataway on
```

*  Start zabbix-java-gataway

```
/etc/init.d/zabbix-java-gataway start
```
*  Kafka configuration

```
    cd /opt/kafka/bin
    mcedit kafka-run-class.sh
```
change from

    # JMX settings
    if [ -z "$KAFKA_JMX_OPTS" ]; then
    KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -   Dcom.sun.management.jmxremote.ssl=false "
    fi

to

    # JMX settings
    if [ -z "$KAFKA_JMX_OPTS" ]; then
    KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=12345 -    Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false "
    fi

* Add Kafka as service
* Add to /etc/supervisord.conf that lines

     [program:kafka]
     command=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
     directory=/opt/kafka/
     autostart=true
     autorestart=true
     stopasgroup=true
     startsecs=10
     startretries=999
     log_stdout=true
     log_stderr=true
     logfile=/var/log/kafka/supervisord-kafka.out
     logfile_maxbytes=20MB
     logfile_backups=10

* Restart supervisor 
 
 ```
 /etc/init.d/supervisord restart
 ```
# Zabbix configuration

#Upload scripts for discovery JMX

     git clone https://github.com/yangcvo/Zabbix-Monitoring-Kafka.git
     cd /kafka
     cp jmx_discovery /etc/zabbix/externalscripts
     cp JMXDiscovery-0.0.1.jar /etc/zabbix/externalscripts

##Import template
Log in to your zabbix web

**Click Configuration->Templates->Import**

Download template [zbx_kafka_templates.xml](https://github.com/yangcvo/Zabbix-Monitoring-Kafka/blob/master/zbx_export_templates.xml) and upload to zabbix
Then add this template to Kafka and configure JMX interfaces on zabbix 

Enter Kafka IP address and JMX port
If you see jmx icon, you configured JMX monitoring  good!

# Troubles 
if you have problems you can check JMX using this script
     #!/usr/bin/env bash
     
     ZBXGET="/usr/bin/zabbix_get"
     if [ $# != 5 ]
     then
     echo "Usage: $0 <JAVA_GATEWAY_HOST> <JAVA_GATEWAY_PORT> <JMX_SERVER> <JMX_PORT> <KEY>"
     exit;
     fi
     QUERY="{\"request\": \"java gateway jmx\",\"conn\": \"$3\",\"port\": $4,\"keys\": [\"$5\"]}"
     $ZBXGET -s $1 -p $2 -k "$QUERY"

**eg.:** ./zabb_get_java  zabbix-java-gateway-ip 10052 server-test-ip 12345 
'jmx[java.lang:type=Threading,PeakThreadCount]'

For monitoring kafka consumers you should install [Burrow](https://github.com/linkedin/Burrow/) daemon and [jq](https://stedolan.github.io/jq/download/) tools on kafka host

***
