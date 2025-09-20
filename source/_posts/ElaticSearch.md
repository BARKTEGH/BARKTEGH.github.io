---

title: ElasticSearch简介
date: 2019-07-01 21:16:20
tags:
- ElasticSearch
categories:
- ElasticSearch

---


## ElasticSearch基本概念


- **节点(node)**: 一个节点是一个逻辑上独立的服务，可以存储数据，并参与集群的索引和搜索功能, 一个节点也有唯一的名字，群集通过节点名称进行管理和通信.
- **索引（Index)** ： 索引与关系型数据库实例(Database)相当。索引只是一个 逻辑命名空间，它指向一个或多个分片(shards)，内部用Apache Lucene实现索引中数据的读写
- **文档类型（Type）**：相当于数据库中的table概念。每个文档在ElasticSearch中都必须设定它的类型。文档类型使得同一个索引中在存储结构不同文档时，只需要依据文档类型就可以找到对应的参数映射(Mapping)信息，方便文档的存取
- **文档（Document)** ：相当于数据库中的row， 是可以被索引的基本单位。文档是以JSON格式存储的。在一个索引中，您可以存储多个的文档。请注意，虽然在一个索引中有多分文档，但这些文档的结构是一致的，并在第一次存储的时候指定, 文档属于一种 类型(type)，各种各样的类型存在于一个索引中。
- **集群（Cluster)**: 包含一个或多个具有相同 cluster.name 的节点.
	1. 	集群内节点协同工作，共享数据，并共同分担工作负荷。
	2. 	由于节点是从属集群的，集群会自我重组来均匀地分发数据. 
	3. 	cluster Name是很重要的，因为每个节点只能是群集的一部分，当该节点被设置为相同的名称时，就会自动加入群集。
	4. 	集群中通过选举产生一个mater节点，它将负责管理集群范畴的变更，例如创建或删除索引，添加节点到集群或从集群删除节点。master 节点无需参与文档层面的变更和搜索，这意味着仅有一个 master 节点并不会因流量增长而成为瓶颈。任意一个节点都可以成为 master 节点。我们例举的集群只有一个节点，因此它会扮演 master 节点的角色。
	5. 	作为用户，我们可以访问包括 master 节点在内的集群中的任一节点。每个节点都知道各个文档的位置，并能够将我们的请求直接转发到拥有我们想要的数据的节点。无论我们访问的是哪个节点，它都会控制从拥有数据的节点收集响应的过程，并返回给客户端最终的结果。这一切都是由 Elasticsearch 透明管理的
- **分片(shard)** ：是 工作单元(worker unit) 底层的一员，用来分配集群中的数据，它只负责保存索引中所有数据的一小片。
	1. 分片是一个独立的Lucene实例，并且它自身也是一个完整的搜索引擎。
	1. 	文档存储并且被索引在分片中，但是我们的程序并不会直接与它们通信。取而代之，它们直接与索引进行通信的
	1. 	把分片想象成一个数据的容器。数据被存储在分片中，然后分片又被分配在集群的节点上。当你的集群扩展或者缩小时，elasticsearch 会自动的在节点之间迁移分配分片，以便集群保持均衡
	1. 	分片分为 主分片(primary shard) 以及 从分片(replica shard) 两种。在你的索引中，每一个文档都属于一个主分片
	1. 	从分片只是主分片的一个副本，它用于提供数据的冗余副本，在硬件故障时提供数据保护，同时服务于搜索和检索这种只读请求
	1. 	索引中的主分片的数量在索引创建后就固定下来了，但是从分片的数量可以随时改变。
	1. 	一个索引默认设置了5个主分片，每个主分片有一个从分片对应

- **副本（Replica）：**同一个分片(Shard)的备份数据，一个分片可能会有0个或多个副本，这些副本中的数据保证强一致或最终一致。

##  Elasticsearch集群

![](https://i.imgur.com/GXwjiNi.png)

## Elasticsearch集群搜索
（全文转载自 [Elasticsearch内核解析 - 查询篇](https://zhuanlan.zhihu.com/p/34674517)）

目前的Elasticsearch有两个明显的身份，一个是分布式搜索系统，另一个是分布式NoSQL数据库，对于这两种不同的身份，读写语义基本类似，但也有一点差异。
![](https://i.imgur.com/8GPAX41.png)

### 读操作

实时性对于搜索而言是近实时的，延迟在100ms以上，对于NoSQL则需要是实时的。

一致性指的是写入成功后，下次读操作一定要能读取到最新的数据。对于搜索，这个要求会低一些，可以有一些延迟。但是对于NoSQL数据库，则一般要求最好是强一致性的。

结果匹配上，NoSQL作为数据库，查询过程中只有符合不符合两种情况，而搜索里面还有是否相关，类似于NoSQL的结果只能是0或1，而搜索里面可能会有0.1，0.5，0.9等部分匹配或者更相关的情况。

结果召回上，搜索一般只需要召回最满足条件的Top N结果即可，而NoSQL一般都需要返回满足条件的所有结果。

搜索系统一般都是两阶段查询，第一个阶段查询到对应的Doc ID，也就是PK；第二阶段再通过Doc ID去查询完整文档，而NoSQL数据库一般是一阶段就返回结果。在Elasticsearch中两种都支持。

目前NoSQL的查询，聚合、分析和统计等功能上都是要比搜索弱的。

### Lucene的读

Elasticsearch使用了Lucene作为搜索引擎库，通过Lucene完成特定字段的搜索等功能，在Lucene中这个功能是通过IndexSearcher的下列接口实现的：

	public TopDocs search(Query query, int n);
	public Document doc(int docID);
	public int count(Query query);
	......(其他)

第一个search接口实现搜索功能，返回最满足Query的N个结果；第二个doc接口通过doc id查询Doc内容；第三个count接口通过Query获取到命中数。

这三个功能是搜索中的最基本的三个功能点，对于大部分Elasticsearch中的查询都是比较复杂的，直接用这个接口是无法满足需求的，比如分布式问题。这些问题都留给了Elasticsearch解决，我们接下来看Elasticsearch中相关读功能的剖析。

### Elasticsearch的读
Elasticsearch中每个Shard都会有多个Replica，主要是为了保证数据可靠性，除此之外，还可以增加读能力，因为写的时候虽然要写大部分Replica Shard，但是查询的时候只需要查询Primary和Replica中的任何一个就可以了。
![](https://i.imgur.com/iRNbl8w.png)


在上图中，该Shard有1个Primary和2个Replica Node，当查询的时候，从三个节点中根据Request中的preference参数选择一个节点查询。preference可以设置_local，_primary，_replica以及其他选项。如果选择了primary，则每次查询都是直接查询Primary，可以保证每次查询都是最新的。如果设置了其他参数，那么可能会查询到R1或者R2，这时候就有可能查询不到最新的数据。

接下来看一下，Elasticsearch中的查询是如何支持分布式的。

![](https://i.imgur.com/9qTJ332.png)

Elasticsearch中通过分区实现分布式，数据写入的时候根据_routing规则将数据写入某一个Shard中，这样就能将海量数据分布在多个Shard以及多台机器上，已达到分布式的目标。这样就导致了查询的时候，潜在数据会在当前index的所有的Shard中，所以Elasticsearch查询的时候需要查询所有Shard，同一个Shard的Primary和Replica选择一个即可，查询请求会分发给所有Shard，每个Shard中都是一个独立的查询引擎，比如需要返回Top 10的结果，那么每个Shard都会查询并且返回Top 10的结果，然后在Client Node里面会接收所有Shard的结果，然后通过优先级队列二次排序，选择出Top 10的结果返回给用户。

这里有一个问题就是请求膨胀，用户的一个搜索请求在Elasticsearch内部会变成Shard个请求，这里有个优化点，虽然是Shard个请求，但是这个Shard个数不一定要是当前Index中的Shard个数，只要是当前查询相关的Shard即可，这个需要基于业务和请求内容优化，通过这种方式可以优化请求膨胀数。

Elasticsearch中的查询主要分为两类，Get请求：通过ID查询特定Doc；Search请求：通过Query查询匹配Doc。
![](https://i.imgur.com/Jvc3vIp.png)

> 上图中内存中的Segment是指刚Refresh Segment，但是还没持久化到磁盘的新Segment，而非从磁盘加载到内存中的Segment。


对于Search类请求，查询的时候是一起查询内存和磁盘上的Segment，最后将结果合并后返回。这种查询是近实时（Near Real Time）的，主要是由于内存中的Index数据需要一段时间后才会刷新为Segment。

对于Get类请求，查询的时候是先查询内存中的TransLog，如果找到就立即返回，如果没找到再查询磁盘上的TransLog，如果还没有则再去查询磁盘上的Segment。这种查询是实时（Real Time）的。这种查询顺序可以保证查询到的Doc是最新版本的Doc，这个功能也是为了保证NoSQL场景下的实时性要求。

![](https://i.imgur.com/HV3mnwG.png)


所有的搜索系统一般都是两阶段查询，第一阶段查询到匹配的DocID，第二阶段再查询DocID对应的完整文档，这种在Elasticsearch中称为query_then_fetch，还有一种是一阶段查询的时候就返回完整Doc，在Elasticsearch中称作query_and_fetch，一般第二种适用于只需要查询一个Shard的请求。

除了一阶段，两阶段外，还有一种三阶段查询的情况。搜索里面有一种算分逻辑是根据TF（Term Frequency）和DF（Document Frequency）计算基础分，但是Elasticsearch中查询的时候，是在每个Shard中独立查询的，每个Shard中的TF和DF也是独立的，虽然在写入的时候通过_routing保证Doc分布均匀，但是没法保证TF和DF均匀，那么就有会导致局部的TF和DF不准的情况出现，这个时候基于TF、DF的算分就不准。为了解决这个问题，Elasticsearch中引入了DFS查询，比如DFS_query_then_fetch，会先收集所有Shard中的TF和DF值，然后将这些值带入请求中，再次执行query_then_fetch，这样算分的时候TF和DF就是准确的，类似的有DFS_query_and_fetch。这种查询的优势是算分更加精准，但是效率会变差。另一种选择是用BM25代替TF/DF模型。

在新版本Elasticsearch中，用户没法指定DFS_query_and_fetch和query_and_fetch，这两种只能被Elasticsearch系统改写。

### Elasticsearch查询流程
Elasticsearch中的大部分查询，以及核心功能都是Search类型查询，上面我们了解到查询分为一阶段，二阶段和三阶段，这里我们就以最常见的的二阶段查询为例来介绍查询流程。

![](https://i.imgur.com/0dEOtB8.png)

#### 注册Action

Elasticsearch中，查询和写操作一样都是在ActionModule.java中注册入口处理函数的。

	registerHandler.accept(new RestSearchAction(settings, restController));
	......
	actions.register(SearchAction.INSTANCE, TransportSearchAction.class);
	......

如果请求是Rest请求，则会在RestSearchAction中解析请求，检查查询类型，不能设置为dfs_query_and_fetch或者query_and_fetch，这两个目前只能用于Elasticsearch中的优化场景，然后将请求发给后面的TransportSearchAction处理。然后构造SearchRequest，将请求发送给TransportSearchAction处理。


如果是第一阶段的Query Phase请求，则会调用SearchService的executeQueryPhase方法。


如果是第二阶段的Fetch Phase请求，则会调用SearchService的executeFetchPhase方法。



#### Client Node

Client Node 也包括了前面说过的Parse Request，这里就不再赘述了，接下来看一下其他的部分。

1. Get Remove Cluster Shard

判断是否需要跨集群访问，如果需要，则获取到要访问的Shard列表。

2. Get Search Shard Iterator

获取当前Cluster中要访问的Shard，和上一步中的Remove Cluster Shard合并，构建出最终要访问的完整Shard列表。

这一步中，会根据Request请求中的参数从Primary Node和多个Replica Node中选择出一个要访问的Shard。

3. For Every Shard:Perform

遍历每个Shard，对每个Shard执行后面逻辑。

4. Send Request To Query Shard

将查询阶段请求发送给相应的Shard。

5. Merge Docs

上一步将请求发送给多个Shard后，这一步就是异步等待返回结果，然后对结果合并。这里的合并策略是维护一个Top N大小的优先级队列，每当收到一个shard的返回，就把结果放入优先级队列做一次排序，直到所有的Shard都返回。

翻页逻辑也是在这里，如果需要取Top 30~ Top 40的结果，这个的意思是所有Shard查询结果中的第30到40的结果，那么在每个Shard中无法确定最终的结果，每个Shard需要返回Top 40的结果给Client Node，然后Client Node中在merge docs的时候，计算出Top 40的结果，最后再去除掉Top 30，剩余的10个结果就是需要的Top 30~ Top 40的结果。

上述翻页逻辑有一个明显的缺点就是每次Shard返回的数据中包括了已经翻过的历史结果，如果翻页很深，则在这里需要排序的Docs会很多，比如Shard有1000，取第9990到10000的结果，那么这次查询，Shard总共需要返回1000 * 10000，也就是一千万Doc，这种情况很容易导致OOM。

另一种翻页方式是使用search_after，这种方式会更轻量级，如果每次只需要返回10条结构，则每个Shard只需要返回search_after之后的10个结果即可，返回的总数据量只是和Shard个数以及本次需要的个数有关，和历史已读取的个数无关。这种方式更安全一些，推荐使用这种。

如果有aggregate，也会在这里做聚合，但是不同的aggregate类型的merge策略不一样，具体的可以在后面的aggregate文章中再介绍。

6. Send Request To Fetch Shard

选出Top N个Doc ID后发送给这些Doc ID所在的Shard执行Fetch Phase，最后会返回Top N的Doc的内容。



#### Query Phase
接下来我们看第一阶段查询的步骤：

1. Create Search Context

创建Search Context，之后Search过程中的所有中间状态都会存在Context中，这些状态总共有50多个，具体可以查看DefaultSearchContext或者其他SearchContext的子类。

2. Parse Query

解析Query的Source，将结果存入Search Context。这里会根据请求中Query类型的不同创建不同的Query对象，比如TermQuery、FuzzyQuery等，最终真正执行TermQuery、FuzzyQuery等语义的地方是在Lucene中。

这里包括了dfsPhase、queryPhase和fetchPhase三个阶段的preProcess部分，只有queryPhase的preProcess中有执行逻辑，其他两个都是空逻辑，执行完preProcess后，所有需要的参数都会设置完成。

由于Elasticsearch中有些请求之间是相互关联的，并非独立的，比如scroll请求，所以这里同时会设置Context的生命周期。

同时会设置lowLevelCancellation是否打开，这个参数是集群级别配置，同时也能动态开关，打开后会在后面执行时做更多的检测，检测是否需要停止后续逻辑直接返回。

3. Get From Cache

判断请求是否允许被Cache，如果允许，则检查Cache中是否已经有结果，如果有则直接读取Cache，如果没有则继续执行后续步骤，执行完后，再将结果加入Cache。

4. Add Collectors

Collector主要目标是收集查询结果，实现排序，对自定义结果集过滤和收集等。这一步会增加多个Collectors，多个Collector组成一个List。

1. FilteredCollector：先判断请求中是否有Post Filter，Post Filter用于Search，Agg等结束后再次对结果做Filter，希望Filter不影响Agg结果。如果有Post Filter则创建一个FilteredCollector，加入Collector List中。
1. PluginInMultiCollector：判断请求中是否制定了自定义的一些Collector，如果有，则创建后加入Collector List。
1. MinimumScoreCollector：判断请求中是否制定了最小分数阈值，如果指定了，则创建MinimumScoreCollector加入Collector List中，在后续收集结果时，会过滤掉得分小于最小分数的Doc。
1. EarlyTerminatingCollector：判断请求中是否提前结束Doc的Seek，如果是则创建EarlyTerminatingCollector，加入Collector List中。在后续Seek和收集Doc的过程中，当Seek的Doc数达到Early Terminating后会停止Seek后续倒排链。
1. CancellableCollector：判断当前操作是否可以被中断结束，比如是否已经超时等，如果是会抛出一个TaskCancelledException异常。该功能一般用来提前结束较长的查询请求，可以用来保护系统。
1. EarlyTerminatingSortingCollector：如果Index是排序的，那么可以提前结束对倒排链的Seek，相当于在一个排序递减链表上返回最大的N个值，只需要直接返回前N个值就可以了。这个Collector会加到Collector List的头部。EarlyTerminatingSorting和EarlyTerminating的区别是，EarlyTerminatingSorting是一种对结果无损伤的优化，而EarlyTerminating是有损的，人为掐断执行的优化。
1. TopDocsCollector：这个是最核心的Top N结果选择器，会加入到Collector List的头部。TopScoreDocCollector和TopFieldCollector都是TopDocsCollector的子类，TopScoreDocCollector会按照固定的方式算分，排序会按照分数+doc id的方式排列，如果多个doc的分数一样，先选择doc id小的文档。而TopFieldCollector则是根据用户指定的Field的值排序。

5. lucene::search

这一步会调用Lucene中IndexSearch的search接口，执行真正的搜索逻辑。每个Shard中会有多个Segment，每个Segment对应一个LeafReaderContext，这里会遍历每个Segment，到每个Segment中去Search结果，然后计算分数。

搜索里面一般有两阶段算分，第一阶段是在这里算的，会对每个Seek到的Doc都计算分数，为了减少CPU消耗，一般是算一个基本分数。这一阶段完成后，会有个排序。然后在第二阶段，再对Top 的结果做一次二阶段算分，在二阶段算分的时候会考虑更多的因子。二阶段算分在后续操作中。

具体请求，比如TermQuery、WildcardQuery的查询逻辑都在Lucene中，后面会有专门文章介绍。

6. rescore

根据Request中是否包含rescore配置决定是否进行二阶段排序，如果有则执行二阶段算分逻辑，会考虑更多的算分因子。二阶段算分也是一种计算机中常见的多层设计，是一种资源消耗和效率的折中。

Elasticsearch中支持配置多个Rescore，这些rescore逻辑会顺序遍历执行。每个rescore内部会先按照请求参数window选择出Top window的doc，然后对这些doc排序，排完后再合并回原有的Top 结果顺序中。

7. suggest::execute()

如果有推荐请求，则在这里执行推荐请求。如果请求中只包含了推荐的部分，则很多地方可以优化。推荐不是今天的重点，这里就不介绍了，后面有机会再介绍。

8. aggregation::execute()

如果含有聚合统计请求，则在这里执行。Elasticsearch中的aggregate的处理逻辑也类似于Search，通过多个Collector来实现。在Client Node中也需要对aggregation做合并。aggregate逻辑更复杂一些，就不在这里赘述了，后面有需要就再单独开文章介绍。

上述逻辑都执行完成后，如果当前查询请求只需要查询一个Shard，那么会直接在当前Node执行Fetch Phase。



#### Fetch Phase
Elasticsearch作为搜索系统时，或者任何搜索系统中，除了Query阶段外，还会有一个Fetch阶段，这个Fetch阶段在数据库类系统中是没有的，是搜索系统中额外增加的阶段。搜索系统中额外增加Fetch阶段的原因是搜索系统中数据分布导致的，在搜索中，数据通过routing分Shard的时候，只能根据一个主字段值来决定，但是查询的时候可能会根据其他非主字段查询，那么这个时候所有Shard中都可能会存在相同非主字段值的Doc，所以需要查询所有Shard才能不会出现结果遗漏。同时如果查询主字段，那么这个时候就能直接定位到Shard，就只需要查询特定Shard即可，这个时候就类似于数据库系统了。另外，数据库中的二级索引又是另外一种情况，但类似于查主字段的情况，这里就不多说了。

基于上述原因，第一阶段查询的时候并不知道最终结果会在哪个Shard上，所以每个Shard中管都需要查询完整结果，比如需要Top 10，那么每个Shard都需要查询当前Shard的所有数据，找出当前Shard的Top 10，然后返回给Client Node。如果有100个Shard，那么就需要返回100 * 10 = 1000个结果，而Fetch Doc内容的操作比较耗费IO和CPU，如果在第一阶段就Fetch Doc，那么这个资源开销就会非常大。所以，一般是当Client Node选择出最终Top N的结果后，再对最终的Top N读取Doc内容。通过增加一点网络开销而避免大量IO和CPU操作，这个折中是非常划算的。

Fetch阶段的目的是通过DocID获取到用户需要的完整Doc内容。这些内容包括了DocValues，Store，Source，Script和Highlight等，具体的功能点是在SearchModule中注册的，系统默认注册的有：

- ExplainFetchSubPhase
- DocValueFieldsFetchSubPhase
- ScriptFieldsFetchSubPhase
- FetchSourceSubPhase
- VersionFetchSubPhase
- MatchedQueriesFetchSubPhase
- HighlightPhase
- ParentFieldSubFetchPhase

除了系统默认的8种外，还有通过插件的形式注册自定义的功能，这些SubPhase中最重要的是Source和Highlight，Source是加载原文，Highlight是计算高亮显示的内容片断。

上述多个SubPhase会针对每个Doc顺序执行，可能会产生多次的随机IO，这里会有一些优化方案，但是都是针对特定场景的，不具有通用性。

Fetch Phase执行完后，整个查询流程就结束了。

## Elasticsearch集群写入

（全文转载自 [Elasticsearch内核解析 - 写入篇](https://zhuanlan.zhihu.com/p/34669354)）
### 写操作

- 实时性：
	- 搜索系统的Index一般都是NRT（Near Real Time），近实时的，比如Elasticsearch中，Index的实时性是由refresh控制的，默认是1s，最快可到100ms，那么也就意味着Index doc成功后，需要等待一秒钟后才可以被搜索到。
	- NoSQL数据库的Write基本都是RT（Real Time），实时的，写入成功后，立即是可见的。Elasticsearch中的Index请求也能保证是实时的，因为Get请求会直接读内存中尚未Flush到存储介质的TransLog。
- 可靠性：
	- 搜索系统对可靠性要求都不高，一般数据的可靠性通过将原始数据存储在另一个存储系统来保证，当搜索系统的数据发生丢失时，再从其他存储系统导一份数据过来重新rebuild就可以了。在Elasticsearch中，通过设置TransLog的Flush频率可以控制可靠性，要么是按请求，每次请求都Flush；要么是按时间，每隔一段时间Flush一次。一般为了性能考虑，会设置为每隔5秒或者1分钟Flush一次，Flush间隔时间越长，可靠性就会越低。
	- NoSQL数据库作为一款数据库，必须要有很高的可靠性，数据可靠性是生命底线，决不能有闪失。如果把Elasticsearch当做NoSQL数据库，此时需要设置TransLog的Flush策略为每个请求都要Flush，这样才能保证当前Shard写入成功后，数据能尽量持久化下来。


### 写操作的关键点

在考虑或分析一个分布式系统的写操作时，一般需要从下面几个方面考虑：

- 可靠性：或者是持久性，数据写入系统成功后，数据不会被回滚或丢失。
- 一致性：数据写入成功后，再次查询时必须能保证读取到最新版本的数据，不能读取到旧数据。
- 原子性：一个写入或者更新操作，要么完全成功，要么完全失败，不允许出现中间状态。
- 隔离性：多个写入操作相互不影响。
- 实时性：写入后是否可以立即被查询到。
- 性能：写入性能，吞吐量到底怎么样。


Elasticsearch作为分布式系统，也需要在写入的时候满足上述的四个特点，我们在后面的写流程介绍中会涉及到上述四个方面。

接下来,我们一层一层剖析Elasticsearch内部的写机制。

### Lucene的写
众所周知，Elasticsearch内部使用了Lucene完成索引创建和搜索功能，Lucene中写操作主要是通过IndexWriter类实现，IndexWriter提供三个接口：

	 public long addDocument();
	 public long updateDocuments();
	 public long deleteDocuments();

通过这三个接口可以完成单个文档的写入，更新和删除功能，包括了分词，倒排创建，正排创建等等所有搜索相关的流程。只要Doc通过IndesWriter写入后，后面就可以通过IndexSearcher搜索了，看起来功能已经完善了，但是仍然有一些问题没有解：

1. 上述操作是单机的，而不是我们需要的分布式。
1. 文档写入Lucene后并不是立即可查询的，需要生成完整的Segment后才可被搜索，如何保证实时性？
1. Lucene生成的Segment是在内存中，如果机器宕机或掉电后，内存中的Segment会丢失，如何保证数据可靠性 ？
1. Lucene不支持部分文档更新，但是这又是一个强需求，如何支持部分更新？

上述问题，在Lucene中是没有解决的，那么就需要Elasticsearch中解决上述问题。


### Elasticsearch的写

Elasticsearch采用多Shard方式，通过配置routing规则将数据分成多个数据子集，每个数据子集提供独立的索引和搜索功能。当写入文档的时候，根据routing规则，将文档发送给特定Shard中建立索引。这样就能实现分布式了。

此外，Elasticsearch整体架构上采用了一主多副的方式：

![](https://i.imgur.com/4letnXs.png)


每个Index由多个Shard组成，每个Shard有一个主节点和多个副本节点，副本个数可配。但每次写入的时候，写入请求会先根据_routing规则选择发给哪个Shard，Index Request中可以设置使用哪个Filed的值作为路由参数，如果没有设置，则使用Mapping中的配置，如果mapping中也没有配置，则使用_id作为路由参数，然后通过_routing的Hash值选择出Shard（在OperationRouting类中），最后从集群的Meta中找出出该Shard的Primary节点。

请求接着会发送给Primary Shard，在Primary Shard上执行成功后，再从Primary Shard上将请求同时发送给多个Replica Shard，请求在多个Replica Shard上执行成功并返回给Primary Shard后，写入请求执行成功，返回结果给客户端。

这种模式下，写入操作的延时就等于latency = Latency(Primary Write) + Max(Replicas Write)。只要有副本在，写入延时最小也是两次单Shard的写入时延总和，写入效率会较低，但是这样的好处也很明显，避免写入后，单机或磁盘故障导致数据丢失，在数据重要性和性能方面，一般都是优先选择数据，除非一些允许丢数据的特殊场景。

采用多个副本后，避免了单机或磁盘故障发生时，对已经持久化后的数据造成损害，但是Elasticsearch里为了减少磁盘IO保证读写性能，一般是每隔一段时间（比如5分钟）才会把Lucene的Segment写入磁盘持久化，对于写入内存，但还未Flush到磁盘的Lucene数据，如果发生机器宕机或者掉电，那么内存中的数据也会丢失，这时候如何保证？

对于这种问题，Elasticsearch学习了数据库中的处理方式：增加CommitLog模块，Elasticsearch中叫TransLog。

![](https://i.imgur.com/318wR0R.png)

在每一个Shard中，写入流程分为两部分，先写入Lucene，再写入TransLog。

写入请求到达Shard后，先写Lucene文件，创建好索引，此时索引还在内存里面，接着去写TransLog，写完TransLog后，刷新TransLog数据到磁盘上，写磁盘成功后，请求返回给用户。这里有几个关键点，一是和数据库不同，数据库是先写CommitLog，然后再写内存，而Elasticsearch是先写内存，最后才写TransLog，一种可能的原因是Lucene的内存写入会有很复杂的逻辑，很容易失败，比如分词，字段长度超过限制等，比较重，为了避免TransLog中有大量无效记录，减少recover的复杂度和提高速度，所以就把写Lucene放在了最前面。二是写Lucene内存后，并不是可被搜索的，需要通过Refresh把内存的对象转成完整的Segment后，然后再次reopen后才能被搜索，一般这个时间设置为1秒钟，导致写入Elasticsearch的文档，最快要1秒钟才可被从搜索到，所以Elasticsearch在搜索方面是NRT（Near Real Time）近实时的系统。三是当Elasticsearch作为NoSQL数据库时，查询方式是GetById，这种查询可以直接从TransLog中查询，这时候就成了RT（Real Time）实时系统。四是每隔一段比较长的时间，比如30分钟后，Lucene会把内存中生成的新Segment刷新到磁盘上，刷新后索引文件已经持久化了，历史的TransLog就没用了，会清空掉旧的TransLog。

上面介绍了Elasticsearch在写入时的两个关键模块，Replica和TransLog，接下来，我们看一下Update流程：

![](https://i.imgur.com/eYTGXM6.png)

Lucene中不支持部分字段的Update，所以需要在Elasticsearch中实现该功能，具体流程如下：

1. 收到Update请求后，从Segment或者TransLog中读取同id的完整Doc，记录版本号为V1。
1. 将版本V1的全量Doc和请求中的部分字段Doc合并为一个完整的Doc，同时更新内存中的VersionMap。获取到完整Doc后，Update请求就变成了Index请求。
1. 加锁。
1. 再次从versionMap中读取该id的最大版本号V2，如果versionMap中没有，则从Segment或者TransLog中读取，这里基本都会从versionMap中获取到。
1. 检查版本是否冲突(V1==V2)，如果冲突，则回退到开始的“Update doc”阶段，重新执行。如果不冲突，则执行最新的Add请求。
1. 在Index Doc阶段，首先将Version + 1得到V3，再将Doc加入到Lucene中去，Lucene中会先删同id下的已存在doc id，然后再增加新Doc。写入Lucene成功后，将当前V3更新到versionMap中。
1. 释放锁，部分更新的流程就结束了。


介绍完部分更新的流程后，大家应该从整体架构上对Elasticsearch的写入有了一个初步的映象，接下来我们详细剖析下写入的详细步骤。

### Elasticsearch写入请求类型
Elasticsearch中的写入请求类型，主要包括下列几个：Index(Create)，Update，Delete和Bulk，其中前3个是单文档操作，后一个Bulk是多文档操作，其中Bulk中可以包括Index(Create)，Update和Delete。

在6.0.0及其之后的版本中，前3个单文档操作的实现基本都和Bulk操作一致，甚至有些就是通过调用Bulk的接口实现的。估计接下来几个版本后，Index(Create)，Update，Delete都会被当做Bulk的一种特例化操作被处理。这样，代码和逻辑都会更清晰一些。

下面，我们就以Bulk请求为例来介绍写入流程。

### Elasticsearch写入流程图

![](https://i.imgur.com/gijyfoP.png)

- 红色：Client Node。
- 绿色：Primary Node。
- 蓝色：Replica Node。

#### 注册Action
在Elasticsearch中，所有action的入口处理方法都是注册在ActionModule.java中，比如Bulk Request有两个注册入口，分别是Rest和Transport入口.


如果请求是Rest请求，则会在RestBulkAction中Parse Request，构造出BulkRequest，然后发给后面的TransportAction处理。

TransportShardBulkAction的基类TransportReplicationAction中注册了对Primary，Replica等的不同处理入口:


这里对原始请求，Primary Node请求和Replica Node请求各自注册了一个handler处理入口。

#### Client Node

Client Node 也包括了前面说过的Parse Request，这里就不再赘述了，接下来看一下其他的部分。

1. Ingest Pipeline

在这一步可以对原始文档做一些处理，比如HTML解析，自定义的处理，具体处理逻辑可以通过插件来实现。在Elasticsearch中，由于Ingest Pipeline会比较耗费CPU等资源，可以设置专门的Ingest Node，专门用来处理Ingest Pipeline逻辑。

如果当前Node不能执行Ingest Pipeline，则会将请求发给另一台可以执行Ingest Pipeline的Node。

2. Auto Create Index

判断当前Index是否存在，如果不存在，则需要自动创建Index，这里需要和Master交互。也可以通过配置关闭自动创建Index的功能。

3. Set Routing

设置路由条件，如果Request中指定了路由条件，则直接使用Request中的Routing，否则使用Mapping中配置的，如果Mapping中无配置，则使用默认的_id字段值。

在这一步中，如果没有指定id字段，则会自动生成一个唯一的_id字段，目前使用的是UUID。

4. Construct BulkShardRequest

由于Bulk Request中会包括多个(Index/Update/Delete)请求，这些请求根据routing可能会落在多个Shard上执行，这一步会按Shard挑拣Single Write Request，同一个Shard中的请求聚集在一起，构建BulkShardRequest，每个BulkShardRequest对应一个Shard。

5. Send Request To Primary

这一步会将每一个BulkShardRequest请求发送给相应Shard的Primary Node。



#### Primary Node
Primary 请求的入口是在PrimaryOperationTransportHandler的messageReceived，我们来看一下相关的逻辑流程。

1. Index or Update or Delete

循环执行每个Single Write Request，对于每个Request，根据操作类型(CREATE/INDEX/UPDATE/DELETE)选择不同的处理逻辑。

其中，Create/Index是直接新增Doc，Delete是直接根据_id删除Doc，Update会稍微复杂些，我们下面就以Update为例来介绍。

2. Translate Update To Index or Delete

这一步是Update操作的特有步骤，在这里，会将Update请求转换为Index或者Delete请求。首先，会通过GetRequest查询到已经存在的同_id Doc（如果有）的完整字段和值（依赖_source字段），然后和请求中的Doc合并。同时，这里会获取到读到的Doc版本号，记做V1。

3. Parse Doc

这里会解析Doc中各个字段。生成ParsedDocument对象，同时会生成uid Term。在Elasticsearch中，_uid = type # _id，对用户，_Id可见，而Elasticsearch中存储的是_uid。这一部分生成的ParsedDocument中也有Elasticsearch的系统字段，大部分会根据当前内容填充，部分未知的会在后面继续填充ParsedDocument。

4. Update Mapping

Elasticsearch中有个自动更新Mapping的功能，就在这一步生效。会先挑选出Mapping中未包含的新Field，然后判断是否运行自动更新Mapping，如果允许，则更新Mapping。

5. Get Sequence Id and Version

由于当前是Primary Shard，则会从SequenceNumber Service获取一个sequenceID和Version。SequenceID在Shard级别每次递增1，SequenceID在写入Doc成功后，会用来初始化LocalCheckpoint。Version则是根据当前Doc的最大Version递增1。

6. Add Doc To Lucene

这一步开始的时候会给特定_uid加锁，然后判断该_uid对应的Version是否等于之前Translate Update To Index步骤里获取到的Version，如果不相等，则说明刚才读取Doc后，该Doc发生了变化，出现了版本冲突，这时候会抛出一个VersionConflict的异常，该异常会在Primary Node最开始处捕获，重新从“Translate Update To Index or Delete”开始执行。

如果Version相等，则继续执行，如果已经存在同id的Doc，则会调用Lucene的UpdateDocument(uid, doc)接口，先根据uid删除Doc，然后再Index新Doc。如果是首次写入，则直接调用Lucene的AddDocument接口完成Doc的Index，AddDocument也是通过UpdateDocument实现。

这一步中有个问题是，如何保证Delete-Then-Add的原子性，怎么避免中间状态时被Refresh？答案是在开始Delete之前，会加一个Refresh Lock，禁止被Refresh，只有等Add完后释放了Refresh Lock后才能被Refresh，这样就保证了Delete-Then-Add的原子性。

Lucene的UpdateDocument接口中就只是处理多个Field，会遍历每个Field逐个处理，处理顺序是invert index，store field，doc values，point dimension，后续会有文章专门介绍Lucene中的写入。

7. Write Translog

写完Lucene的Segment后，会以keyvalue的形式写TransLog，Key是_id，Value是Doc内容。当查询的时候，如果请求是GetDocByID，则可以直接根据_id从TransLog中读取到，满足NoSQL场景下的实时性要去。

需要注意的是，这里只是写入到内存的TransLog，是否Sync到磁盘的逻辑还在后面。

这一步的最后，会标记当前SequenceID已经成功执行，接着会更新当前Shard的LocalCheckPoint。

8. Renew Bulk Request

这里会重新构造Bulk Request，原因是前面已经将UpdateRequest翻译成了Index或Delete请求，则后续所有Replica中只需要执行Index或Delete请求就可以了，不需要再执行Update逻辑，一是保证Replica中逻辑更简单，性能更好，二是保证同一个请求在Primary和Replica中的执行结果一样。

9. Flush Translog

这里会根据TransLog的策略，选择不同的执行方式，要么是立即Flush到磁盘，要么是等到以后再Flush。Flush的频率越高，可靠性越高，对写入性能影响越大。

10. Send Requests To Replicas

这里会将刚才构造的新的Bulk Request并行发送给多个Replica，然后等待Replica的返回，这里需要等待所有Replica返回后（可能有成功，也有可能失败），Primary Node才会返回用户。如果某个Replica失败了，则Primary会给Master发送一个Remove Shard请求，要求Master将该Replica Shard从可用节点中移除。

这里，同时会将SequenceID，PrimaryTerm，GlobalCheckPoint等传递给Replica。

发送给Replica的请求中，Action Name等于原始ActionName + [R]，这里的R表示Replica。通过这个[R]的不同，可以找到处理Replica请求的Handler。

11. Receive Response From Replicas

Replica中请求都处理完后，会更新Primary Node的LocalCheckPoint。



#### Replica Node
Replica 请求的入口是在ReplicaOperationTransportHandler的messageReceived，我们来看一下相关的逻辑流程。

1. Index or Delete

根据请求类型是Index还是Delete，选择不同的执行逻辑。这里没有Update，是因为在Primary Node中已经将Update转换成了Index或Delete请求了。

2. Parse Doc

3. Update Mapping

以上都和Primary Node中逻辑一致。

4. Get Sequence Id and Version

Primary Node中会生成Sequence ID和Version，然后放入ReplicaRequest中，这里只需要从Request中获取到就行。

5. Add Doc To Lucene

由于已经在Primary Node中将部分Update请求转换成了Index或Delete请求，这里只需要处理Index和Delete两种请求，不再需要处理Update请求了。比Primary Node会更简单一些。

6. Write Translog

7. Flush Translog

以上都和Primary Node中逻辑一致。



### ES写入总结
上面详细介绍了Elasticsearch的写入流程及其各个流程的工作机制，我们在这里再次总结下之前提出的分布式系统中的六大特性：

可靠性：由于Lucene的设计中不考虑可靠性，在Elasticsearch中通过Replica和TransLog两套机制保证数据的可靠性。
一致性：Lucene中的Flush锁只保证Update接口里面Delete和Add中间不会Flush，但是Add完成后仍然有可能立即发生Flush，导致Segment可读。这样就没法保证Primary和所有其他Replica可以同一时间Flush，就会出现查询不稳定的情况，这里只能实现最终一致性。
原子性：Add和Delete都是直接调用Lucene的接口，是原子的。当部分更新时，使用Version和锁保证更新是原子的。
隔离性：仍然采用Version和局部锁来保证更新的是特定版本的数据。
实时性：使用定期Refresh Segment到内存，并且Reopen Segment方式保证搜索可以在较短时间（比如1秒）内被搜索到。通过将未刷新到磁盘数据记入TransLog，保证对未提交数据可以通过ID实时访问到。
性能：性能是一个系统性工程，所有环节都要考虑对性能的影响，在Elasticsearch中，在很多地方的设计都考虑到了性能，一是不需要所有Replica都返回后才能返回给用户，只需要返回特定数目的就行；二是生成的Segment现在内存中提供服务，等一段时间后才刷新到磁盘，Segment在内存这段时间的可靠性由TransLog保证；三是TransLog可以配置为周期性的Flush，但这个会给可靠性带来伤害；四是每个线程持有一个Segment，多线程时相互不影响，相互独立，性能更好；五是系统的写入流程对版本依赖较重，读取频率较高，因此采用了versionMap，减少热点数据的多次磁盘IO开销。Lucene中针对性能做了大量的优化。后面我们也会有文章专门介绍Lucene中的优化思路。

## Elasticsearch分布式一致性原理剖析-节点

（全文转载自https://zhuanlan.zhihu.com/p/34858035）

### Elasticsearch分布式一致性原理剖析-节点目录

Elasticsearch分布式一致性原理剖析”系列将会对Elasticsearch的分布式一致性原理进行详细的剖析，介绍其实现方式、原理以及其存在的问题等(基于6.2版本)。

ES目前是最流行的分布式搜索引擎系统，其使用Lucene作为单机存储引擎并提供强大的搜索查询能力。学习其搜索原理，则必须了解Lucene，而学习ES的架构，就必须了解其分布式如何实现，而一致性是分布式系统的核心之一。

本篇将介绍ES的集群组成、节点发现与Master选举，错误检测与扩缩容相关的内容。ES在处理节点发现与Master选举等方面没有选择Zookeeper等外部组件，而是自己实现的一套，本文会介绍ES的这套机制是如何工作的，存在什么问题。本文的主要内容如下：

1. ES集群构成
2. 节点发现
3. Master选举
4. 错误检测
5. 集群扩缩容
6. 与Zookeeper、raft等实现方式的比较

### ES集群构成

首先，一个Elasticsearch集群(下面简称ES集群)是由许多节点(Node)构成的，Node可以有不同的类型，通过以下配置，可以产生四种不同类型的Node：

	conf/elasticsearch.yml:
		node.master: true/false
		node.data: true/false

四种不同类型的Node是一个node.master和node.data的true/false的两两组合。当然还有其他类型的Node，比如IngestNode(用于数据预处理等)，不在本文讨论范围内。

当node.master为true时，其表示这个node是一个master的候选节点，可以参与选举，在ES的文档中常被称作master-eligible node，类似于MasterCandidate。ES正常运行时只能有一个master(即leader)，多于1个时会发生脑裂。

当node.data为true时，这个节点作为一个数据节点，会存储分配在该node上的shard的数据并负责这些shard的写入、查询等。

此外，任何一个集群内的node都可以执行任何请求，其会负责将请求转发给对应的node进行处理，所以当node.master和node.data都为false时，这个节点可以作为一个类似proxy的节点，接受请求并进行转发、结果聚合等。

![](https://i.imgur.com/gXb00hk.png)


上图是一个ES集群的示意图，其中NodeA是当前集群的Master，NodeB和NodeC是Master的候选节点，其中NodeA和NodeB同时也是数据节点(DataNode)，此外，NodeD是一个单纯的数据节点，Node_E是一个proxy节点。每个Node会跟其他所有Node建立连接。

到这里，我们提一个问题，供读者思考：一个ES集群应当配置多少个master-eligible node，当集群的存储或者计算资源不足，需要扩容时，新扩上去的节点应该设置为何种类型？


### 节点发现


ZenDiscovery是ES自己实现的一套用于节点发现和选主等功能的模块，没有依赖Zookeeper等工具，官方文档：

[https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html)

简单来说，节点发现依赖以下配置：

	conf/elasticsearch.yml:
	    discovery.zen.ping.unicast.hosts: [1.1.1.1, 1.1.1.2, 1.1.1.3]

这个配置可以看作是，在本节点到每个hosts中的节点建立一条边，当整个集群所有的node形成一个联通图时，所有节点都可以知道集群中有哪些节点，不会形成孤岛。

官方推荐这里设置为所有的master-eligible node，读者可以想想这样有何好处：

    It is recommended that the unicast hosts list be maintained as the list of master-eligible


### Master选举

上面提到，集群中可能会有多个master-eligible node，此时就要进行master选举，保证只有一个当选master。如果有多个node当选为master，则集群会出现脑裂，脑裂会破坏数据的一致性，导致集群行为不可控，产生各种非预期的影响。

为了避免产生脑裂，ES采用了常见的分布式系统思路，保证选举出的master被多数派(quorum)的master-eligible node认可，以此来保证只有一个master。这个quorum通过以下配置进行配置：

	conf/elasticsearch.yml:
	    discovery.zen.minimum_master_nodes: 2

这个配置对于整个集群非常重要。

#### master选举谁发起，什么时候发起？
master选举当然是由master-eligible节点发起，当一个master-eligible节点发现满足以下条件时发起选举：

该master-eligible节点的当前状态不是master。
该master-eligible节点通过ZenDiscovery模块的ping操作询问其已知的集群其他节点，没有任何节点连接到master。
包括本节点在内，当前已有超过minimum_master_nodes个节点没有连接到master。
总结一句话，即当一个节点发现包括自己在内的多数派的master-eligible节点认为集群没有master时，就可以发起master选举。

#### 当需要选举master时，选举谁？
首先是选举谁的问题，如下面源码所示，选举的是排序后的第一个MasterCandidate(即master-eligible node)。

	public MasterCandidate electMaster(Collection<MasterCandidate> candidates) {
	        assert hasEnoughCandidates(candidates);
	        List<MasterCandidate> sortedCandidates = new ArrayList<>(candidates);
	        sortedCandidates.sort(MasterCandidate::compare);
	        return sortedCandidates.get(0);
	    }

那么是按照什么排序的？

	public static int compare(MasterCandidate c1, MasterCandidate c2) {
	    // we explicitly swap c1 and c2 here. the code expects "better" is lower in a sorted
	    // list, so if c2 has a higher cluster state version, it needs to come first.
	    int ret = Long.compare(c2.clusterStateVersion, c1.clusterStateVersion);
	    if (ret == 0) {
	        ret = compareNodes(c1.getNode(), c2.getNode());
	    }
	    return ret;
	}

如上面源码所示，先根据节点的clusterStateVersion比较，clusterStateVersion越大，优先级越高。clusterStateVersion相同时，进入compareNodes，其内部按照节点的Id比较(Id为节点第一次启动时随机生成)。

总结一下：

当clusterStateVersion越大，优先级越高。这是为了保证新Master拥有最新的clusterState(即集群的meta)，避免已经commit的meta变更丢失。因为Master当选后，就会以这个版本的clusterState为基础进行更新。(一个例外是集群全部重启，所有节点都没有meta，需要先选出一个master，然后master再通过持久化的数据进行meta恢复，再进行meta同步)。
当clusterStateVersion相同时，节点的Id越小，优先级越高。即总是倾向于选择Id小的Node，这个Id是节点第一次启动时生成的一个随机字符串。之所以这么设计，应该是为了让选举结果尽可能稳定，不要出现都想当master而选不出来的情况。

#### 什么时候选举成功？

当一个master-eligible node(我们假设为Node_A)发起一次选举时，它会按照上述排序策略选出一个它认为的master。

假设Node_A选Node_B当Master：
Node_A会向Node_B发送join请求，那么此时：

(1) 如果Node_B已经成为Master，Node_B就会把Node_A加入到集群中，然后发布最新的cluster_state, 最新的cluster_state就会包含Node_A的信息。相当于一次正常情况的新节点加入。对于Node_A，等新的cluster_state发布到Node_A的时候，Node_A也就完成join了。

(2) 如果Node_B在竞选Master，那么Node_B会把这次join当作一张选票。对于这种情况，Node_A会等待一段时间，看Node_B是否能成为真正的Master，直到超时或者有别的Master选成功。

(3) 如果Node_B认为自己不是Master(现在不是，将来也选不上)，那么Node_B会拒绝这次join。对于这种情况，Node_A会开启下一轮选举。

假设Node_A选自己当Master：
此时NodeA会等别的node来join，即等待别的node的选票，当收集到超过半数的选票时，认为自己成为master，然后变更cluster_state中的master node为自己，并向集群发布这一消息。

有兴趣的同学可以看看下面这段源码：

	if (transportService.getLocalNode().equals(masterNode)) {
	            final int requiredJoins = Math.max(0, electMaster.minimumMasterNodes() - 1); // we count as one
	            logger.debug("elected as master, waiting for incoming joins ([{}] needed)", requiredJoins);
	            nodeJoinController.waitToBeElectedAsMaster(requiredJoins, masterElectionWaitForJoinsTimeout,
	                    new NodeJoinController.ElectionCallback() {
	                        @Override
	                        public void onElectedAsMaster(ClusterState state) {
	                            synchronized (stateMutex) {
	                                joinThreadControl.markThreadAsDone(currentThread);
	                            }
	                        }
	
	                        @Override
	                        public void onFailure(Throwable t) {
	                            logger.trace("failed while waiting for nodes to join, rejoining", t);
	                            synchronized (stateMutex) {
	                                joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
	                            }
	                        }
	                    }
	
	            );
	        } else {
	            // process any incoming joins (they will fail because we are not the master)
	            nodeJoinController.stopElectionContext(masterNode + " elected");
	
	            // send join request
	            final boolean success = joinElectedMaster(masterNode);
	
	            synchronized (stateMutex) {
	                if (success) {
	                    DiscoveryNode currentMasterNode = this.clusterState().getNodes().getMasterNode();
	                    if (currentMasterNode == null) {
	                        // Post 1.3.0, the master should publish a new cluster state before acking our join request. we now should have
	                        // a valid master.
	                        logger.debug("no master node is set, despite of join request completing. retrying pings.");
	                        joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
	                    } else if (currentMasterNode.equals(masterNode) == false) {
	                        // update cluster state
	                        joinThreadControl.stopRunningThreadAndRejoin("master_switched_while_finalizing_join");
	                    }
	
	                    joinThreadControl.markThreadAsDone(currentThread);
	                } else {
	                    // failed to join. Try again...
	                    joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
	                }
	            }
	        }

按照上述流程，我们描述一个简单的场景来帮助大家理解：

假如集群中有3个master-eligible node，分别为Node_A、 Node_B、 Node_C, 选举优先级也分别为Node_A、Node_B、Node_C。三个node都认为当前没有master，于是都各自发起选举，选举结果都为Node_A(因为选举时按照优先级排序，如上文所述)。于是Node_A开始等join(选票)，Node_B、Node_C都向Node_A发送join，当Node_A接收到一次join时，加上它自己的一票，就获得了两票了(超过半数)，于是Node_A成为Master。此时cluster_state(集群状态)中包含两个节点，当Node_A再收到另一个节点的join时，cluster_state包含全部三个节点。

#### 选举怎么保证不脑裂？

基本原则还是多数派的策略，如果必须得到多数派的认可才能成为Master，那么显然不可能有两个Master都得到多数派的认可。

上述流程中，master候选人需要等待多数派节点进行join后才能真正成为master，就是为了保证这个master得到了多数派的认可。但是我这里想说的是，上述流程在绝大部份场景下没问题，听上去也非常合理，但是却是有bug的。

因为上述流程并没有限制在选举过程中，一个Node只能投一票，那么什么场景下会投两票呢？比如NodeB投NodeA一票，但是NodeA迟迟不成为Master，NodeB等不及了发起了下一轮选主，这时候发现集群里多了个Node0，Node0优先级比NodeA还高，那NodeB肯定就改投Node0了。假设Node0和NodeA都处在等选票的环节，那显然这时候NodeB其实发挥了两票的作用，而且投给了不同的人。

那么这种问题应该怎么解决呢，比如raft算法中就引入了选举周期(term)的概念，保证了每个选举周期中每个成员只能投一票，如果需要再投就会进入下一个选举周期，term+1。假如最后出现两个节点都认为自己是master，那么肯定有一个term要大于另一个的term，而且因为两个term都收集到了多数派的选票，所以多数节点的term是较大的那个，保证了term小的master不可能commit任何状态变更(commit需要多数派节点先持久化日志成功，由于有term检测，不可能达到多数派持久化条件)。这就保证了集群的状态变更总是一致的。

而ES目前(6.2版本)并没有解决这个问题，构造类似场景的测试case可以看到会选出两个master，两个node都认为自己是master，向全集群发布状态变更，这个发布也是两阶段的，先保证多数派节点“接受”这次变更，然后再要求全部节点commit这次变更。很不幸，目前两个master可能都完成第一个阶段，进入commit阶段，导致节点间状态出现不一致，而在raft中这是不可能的。那么为什么都能完成第一个阶段呢，因为第一个阶段ES只是将新的cluster_state做简单的检查后放入内存队列，如果当前cluster_state的master为空，不会对新的clusterstate中的master做检查，即在接受了NodeA成为master的cluster_state后(还未commit)，还可以继续接受NodeB成为master的cluster_state。这就使NodeA和NodeB都能达到commit条件，发起commit命令，从而将集群状态引向不一致。当然，这种脑裂很快会自动恢复，因为不一致发生后某个master再次发布cluster_state时就会发现无法达到多数派条件，或者是发现它的follower并不构成多数派而自动降级为candidate等。

这里要表达的是，ES的ZenDiscovery模块与成熟的一致性方案相比，在某些特殊场景下存在缺陷，下一篇文章讲ES的meta变更流程时也会分析其他的ES无法满足一致性的场景。


### 错误检测

####  MasterFaultDetection与NodesFaultDetection
这里的错误检测可以理解为类似心跳的机制，有两类错误检测，一类是Master定期检测集群内其他的Node，另一类是集群内其他的Node定期检测当前集群的Master。检查的方法就是定期执行ping请求。ES文档：

There are two fault detection processes running. The first is by the master, to ping all the other nodes in the cluster and verify that they are alive. And on the other end, each node pings to master to verify if its still alive or an election process needs to be initiated.
如果Master检测到某个Node连不上了，会执行removeNode的操作，将节点从cluste_state中移除，并发布新的cluster_state。当各个模块apply新的cluster_state时，就会执行一些恢复操作，比如选择新的primaryShard或者replica，执行数据复制等。

如果某个Node发现Master连不上了，会清空pending在内存中还未commit的new cluster_state，然后发起rejoin，重新加入集群(如果达到选举条件则触发新master选举)。

#### rejoin
除了上述两种情况，还有一种情况是Master发现自己已经不满足多数派条件(>=minimumMasterNodes)了，需要主动退出master状态(退出master状态并执行rejoin)以避免脑裂的发生，那么master如何发现自己需要rejoin呢？

上面提到，当有节点连不上时，会执行removeNode。在执行removeNode时判断剩余的Node是否满足多数派条件，如果不满足，则执行rejoin。
        
	if (electMasterService.hasEnoughMasterNodes(remainingNodesClusterState.nodes()) == false) {
                final int masterNodes = electMasterService.countMasterNodes(remainingNodesClusterState.nodes());
                rejoin.accept(LoggerMessageFormat.format("not enough master nodes (has [{}], but needed [{}])",
                                                         masterNodes, electMasterService.minimumMasterNodes()));
                return resultBuilder.build(currentState);
            } else {
                return resultBuilder.build(allocationService.deassociateDeadNodes(remainingNodesClusterState, true, describeTasks(tasks)));
            }

在publish新的cluster_state时，分为send阶段和commit阶段，send阶段要求多数派必须成功，然后再进行commit。如果在send阶段没有实现多数派返回成功，那么可能是有了新的master或者是无法连接到多数派个节点等，则master需要执行rejoin。
    
	try {
            publishClusterState.publish(clusterChangedEvent, electMaster.minimumMasterNodes(), ackListener);
        } catch (FailedToCommitClusterStateException t) {
            // cluster service logs a WARN message
            logger.debug("failed to publish cluster state version [{}] (not enough nodes acknowledged, min master nodes [{}])",
                newState.version(), electMaster.minimumMasterNodes());

            synchronized (stateMutex) {
                pendingStatesQueue.failAllStatesAndClear(
                    new ElasticsearchException("failed to publish cluster state"));

                rejoin("zen-disco-failed-to-publish");
            }
            throw t;
        }

在对其他节点进行定期的ping时，发现有其他节点也是master，此时会比较本节点与另一个master节点的cluster_state的version，谁的version大谁成为master，version小的执行rejoin。

    if (otherClusterStateVersion > localClusterState.version()) {
            rejoin("zen-disco-discovered another master with a new cluster_state [" + otherMaster + "][" + reason + "]");
        } else {
            // TODO: do this outside mutex
            logger.warn("discovered [{}] which is also master but with an older cluster_state, telling [{}] to rejoin the cluster ([{}])", otherMaster, otherMaster, reason);
            try {
                // make sure we're connected to this node (connect to node does nothing if we're already connected)
                // since the network connections are asymmetric, it may be that we received a state but have disconnected from the node
                // in the past (after a master failure, for example)
                transportService.connectToNode(otherMaster);
                transportService.sendRequest(otherMaster, DISCOVERY_REJOIN_ACTION_NAME, new RejoinClusterRequest(localClusterState.nodes().getLocalNodeId()), new EmptyTransportResponseHandler(ThreadPool.Names.SAME) {

                    @Override
                    public void handleException(TransportException exp) {
                        logger.warn((Supplier<?>) () -> new ParameterizedMessage("failed to send rejoin request to [{}]", otherMaster), exp);
                    }
                });
            } catch (Exception e) {
                logger.warn((Supplier<?>) () -> new ParameterizedMessage("failed to send rejoin request to [{}]", otherMaster), e);
            }
        }



### 集群扩缩容
上面讲了节点发现、Master选举、错误检测等机制，那么现在我们可以来看一下如何对集群进行扩缩容。

#### 扩容DataNode
假设一个ES集群存储或者计算资源不够了，我们需要进行扩容，这里我们只针对DataNode，即配置为：

	conf/elasticsearch.yml:
	    node.master: false
	    node.data: true
然后需要配置集群名、节点名等其他配置，为了让该节点能够加入集群，我们把discovery.zen.ping.unicast.hosts配置为集群中的master-eligible node。

	conf/elasticsearch.yml:
	    cluster.name: es-cluster
	    node.name: node_Z
	    discovery.zen.ping.unicast.hosts: ["x.x.x.x", "x.x.x.y", "x.x.x.z"]

然后启动节点，节点会自动加入到集群中，集群会自动进行rebalance，或者通过reroute api进行手动操作。

[https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html)

[https://www.elastic.co/guide/en/elasticsearch/reference/current/shards-allocation.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/shards-allocation.html)

#### 缩容DataNode

假设一个ES集群使用的机器数太多了，需要缩容，我们怎么安全的操作来保证数据安全，并且不影响可用性呢？

首先，我们选择需要缩容的节点，注意本节只针对DataNode的缩容，MasterNode缩容涉及到更复杂的问题，下面再讲。

然后，我们需要把这个Node上的Shards迁移到其他节点上，方法是先设置allocation规则，禁止分配Shard到要缩容的机器上，然后让集群进行rebalance。

	PUT _cluster/settings
	{
	  "transient" : {
	    "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
	  }
	}
等这个节点上的数据全部迁移完成后，节点可以安全下线。

更详细的操作方式可以参考官方文档：

[https://www.elastic.co/guide/en/elasticsearch/reference/current/allocation-filtering.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/allocation-filtering.html)

#### 扩容MasterNode

假如我们想扩容一个MasterNode(master-eligible node)， 那么有个需要考虑的问题是，上面提到为了避免脑裂，ES是采用多数派的策略，需要配置一个quorum数：

	conf/elasticsearch.yml:
	    discovery.zen.minimum_master_nodes: 2

假设之前3个master-eligible node，我们可以配置quorum为2，如果扩容到4个master-eligible node，那么quorum就要提高到3。

所以我们应该先把discovery.zen.minimum_master_nodes这个配置改成3，再扩容master，更改这个配置可以通过API的方式：

	curl -XPUT localhost:9200/_cluster/settings -d '{
	    "persistent" : {
	        "discovery.zen.minimum_master_nodes" : 3
	    }
	}

这个API发送给当前集群的master，然后新的值立即生效，然后master会把这个配置持久化到cluster meta中，之后所有节点都会以这个配置为准。

但是这种方式有个问题在于，配置文件中配置的值和cluster meta中的值很可能出现不一致，不一致很容易导致一些奇怪的问题，比如说集群重启后，在恢复cluster meta前就需要进行master选举，此时只可能拿配置中的值，拿不到cluster meta中的值，但是cluster meta恢复后，又需要以cluster meta中的值为准，这中间肯定存在一些正确性相关的边界case。

总之，动master节点以及相关的配置一定要谨慎，master配置错误很有可能导致脑裂甚至数据写坏、数据丢失等场景。

### 缩容MasterNode

缩容MasterNode与扩容跟扩容是相反的流程，我们需要先把节点缩下来，再把quorum数调下来，不再详细描述


### 与Zookeeper、raft等实现方式的比较

#### 与使用Zookeeper相比
本篇讲了ES集群中节点相关的几大功能的实现方式：

1. 节点发现
1. Master选举
1. 错误检测
1. 集群扩缩容

试想下，如果我们使用Zookeeper来实现这几个功能，会带来哪些变化？

##### Zookeeper介绍
我们首先介绍一下Zookeeper，熟悉的同学可以略过。

Zookeeper分布式服务框架是Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。

简单来说，Zookeeper就是用于管理分布式系统中的节点、配置、状态，并完成各个节点间配置和状态的同步等。大量的分布式系统依赖Zookeeper或者是类似的组件。

Zookeeper通过目录树的形式来管理数据，每个节点称为一个znode，每个znode由3部分组成:

- stat. 此为状态信息, 描述该znode的版本, 权限等信息.
- data. 与该znode关联的数据.
- children. 该znode下的子节点.

stat中有一项是ephemeralOwner，如果有值，代表是一个临时节点，临时节点会在session结束后删除，可以用来辅助应用进行master选举和错误检测。

Zookeeper提供watch功能，可以用于监听相应的事件，比如某个znode下的子节点的增减，某个znode本身的增减，某个znode的更新等。

##### 怎么使用Zookeeper实现ES的上述功能

1. 节点发现：每个节点的配置文件中配置一下Zookeeper服务器的地址，节点启动后到Zookeeper中某个目录中注册一个临时的znode。当前集群的master监听这个目录的子节点增减的事件，当发现有新节点时，将新节点加入集群。
2. master选举：当一个master-eligible node启动时，都尝试到固定位置注册一个名为master的临时znode，如果注册成功，即成为master，如果注册失败则监听这个znode的变化。当master出现故障时，由于是临时znode，会自动删除，这时集群中其他的master-eligible node就会尝试再次注册。使用Zookeeper后其实是把选master变成了抢master。
3. 错误检测：由于节点的znode和master的znode都是临时znode，如果节点故障，会与Zookeeper断开session，znode自动删除。集群的master只需要监听znode变更事件即可，如果master故障，其他的候选master则会监听到master znode被删除的事件，尝试成为新的master。
4. 集群扩缩容：扩缩容将不再需要考虑minimum_master_nodes配置的问题，会变得更容易。


##### 使用Zookeeper的优劣点
使用Zookeeper的好处是，把一些复杂的分布式一致性问题交给Zookeeper来做，ES本身的逻辑就可以简化很多，正确性也有保证，这也是大部分分布式系统实践过的路子。而ES的这套ZenDiscovery机制经历过很多次bug fix，到目前仍有一些边角的场景存在bug，而且运维也不简单。

那为什么ES不使用Zookeeper呢，大概是官方开发觉得增加Zookeeper依赖后会多依赖一个组件，使集群部署变得更复杂，用户在运维时需要多运维一个Zookeeper。

那么在自主实现这条路上，还有什么别的算法选择吗？当然有的，比如raft。

#### 与使用raft相比
raft算法是近几年很火的一个分布式一致性算法，其实现相比paxos简单，在各种分布式系统中也得到了应用。这里不再描述其算法的细节，我们单从master选举算法角度，比较一下raft与ES目前选举算法的异同点：

##### 相同点

1. 多数派原则：必须得到超过半数的选票才能成为master。
2. 选出的leader一定拥有最新已提交数据：在raft中，数据更新的节点不会给数据旧的节点投选票，而当选需要多数派的选票，则当选人一定有最新已提交数据。在es中，version大的节点排序优先级高，同样用于保证这一点。

##### 不同点

1. 正确性论证：raft是一个被论证过正确性的算法，而ES的算法是一个没有经过论证的算法，只能在实践中发现问题，做bug fix，这是我认为最大的不同。
2. 是否有选举周期term：raft引入了选举周期的概念，每轮选举term加1，保证了在同一个term下每个参与人只能投1票。ES在选举时没有term的概念，不能保证每轮每个节点只投一票。
3. 选举的倾向性：raft中只要一个节点拥有最新的已提交的数据，则有机会选举成为master。在ES中，version相同时会按照NodeId排序，总是NodeId小的人优先级高。
看法
raft从正确性上看肯定是更好的选择，而ES的选举算法经过几次bug fix也越来越像raft。当然，在ES最早开发时还没有raft，而未来ES如果继续沿着这个方向走很可能最终就变成一个raft实现。





-------------

> [Elasticsearch分布式一致性原理剖析(一)-节点篇](https://zhuanlan.zhihu.com/p/34858035) 
> 
> [Elasticsearch内核解析 - 写入篇](https://zhuanlan.zhihu.com/p/34669354)
> 
> ElasticSearch权威指南


