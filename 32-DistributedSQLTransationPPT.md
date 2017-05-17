
## 分布式SQL数据库
Note: https://github.com/hakimel/reveal.js
---
## 关系型数据库现状 
---
## NOSQL的兴起和优缺点回顾
--
### 优点
--
### 缺点
--
### NoSQL与SQL的对比 
---
## NoSQL数据库的分类
--
### 键值(Key-Value)存储数据库 
--
### 列存储数据库 
--
### 文档型数据库 
--
### 图形(Graph)数据库
---
## 新一代分布式SQL数据库(NewSQL)
NewSQL的必备特点 TiDB
---
## CockroachDB
---
## BDRT
--
### BDRT查询引擎具有下列特性
--
### 产品优势
--
### 适用场景
---
## 基础技术原理和名称术语
---
## CAP 
---
## ACID
### 原子性 
--
### 一致性(Consistency) 
--
### 隔离性(Isolation) 
--
### 持久性(Durability)
---
## 权衡一致性与可用性 - BASE理论
### BA - Basically Available - 基本可用
--
### S – Soft State - 软状态
--
### E - Eventually Consistent - 最终一致性
---
## 分布式存储算法和技术实现(Atomic)
---
## 分布式系统的定义
---
## 一致性的基础知识
一致性问题
一致性基本概念
---
## 选举、多数派、租约
---
## 分布式系统的时间
### 时间、时钟和时序
--
### Logic Clock (Lamport Clock) Vector Clock
--
### Hybrid Logic Clock
--
### TSO
---
## 一致性模型
分布式一致性协议与算法 Paxos 一致性算法
## 分布式协调和配置服务
## Etcd 架构与实现 
## 分区(Partitioning) 
## 分片(Replication) 
## 一致性哈希(Consistent Hashing)
---
## 开源SQL引擎介绍
### Apache Hive 
--
### Apache Impala 
--
### Spark SQL 
--
### Apache Drill 
--
### Apache HAWQ 
--
### Presto
--
### Apache Calcite 
--
### Apache Kylin 
--
### Apache Phoenix 
--
### Apache Tajo 
--
### Apache Trafodion 
--
### OceanBase
---
## 分布式数据库 Schema 设计 索引
--
### TiDB的索引
--
### Spanner 的白皮书建议避免的情况 
--
### 查询优化
---
## TiDB的查询优化
--- 
## 分布式数据库开发
TiDB和CockroachDB为什么使用GO语言实现
---
## BDRT为什么使用JAVA语言实现 
---
## 分布式数据库性能测试
---