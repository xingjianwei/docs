# Solr VS Elasticsearch

---

## 背景

Solr和Elasticsearch是很类似的搜索引擎，它们都基于Lucene开发的，拥有很多相同的核心特性，Lucene是一个搜索引擎基础jar包集合，很多搜索引擎应用都是基于Lucene API来开发的。Solr和ES也是基于Lucene API来开发的，Lucene上增加自己的特性，然后使用更简单的方式去访问。这样就不需要再通过Lucene进行编码，开发者可以轻松的通过HTTP去访问搜索服务，来进行索引/搜索。下面将对二者进行一个详尽的对比。

## API层面

Solr发布于2008年，ES发布于2010年，受限于发布的时代背景，二者在这方面还是有很大的不同的，不能简单的说哪个更好。

> Solr 问世于 2004 年（非release版本），但是一开始只是某公司的内部项目。2006 年公开发布其源代码，2007 开始应用于一些高流量的网站。2008 年1.3 版本的问世增加了一定的分布式搜索功能，例如可以做 Sharding。2010 年，ES 问世。ES 早期在各种搜索功能、社区、品牌、成熟度上都远远不及 Solr，但是因为它是针对分布式搜索而设计的，并且很快吸引了很多用户且不断迭代，因此很快在该领域成为 Solr 的有力的对手。虽然 2012 年 Solr 发布了 SolrCloud 对分布式搜索有了更好的支持，但是一来这时候 ES 已经在市场上站住脚，二来 ES 后起却在设计上致力于解决分布式搜索，因此很多需要分布式搜索的地方 ES 依然成为首选。

- 格式方面，Solr支持XML, CSV, JSON，而ES主要支持JSON。尽管如此，也不能简单的认为Solr就比ES在这方面要强。受限于时代背景，Solr早期不支持JSON，而由于ES天生支持JSON，对JSON的支持非常好，对父子、嵌套等支持的非常好，而Solr在4.8.0版本后才有改善。所以如果单纯使用JSON的角度使用ES更好，如果不是可以考虑使用Solr。

  > Solr 4.8.0 Release Highlights:
  >
  > JSON updates now support nested child documents, enabling {!child} and {!parent} block join queries.

- Elasticsearch发布于REST API时代，如果喜欢REST API，可能会觉得ES更好用，也更符合Web2.0标准。Solr也支持REST API，但支持没有ES那么全面，ES基本所有操作都可以通过REST API实现。

- 二者都支持多种语言，可能ES支持的语言更多一点，但ES没有一个类似于SolrJ那么好的API，非要使用的话只能使用TransportClient或一些额外的插件（Thrift ）。

- 状态监控，Solr支持jmx，ES通过REST API。

- 第三方产品集成，不管是开源产品还是商业产品，二者都被很多第三方产品所集成，很难说谁优谁劣。

## 分布式特性

从两个组件的发布顺序上看，ES要优于Solr，Solr发布时并不是分布式的搜索引擎，而ES的发布就是源于Solr缺乏分布式特性，在2012年10月发布SolrCloud后，Solr才开始支持分布式。

- **协调（coordination）**  ES使用自带的协调机制Zen Discovery来处理节点之间的状态，而Solr使用Zookeeper。这就意味着，要使用SolrCloud就需要先使用Zookeeper。当然现在在Hadoop生态圈下，绝大多数组件都基于Zookeeper。使用Zookeeper还可以防止脑裂，而ES存在这个问题。

  > 现在ES也可以外部加载Zookeeper作为协调器使用来防止脑裂。


- **分片切分（shard splitting）** Solr和ES都存在分片，使用分片可以将索引存放在集群的不同机器上。Solr支持分片切分，允许创建更多的分片来切分已有分片，ES不支持。

  > ES官方对于分片切分的看法：
  >
  > 用户经常会问为什么ES不支持分片切分（将已有分片切成多个的能力）。原因是分片切分是个坏主意：
  >
  > - 切分一个分片基本相当于重新索引数据。从一个节点复制一个分片到另外一个节点是一个非常耗资源的处理过程。
  > - 切分需要指数增长，有一个分片只能切分成2、4、8、16。。。切分不允许只增长50%的容量。
  > - 分片切分需要你有足够的容量来存放你的索引的第二份。通常情况下，当你意识到你需要扩展，你没有足够的自由空间来执行分片切分
  >
  > 因此，Elasticsearch不支持分片切分。可以通过为数据重建索引到合适数量的分片的方式解决这个问题。这个过程也是非常耗资源的，但至少可以自己控制分片的数量来容纳新建的索引。

- **分片自动均衡（Automatic Shard Rebalancing）** ES当集群扩容时支持自动均衡分片，移动分片到新节点。而Solr不支持。

- **Change # of shards** Solr分片能增加（使用implicit routing）或切分（使用compositeId）。不能减少。ES主分片数一旦创建不可变。

## Indexing

### Schema Creation

Solr和ElasticSearch都支持动态类型，动态索引新字段 （在定义schema之后）。ES是Schema Free，但是Schema Free不是说没有Schema，和Solr一样，ElasticSearch也可以设置document的schema，ES里的名字叫Mapping，其实无非就是设置document包含哪些Field，然后对每一个Field个性化的设置索引类型，是否存储，以及设置索引分析器和查询使用的分析器。Solr在索引字段之前需要预定义schema。在生产中，索引之前两者都需要预定义schema以便使用高级analyzers/filters。

### 嵌套类型（Nested Typing）

ES支持完整的嵌套类型。例如，有一个地址字段，可以包含家庭住址和工作地点两个字段，每个有可以包含街道、城市、编码等字段。这些嵌套类型包含很多一对多关系。这种关联关系应该将索引建立在一个分片上 。否则，因为嵌套关系可能会导致使用时异常的缓慢。Solr对于嵌套类型支持较差，索引的文档不能包含很多个嵌套。 对于嵌套类型，即使ES是支持的，也应该小心使用，避免出现问题。可参考[nested-type](http://www.elasticsearch.org/guide/reference/mapping/nested-type/)、[object-type](http://www.elasticsearch.org/guide/reference/mapping/object-type/)、[parent-field](http://www.elasticsearch.org/guide/reference/mapping/parent-field/)

## Searching

ES的查询DSL语法非常灵活，并且非常用于容易写一些复杂查询，尽管有点冗长。Solr的查询语法不够人性化，它的优势在于容易编写自定义的查询组件。ES 的纯 JSON 表达是略显臃肿，但这种 JSON 表达更具有一致性，所有的代码会有很类似的结构，更好懂。Solr 从最开始的设计就主要针对文本搜索。而 ES 对于非文本的数据似乎有着更好的支持。加上其比较一致性的语法规范，对于复杂的 Query 的组合，ES 语句的读写都要更简单明了。

> Solr查询语法基于键值对，并使用（）来解决嵌套，例如：q=((name:ryan* AND haircolor:brown) OR interest:zombies) OR (job: engineer*)，而ES主要使用JSON，如
>
> ```
> GET /_search
> {
>   "query": { 
>     "bool": { 
>       "must": [
>         { "match": { "title":   "Search"        }}, 
>         { "match": { "content": "Elasticsearch" }}  
>       ],
>       "filter": [ 
>         { "term":  { "status": "published" }}, 
>         { "range": { "publish_date": { "gte": "2015-01-01" }}} 
>       ]
>     }
>   }
> }
> ```

## 社区支持

两者都是基于apache协议开源的，用户群都很成熟，并没有很明显的区别。此外因为 Solr 是在Apache软件基金会之下的，相对来说更开放，而ES是允许任意用户Contribute 但是必须要经过elastic公司的人 Review 和 Approve 才能 Merge，就导致其可能为elastic公司自身的发展方向所局限。

## 文档

ES的文档编写的比较失败，经常会发现配置说明与真实使用的配置不符。Solr的文档编写非常好，在其中可以找到几乎任何关于查询或者更新索引的东西，而不需要从源码处去查。Solr的schema.xml和solrconfig.xml绝大多数的有用配置都被文档化了。

> 相对而言，Solr的中文资料较少，即使有，大多数是老版本的。而ES官方已经提供了中文版的文档，其他ES中文资料也非常多。

---

综合上面的对比，不论是 ES 还是 Solr，都是 Lucene 上的一层封装。这层封装按照自身的设计，提供给用户一层新的 API来方便用户的操作。但这层封装也必然隐藏了一些底部的功能。如果公司的应用和它们的设计的差别比较大，使用这样的封装就会反而显得很笨拙。所以，使用Solr还是ES，还是要从需求和自身的熟悉程度出发，不存在一个比另一个更好的情况。