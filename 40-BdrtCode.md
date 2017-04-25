# BDRT 源代码结构分析

## src/main/core/java
源自[titan](https://github.com/thinkaurelius/titan)
### com.sky.bdrt.core
[com.thinkaurelius.titan.core](https://github.com/thinkaurelius/titan/blob/titan10/titan-core/src/main/java/com/thinkaurelius/titan/core/TitanTransaction.java)
其中BdrtTransaction源自[TitanTransaction](https://github.com/thinkaurelius/titan/blob/titan10/titan-core/src/main/java/com/thinkaurelius/titan/core/TitanTransaction.java
)
### com.sky.bdrt.diskstorage
[com.thinkaurelius.titan.diskstorage](https://github.com/thinkaurelius/titan/tree/titan10/titan-core/src/main/java/com/thinkaurelius/titan/diskstorage)
### com.sky.bdrt.skytable
[com.thinkaurelius.titan.graphdb](https://github.com/thinkaurelius/titan/tree/titan10/titan-core/src/main/java/com/thinkaurelius/titan/graphdb)

vertices变更为records。
### com.sky.bdrt.util
[com.thinkaurelius.util](https://github.com/thinkaurelius/titan/tree/titan10/titan-core/src/main/java/com/thinkaurelius/titan/util)

## src/main/api/java
https://github.com/apache/tinkerpop
### com.sky.bdrt.api

源自[org.apache.tinkerpop.gremlin](https://github.com/apache/tinkerpop/tree/master/gremlin-core/src/main/java/org/apache/tinkerpop/gremlin)

其中com.sky.bdrt.api.process.computer 中的 DatabaseFilter源自org.apache.tinkerpop.gremlin.process.computer 的 GraphFilter。

## src/main/api-driver/java
### com.sky.bdrt.api.driver
源自[org.apache.tinkerpop.gremlin.driver](https://github.com/apache/tinkerpop/tree/master/gremlin-driver/src/main/java/org/apache/tinkerpop/gremlin/driver)

## src/main/core-server/java
### com.sky.bdrt.api.server
源自[org.apache.tinkerpop.gremlin.server](https://github.com/apache/tinkerpop/tree/master/gremlin-server/src/main/java/org/apache/tinkerpop/gremlin/server)

## src/main/console/java

### com.sky.bdrt.api.console.jsr223
源自[org.apache.tinkerpop.gremlin.console.jsr223](https://github.com/apache/tinkerpop/tree/master/gremlin-console/src/main/java/org/apache/tinkerpop/gremlin/console/jsr223)

## src/main/console/groovy
### com.sky.bdrt.api.console
源自[org.apache.tinkerpop.gremlin.console](https://github.com/apache/tinkerpop/tree/master/gremlin-console/src/main/groovy/org/apache/tinkerpop/gremlin/console)

包含Console.groovy、GremlinGroovysh.groovy等文件和commands、jsr223两个目录。
增加了plugin和groovy.plugin目录和Sql*等文件。

## src/main/groovy/groovy
### com.sky.bdrt.api.groovy
源自[org.apache.tinkerpop.gremlin.groovy](https://github.com/apache/tinkerpop/tree/master/gremlin-groovy/src/main/groovy/org/apache/tinkerpop/gremlin/groovy)
包含jsr223、loaders、util三个目录。

## src/main/groovy/java
### com.sky.bdrt.api.groovy
源自[org.apache.tinkerpop.gremlin.groovy](https://github.com/apache/tinkerpop/tree/master/gremlin-groovy/src/main/java/org/apache/tinkerpop/gremlin/groovy)
增加了function、plugin两个目录和*CustomizerProvider等文件。

## src/main/core-sql/java
### com.sky.bdrt.api.sql
源自[org.twilmes.sql.gremlin](https://github.com/twilmes/sql-gremlin/tree/master/src/main/java/org/twilmes/sql/gremlin)

### com.sky.bdrt.api.jdbc
http://calcite.apache.org/avatica/docs/index.html
https://github.com/apache/calcite-avatica
org.apache.calcite.jdbc.Driver
SQL Parser部分采用的是Apache Calcite。简单的来说Calcite实现的功能是提供了JDBC interface，接收用户的查询请求，然后将SQL Query语句转换成为SQL语法树，也就是逻辑计划。

## insert table 处理流程
http://192.168.100.2:81/bdrt/beagledata-BDRT/commit/6d7986a02a0e335b71f564214c8246af24f6e56e

 plugins/api-sql/src/java/com/sky/bdrt/api/sql/plugin/SqlRemoteAcceptor.java
 src/main/core-server/java/com/sky/bdrt/api/server/op/sql/SqlOpProcessor.java
 （import com.alibaba.druid.sql.ast.statement.SQLInsertStatement;处理insert语句）
 添加src/main/core-server/java/com/sky/bdrt/api/server/op/sql/visitor/InsertTable.java 处理文件
src/main/core-server/java/com/sky/bdrt/api/server/op/sql/visitor/VisitorManagerImpl.java
（visitors.add(InsertTable.class);）
 src/main/core-sql/java/com/sky/bdrt/api/sql/processor/RelUtils.java src/main/core/java/com/sky/bdrt/core/BdrtTable.java
 src/main/core/java/com/sky/bdrt/skytable/types/RecordLabelRecord.java
  src/main/core/java/com/sky/bdrt/skytable/types/system/BaseRecordLabel.java
  test/com/sky/test/SearchCompany.java




## 启动脚本

### bdrt-server.sh
启动脚本中bdrt-server.sh的`-javaagent:$LIB/jamm-0.3.0.jar`：
Compute Java Object Memory Footprint at runtime with JAMM (Java Agent for Memory Measurements)
[Titan使用过程CPU超高问题排查](http://www.tuicool.com/articles/A3iyEf3)

