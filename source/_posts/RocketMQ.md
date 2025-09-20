---
title: RabbitMQ基本介绍
date: 2019-06-15 20:06:20
tags:
- 消息队列
categories:
- 消息队列
---


## RabbitMQ基本概念

### 生产者和消费者

![](https://i.imgur.com/mBAfQhI.png)

Producer：生产者，投递消息的一方。

Consumer: 消费者，就是接收消息的一方。

Broker: 消息中间件的服务节点 。

### 队列

Queue: 队列，是RabbitMQ 的内部对象，用于存储消息。

RabbitMQ中消息都只能存储在队列中。

多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊 CRound-Robin ，即轮询)
给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。


RabbitMQ 不支持队列层面的广播消费。

### 交换器、路由键与绑定

Exchange: 交换器。生产者将消息发送到 Exchange (交换器，通常也
可以用大写的 "X" 来表示)，由交换器将消息路由到一个或者多个队列中。如果路由不到，或许会返回给生产者，或许直接丢弃。

RoutingKey: 路由键 。生产者将消息发给交换器 的时候，一般会指定一个RoutingKey ，用来指定这个消息的路由规则，而这个 RoutingKey 需要与交换器类型和绑定键 (BindingKey) 联
合使用才能最终生效。

Binding: 绑定。RabbitMQ 中通过绑定将交换器与队列关联起来，在绑定的时候一般会指定一个绑定键 ( BindingKey ) ，这样 RabbitMQ 就知道如何正确地将消息路由到队列了。

![](https://i.imgur.com/MN0SEP6.png)

在交换器类型和绑定键 (BindingKey) 固定的情况下，生产者可以在发送消息给交换器时，通过指定RoutingKey来决定消息流向哪里。


### 交换器类型

RabbitMQ 常用的交换器类型有 fanout 、 direct 、 topic 、 headers 这四种 。 

#### fanout
它会把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中。
#### direct
direct 类型的交换器路由规则也很简单，它会把消息路由到那些 BindingKey 和 RoutingKey
完全匹配的队列中。

#### topic

topic 类型的交换器在匹配规则上进行了扩展，它与 direct 类型的交换器相似，也是将消息路由到 BindingKey 和 RoutingKey 相匹配的队列中，但这里的匹配规则有些不同，它约定:

* RoutingKey 为一个点号"."分隔的字符串(被点号"."分隔开的每一段独立的字符串称为一个单词)，如com.rabbitmq.client
* BindingKey 和 RoutingKey 一样也是点号"."分隔的字符串;
* BindingKey 中可以存在两种特殊字符串"*"和"#"，用于做模糊匹配，其中"*"用于匹配一个单词，"#"用于匹配多规格单词(可以是零个)。

#### headers

headers 类型的交换器不依赖于路由键的匹配规则来路由消息，而是根据发送的消息内容中
的 headers 属性进行匹配。在绑定队列和交换器时制定一组键值对 ， 当发送消息到交换器时，
RabbitMQ 会获取到该消息的 headers (也是一个键值对的形式) ，对比其中的键值对是否完全
匹配队列和交换器绑定时指定的键值对，如果完全匹配则消息会路由到该队列，否则不会路由
到该队列 。 headers 类型的交换器性能会很差，而且也不实用，基本上不会看到它的存在。

### RabbitMQ 运转流程

生产者发送消息：

1. 生产者连接到 RabbitMQ Broker，建立一个连接(Connection),开启一个信道 (Channel)
2. 生产者声明一个交换器 ，并设置相关属性，比如 交换机类型、是否持久化等。
3. 生产者声明 一个队列井设置相关属性，比如是否排他、是否持久化、是否自动删除等。
4. 生产者通过路由键将交换器和队列绑定起来。
5. 生产者发送消息至 RabbitMQ Broker，其中包含路由键、交换器等信息。
6. 相应的交换器根据接收到的路由键查找相匹配的队列 。
7. 如果找到，则将从生产者发送过来的消息存入相应的队列中。
8. 如果没有找到，则根据生产者配置的属性选择丢弃还是回退给生产者。
9. 关闭信道。
10. 关闭连接。

消费者接收消息的过程:

1. 消费者连接到 RabbitMQ Broker，建立一个连接 (Connection ) ，开启 一个信道 (Channel) 。
2. 消费者向 RabbitMQ Broker 请求消费相应队列中的消息，可能会设置相应的回调函数，
以及做一些准备工作(详细内容请参考 3 .4节〉。
3. 等待 RabbitMQ Broker 回应并投递相应队列中的消息， 消费者接收消息。
4. 消费者确认 ( ack) 接收到的消息 。
5. RabbitMQ 从队列中删除相应己经被确认的消息 。
6. 关闭信道。
7. 关闭连接。



## RabbitMQ进阶

### 消息特殊情况
mandatory 和 immediate 是 channel . basicPublish 方法中的两个参数，它们都有
当消息传递过程中不可达目的地时将消息返回给生产者的功能。 RabbitMQ 提供的备份交换器
(Altemate Exchange) 可以将未能被交换器路由的消息(没有绑定队列或者没有匹配的绑定〉存
储起来，而不用返回给客户端。

#### mandatory参数

当 mandatory 参数设为 true 时，交换器无法根据自身的类型和路由键找到一个符合条件
的队列，那么 RabbitMQ 会调用 Basic.Return 命令将消息返回给生产者 。当 mandatory 参
数设置为 false 时，出现上述情形，则消息直接被丢弃 。

那么生产者如何获取到没有被正确路由到合适队列的消息呢?这时候可以通过调用
channel.addReturnListener 来添加 ReturnListener 监昕器实现。

![](https://i.imgur.com/AxjpPgI.png)

#### immediate参数

当 imrnediate 参数设为 true 时，如果交换器在将消息路由到队列时发现队列上并不存在
任何消费者，那么这条消息将不会存入队列中。当与路由键匹配的所有队列都没有消费者时 ，
该消息会通过 Basic.Return 返回至生产者。

RabbitMQ3.0版本开始去掉了对 imrnediate 参数的支持，对此 RabbitMQ 官方解释是:imrnediate 参数会影响镜像队列的性能，增加了代码复杂性，建议采用TTL和DLX 的方法替代。

#### 备份交换器

备份交换器，英文名称为 Altemate Exchange，简称庙，或者更直白地称之为"备胎交换器"。
生产者在发送消息的时候如果不设置 mandatory 参数 ， 那么消息在未被路由的情况下将会丢失 :
如果设置了 mandatory 参数，那么需要添加 ReturnListener 的编程逻辑，生产者的代码将
变得复杂。如果既不想复杂化生产者的编程逻辑，又不想消息丢失，那么可以使用备份交换器，
这样可以将未被路由的消息存储在 RabbitMQ 中，再在需要的时候去处理这些消息。

可以通过在声明交换器(调用channel.exchangeDeclare方法)的时候添加alternate-exchange 参数来实现，也可以通过策略的方式实现。如果两者同时使用，则前者的优先级更高，会覆盖掉 Policy 的设置 。


![](https://i.imgur.com/2EGA5BE.png)


### TTL

#### 设置消息的 TTL
目前有两种方法可以设置消息的 TTL。第一种方法是通过队列属性设置，队列中所有消息都有相同的过期时间。第二种方法是对消息本身进行单独设置，每条消息的 TTL 可以不同。如果两种方法一起使用，则消息的 TTL 以两者之间较小的那个数值为准。消息在队列中的生存时间一旦超过设置 的 TTL 值时，就会变成"死信" (Dead Message) ，消费者将无法再收到该消息。

* 通过队列属性设置消息 TTL 的方法是在 channel.queueDeclare 方法中加入
x-message -ttl 参数实现的，这个参数的单位是毫秒。

		Map<String, Object> argss = new HashMap<String , Object>();
		argss.put("x-message-ttl " , 6000);
		channel.queueDeclare(queueName , durable , exclusive , autoDelete , argss) ;
		//同时也可以通过 Policy 的方式来设置 TTL.示例如下 :
		rabbitmqctl set_policy TTL "食" '{"message-ttl":60000}' --apply-to queues
		//还可以通过调用 HTTPAPI 接口设置 :
		$ curl -i -u root:root -H "content-type:application/json"-X PUT
		-d'{"auto_delete":false, "durable":true, "arguments":{"x-message-ttl": 60000}}'
		http://localhost:15672/api/queues/{vhost}/{queuename}

* 针对每条消息设置 TTL 的方法是在 channel.basicPublish 方法中加入 expiration
的属性参数，单位为毫秒。

		AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties . Builder();
		builder.deliveryMode(2); / / 持久化消息
		builder.expiration( " 60000 " );/ / 设置 TTL=60000ms
		AMQP.BasicProperties properties = builder.build() ;
		channel.basicPublish(exchangeName, routingKey, mandatory, properties ,
		"ttlTestMessage".getBytes());

如果不设置 TTL.则表示此消息不会过期 ;如果将 TTL 设置为 0，则表示除非此时可以直接将消息投递到消费者，否则该消息会被立即丢弃，这个特性可以部分替代 RabbitMQ 3.0 版本之前的 immediate 参数，之所以部分代替，是因为immediate 参数在投递失败时会用Basic.Return 将消息返回。


对于第一种设置队列 TTL 属性的方法，一旦消息过期，就会从队列中抹去，而在第二种方
法中，即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费
者之前判定的。

为什么这两种方法处理的方式不一样?因为第一种方法里，队列中己过期的消息肯定在队
列头部， RabbitMQ 只要定期从队头开始扫描是否有过期的消息即可。而第二种方法里，每条消
息的过期时间不同，如果要删除所有过期消息势必要扫描整个队列，所以不如等到此消息即将
被消费时再判定是否过期 ， 如果过期再进行删除即可。

#### 设置队列的TTL

通过 channel.queueDeclare 方法中的 x-expires 参数可以控制队列被自动删除前处于未使用状态的时间。未使用的意思是队列上没有任何的消费者，队列也没有被重新声明，并且在过期时间段内也未调用过Basic.Get命令。

abb itMQ 会确保在过期时间到达后将队列删除，但是不保障删除的动作有多及时 。在
RabbitMQ 重启后 ， 持久化的队列的过期时间会被重新计算。
用于表示过期时间的 x-expires 参数以毫秒为单位 ， 井且服从和 x-message-ttl 一样
的约束条件，不过不能设置为 0。比如该参数设置为 1 000 ，则表示该队列如果在 1 秒钟之内未
使用则会被删除。

### 死信队列

DLX ，全称为 Dead-Letter-Exchange ，可以称之为死信交换器，也有人称之为死信邮箱。当
消息在一个队列中变成死信 (dead message) 之后，它能被重新被发送到另一个交换器中，这个
交换器就是 DLX，绑定 DLX 的队列就称之为死信队列。

DLX 也是一个正常的交换器，和一般的交换器没有区别，它能在任何的队列上被指定，实际上就是设置某个队列的属性。当这个队列中存在死信时，RabbitMQ 就会自动地将这个消息重新发布到设置的 DLX 上去 ，进而被路由到另一个队列，即死信队列。可以监听这个队列中的消息、以进行相应的处理，这个特性与将消息的 TTL 设置为 0 配合使用可以弥补imrnediate参数功能。

### 延迟队列

延迟队列存储的对象是对应的延迟消息，所谓"延迟消息"是指当消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费 。

在图 4-4 中，不仅展示的是死信队列的用法，也是延迟队列的用法，对于 queue.dlx 这个死信队列来说，同样可以看作延迟队列。假设一个应用中需要将每条消息都设置为 10 秒的延迟，生产者通过exchange.normal 这个交换器将发送的消息存储在 queue.normal 这个队列中。消费者订阅的并非是 queue.normal 这个队列，而是 queue.dlx 这个队列 。当消息从 queue.normal 这个队列中过期之后被存入 queue.dlx 这个队列中，消费者就恰巧消费到了延迟 10 秒的这条消息 。

![](https://i.imgur.com/EksuM5p.png)


### 优先队列

优先级队列，顾名思义，具有高优先级的队列具有高的优先权，优先级高的消息具备优先
被消费的特权。可以通过设置队列的 x-max-priority 参数来实现。


### RPC实现

 RPC 的主要功用是让构建分布式计算更容易，在提供强大的远程调用能力时不损失本地调用的语义简洁性。

一般在 RabbitMQ 中进行 RPC 是很简单。客户端发送请求消息，服务端回复响应的消息 。为了接收响应的消息，我们需要在请求消息中发送一个回调队列。

这里就用到两个属性。

* replyTo: 通常用来设置一个回调队列。
* correlationId : 用来关联请求(request) 和其调用RPC之后的回复(response ) 。


RPC 的处理流程如下 :

1. 当客户端启动时，创建一个匿名的回调队列(名称由 RabbitMQ 自动创建，图 4-7 中
的回调队列为 amq.gen-LhQzlgv3GhDOv8PIDabOXA 。
2. 客户端为 RPC 请求设置 2 个属性 : reply T o 用来告知 RPC 服务端回复请求时的目的
队列，即回调队列; correlationld 用来标记一个请求。
3. 请求被发送到 rpc_queue 队列中。
4. RPC 服务端监听 rpc_queue 队列中的请求，当请求到来时 ， 服务端会处理并且把带有
结果的消息发送给客户端。 接收的队列就是 replyTo 设定 的 回调队列。
5. 客户端监昕回调队列，当有消息时 ， 检查 correlationld 属性，如果与请求匹配，
那就是结果了。


### 持久化

RabbitMQ的持久化分为三个部分:交换器的持久化、队列的持久化和消息的持久化 。

交换器的持久化是通过在声明队列是将 durable 参数置为 true 实现的。

队列的持久化是通过在声明队列时将 durable 参数置为 true 实现的。

队列的持久化能保证其本身的元数据不会因异常情况而丢失，但是并不能保证内部所存储的
消息不会丢失。要确保消息不会丢失 ， 需要将其设置为持久化。通过将消息的投递模式
(BasicProperties 中的deliveryMode 属性)设置为 2 即可实现消息的持久化。


### 生产者确认

消息的生产者将消息发送出去之后，消息到底有没有正确地到达服务器呢?如果不进行特殊配置，默认情况下发送消息的操作是不会返回任何信息给生产者的，也就是默认情况下生产者是不知道消息有没有正确地到达服务器。如果在消息到达服务器之前己经丢失，持久化操作也解决不了这个问题，因为消息根本没有到达
服务器 ，何谈持久化?

RabbitMQ 针对这个问题，提供了两种解决方式:
* 通过事务机制实现
* 通过发送方确认publisher confirm  机制实现。

#### 事务机制

Rabb itMQ 客户端中与事务机制相关的方法有 三 个: channel.txSelect 、channel .txCommit 和 channel.txRollbacko。
channel.txSelect 用于将当前的信道设置成事务模式。
channel.txCommit 用于提交事务。
channel.txRollback 用于事务回滚。

在通过 channel.txSelect 方法开启事务之后，我们便可以发布消息给 RabbitMQ 了，如果事务提交成功，则消息一定到达了 RabbitMQ 中，如果在事务提交执行之前由于 RabbitMQ异常崩溃或者其他原因抛出异常，这个时候我们便可以将其捕获，进而通过执行channel.txRollback 方法来实现事务回夜。注意这里的 RabbitMQ 中的事务机制与大多数数据库中的事务概念井不相同，需要注意区分。

务确实能够解决消息发送方和 RabbitMQ 之间消息确认的问题，只有消息成功被
RabbitMQ 接收，事务才能提交成功，否则便可在捕获异常之后进行事务回滚 ，与此同时可以进
行消息重发。但是使用事务机制会"吸干" RabbitMQ 的性能。

#### 发送方确认机制


生产者将信道设置成 confirmn （确认)模式，一旦信道进入 confmn 模式，所有在该信道上面发布的消息都会被指派一个唯一的 ID（从1开始)，一旦消息被投递到所有匹配的队列之后，RabbitMQ 就会发送一个确认 （Basic.Ack) 给生产者(包含消息的唯一 ID) ，这就使得生产者知晓消息已经正确到达了目的地了。如果消息和队列是可持久化的，那么确认消息会在消息写入磁盘之后发出。 RabbitMQ 回传给生产者的确认消息中的 deliveryTag 包含了确认消息的序号，此外 RabbitMQ 也可以设置 channel.basicAck 方法中的multiple 参数，表示到这个序号之前的所有消息都己经得到了处理。

![](https://i.imgur.com/alg8Te7.png)


发送方确认机制最大的好处在于它是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用程序便可以通过回调方法来处理该确认消息，如果 RabbitMQ 因为自身内部错误导致消息丢失，就会发送一条 nack（Basic.Nack) 命令，生产者应用程序同样可以在回调方法中处理该 nack 命令。




> RabbitMQ实战指南