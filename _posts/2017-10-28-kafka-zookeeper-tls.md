---
title:  "Using TLS Between Kafka and Zookeeper"
excerpt: "Step by step guide to build Kafka with Zookeeper 3.5 and use TLS between Kafka and Zookeeper"
categories: 
    - Kafka
tags:
    - kafka
    - zookeeper
    - security
---

## Changes in Zookeeper

### Clone zookeeper repository and switch to 3.5 branch

```shell
$ cd /home/lafranll/prjs/
$ git clone git://git.apache.org/zookeeper.git
$ cd zookeeper
$ git checkout -b branch-3.5 origin/branch-3.5
```

### Configure zookeeper to use SSL

```shell
$ cp conf/zoo_sample.cfg conf/zoo.cfg
$ diff -c conf/zoo_sample.cfg conf/zoo.cfg 
*** conf/zoo_sample.cfg	2016-07-06 13:02:20.699372001 -0300
--- conf/zoo.cfg	2016-07-06 14:23:07.793708000 -0300
***************
*** 26,28 ****
--- 26,30 ----
  # Purge task interval in hours
  # Set to "0" to disable auto purge feature
  #autopurge.purgeInterval=1
+ 
+ secureClientPort=2281

$ git diff
diff --git a/bin/zkEnv.sh b/bin/zkEnv.sh
index 9806a4b..1ef35fb 100755
--- a/bin/zkEnv.sh
+++ b/bin/zkEnv.sh
@@ -138,3 +138,18 @@ export SERVER_JVMFLAGS="-Xmx${ZK_SERVER_HEAP}m $SERVER_JVMFLAGS"
 # default heap for zookeeper client
 ZK_CLIENT_HEAP="${ZK_CLIENT_HEAP:-256}"
 export CLIENT_JVMFLAGS="-Xmx${ZK_CLIENT_HEAP}m $CLIENT_JVMFLAGS"
+
+# SSL server properties
+export SERVER_JVMFLAGS="-Dzookeeper.serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory \
+-Dzookeeper.ssl.keyStore.location=/home/lafranll/prjs/zookeeper/src/java/test/data/ssl/testKeyStore.jks \
+-Dzookeeper.ssl.keyStore.password=testpass \
+-Dzookeeper.ssl.trustStore.location=/home/lafranll/prjs/zookeeper/src/java/test/data/ssl/testTrustStore.jks \
+-Dzookeeper.ssl.trustStore.password=testpass $SERVER_JVMFLAGS"
+
+# SSL client properties
+export CLIENT_JVMFLAGS="-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty \
+-Dzookeeper.client.secure=true \
+-Dzookeeper.ssl.keyStore.location=/home/lafranll/prjs/zookeeper/src/java/test/data/ssl/testKeyStore.jks \
+-Dzookeeper.ssl.keyStore.password=testpass \
+-Dzookeeper.ssl.trustStore.location=/home/lafranll/prjs/zookeeper/src/java/test/data/ssl/testTrustStore.jks \
+-Dzookeeper.ssl.trustStore.password=testpass $CLIENT_JVMFLAGS"
```

### Build zookeeper and optionally build zookeeper tar.gz

```shell
$ ant build
$ ls build/zookeeper-3.5.2-alpha.jar 
build/zookeeper-3.5.2-alpha.jar

$ sudo apt-get install autoconf libtool
$ ant tar
$ ls build/zookeeper-3.5.2-alpha.tar.gz 
build/zookeeper-3.5.2-alpha.tar.gz
```

## Changes in zkClient

### Build zkClient using zookeeper-3.5.2-alpha.jar

Clone zkClient repository and copy zookeeper-3.5.2-alpha.jar to zkClient lib directory.
Note: Instead of copying zookeeper jar into zkclient lib dir,
another option is to add a new flat dir repository pointing to zookeeper build directory.

```shell
$ cd /home/lafranll/prjs/
$ git clone https://github.com/sgroschupf/zkclient
$ cp zookeeper/build/zookeeper-3.5.2-alpha.jar zkclient/lib/
$ cd zkclient
$ pwd
/home/lafranll/prjs/zkclient
$ git branch
* master
```

### Edit build.gradle to use zookeeper-3.5.2-alpha.jar and build zkClient

```shell
$ git diff build.gradle
diff --git a/build.gradle b/build.gradle
index 8c7eff5..49bce40 100644
--- a/build.gradle
+++ b/build.gradle
@@ -33,7 +33,7 @@ dependencies {
         exclude group: "com.sun.jmx", module: "jmxri"
         exclude group: "javax.jms", module: "jms"
     }
-    compile 'org.apache.zookeeper:zookeeper:3.4.8'
+    compile 'org.apache.zookeeper:zookeeper:3.5.2-alpha'
     compile 'org.slf4j:slf4j-api:1.6.1'
     compile 'org.slf4j:slf4j-log4j12:1.6.1'

$ gradle jars
$ ls build/libs/zkclient-0.10-dev.jar 
build/libs/zkclient-0.10-dev.jar
```

## Changes in Kafka

### Clone Kafka repository and switch to 0.9 branch

```shell
$ cd /home/lafranll/prjs/
$ git clone https://github.com/apache/kafka
$ cd kafka
$ git checkout -b 0.9.0 origin/0.9.0
```

### Edit build.gradle to use zookeeper-3.5.2-alpha.jar and zkclient-0.10-dev.jar.

Also include netty that is required by zookeeper client.

```shell
$ git diff build.gradle
diff --git a/build.gradle b/build.gradle
index 19ac665..0a3d894 100644
--- a/build.gradle
+++ b/build.gradle
@@ -256,12 +256,22 @@ project(':core') {
   apply plugin: 'scala'
   archivesBaseName = "kafka_${baseScalaVersion}"
 
+  repositories {
+    flatDir {
+      dirs '/home/lafranll/prjs/zkclient/build/libs'
+    }
+    flatDir {
+      dirs '/home/lafranll/prjs/zookeeper/build'
+    }
+  }
+
   dependencies {
     compile project(':clients')
     compile "$slf4jlog4j"
     compile "org.scala-lang:scala-library:$scalaVersion"
-    compile 'org.apache.zookeeper:zookeeper:3.4.6'
-    compile 'com.101tec:zkclient:0.7'
+    compile 'org.apache.zookeeper:zookeeper:3.5.2-alpha'
+    compile 'io.netty:netty:3.10.6.Final'
+    compile 'com.101tec:zkclient:0.10-dev'
     compile 'com.yammer.metrics:metrics-core:2.2.0'
     compile 'net.sf.jopt-simple:jopt-simple:3.2'
     if (scalaVersion.startsWith('2.11')) {
@@ -289,7 +299,6 @@ project(':core') {
     compile.exclude module: 'jmxri'
     compile.exclude module: 'jmxtools'
     compile.exclude module: 'mail'
-    compile.exclude module: 'netty'
     // To prevent a UniqueResourceException due the same resource existing in both
     // org.apache.directory.api/api-all and org.apache.directory.api/api-ldap-schema-data
     testCompile.exclude module: 'api-ldap-schema-data'
```

### Add ZK JVM flags to use Netty and SSL to kafka-server-start.sh

Make Kafka connect to zookeeper secure port. 

```shell
$ git diff bin/kafka-server-start.sh config/server.properties
diff --git a/bin/kafka-server-start.sh b/bin/kafka-server-start.sh
index 36c2a0d..7b998e0 100755
--- a/bin/kafka-server-start.sh
+++ b/bin/kafka-server-start.sh
@@ -31,6 +31,14 @@ fi
 
 EXTRA_ARGS="-name kafkaServer -loggc"
 
+ZK_CLIENT_JVMFLAGS="-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty \
+-Dzookeeper.client.secure=true \
+-Dzookeeper.ssl.keyStore.location=/home/lafranll/prjs/zookeeper/src/java/test/data/ssl/testKeyStore.jks \
+-Dzookeeper.ssl.keyStore.password=testpass \
+-Dzookeeper.ssl.trustStore.location=/home/lafranll/prjs/zookeeper/src/java/test/data/ssl/testTrustStore.jks \
+-Dzookeeper.ssl.trustStore.password=testpass"
+EXTRA_ARGS="$EXTRA_ARGS $ZK_CLIENT_JVMFLAGS"
+
 COMMAND=$1
 case $COMMAND in
   -daemon)
diff --git a/config/server.properties b/config/server.properties
index ddb695a..92b19c5 100644
--- a/config/server.properties
+++ b/config/server.properties
@@ -113,7 +113,7 @@ log.retention.check.interval.ms=300000
 # server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
 # You can also append an optional chroot string to the urls to specify the
 # root directory for all kafka znodes.
-zookeeper.connect=localhost:2181
+zookeeper.connect=localhost:2281
 
 # Timeout in ms for connecting to zookeeper
 zookeeper.connection.timeout.ms=6000
```

### Build Kafka release tar.gz

```shell
$ gradle -PscalaVersion=2.11 clean releaseTarGz
$ ls ./core/build/distributions/kafka_2.11-0.9.0.2-SNAPSHOT.tgz 
./core/build/distributions/kafka_2.11-0.9.0.2-SNAPSHOT.tgz
```
