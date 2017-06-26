# hadoop on macosX

## protobuf

protobuf-2.5.0.tar.gz，hadoop2.7.3版本要求必须为2.5.0。

https://github.com/google/protobuf

https://github.com/google/googletest

brew  install autoconf automake libtool zlib openssl 


## apache hadoop-2.7.3

hadoop-2.7.3-src/hadoop-hdfs-project/hadoop-hdfs-httpfs/pom.xml

http://archive.apache.org/dist/tomcat/tomcat-6/v6.0.44/bin/apache-tomcat-6.0.44.tar.gz

拷贝到hadoop-2.7.3-src/hadoop-hdfs-project/hadoop-hdfs-httpfs/download目录。

export  OPENSSL_ROOT_DIR=/usr/local/Cellar/openssl/1.0.2l

mvn clean package -Pdist,native -DskipTests -Dtar

mvn package -Pdist,native -DskipTests -Dtar



## spark

http://spark.apache.org/docs/latest/building-spark.html

brew install sbt

brew install zinc

Maven is the official build tool recommended for packaging Spark, and is the build of reference. But SBT is supported for day-to-day development since it can provide much faster iterative compilation. More advanced developers may wish to use SBT.

The SBT build is derived from the Maven POM files, and so the same Maven profiles and variables can be set to control the SBT build. For example:

`./build/sbt -Pyarn -Phadoop-2.3 package`

`./build/mvn -Pyarn -Phadoop-2.7 -Dhadoop.version=2.7.0 -Phive -Phive-thriftserver  -DskipTests clean package`

如果没有安装zinc和scala，则自动下载安装。
```
curl --progress-bar -L https://downloads.typesafe.com/zinc/0.3.9/zinc-0.3.9.tgz
curl --progress-bar -L https://downloads.typesafe.com/scala/2.11.8/scala-2.11.8.tgz
```