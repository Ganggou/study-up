## rabbitmq
RabbitMQ 是 AMQP（高级消息队列协议）的标准实现
* 从 AMQP 协议可以看出，Queue、Exchange 和 Binding 构成了 AMQP 协议的核心
  * Producer：消息生产者，即投递消息的程序。
  * Broker：消息队列服务器实体。
    * Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列。
    * Binding：绑定，它的作用就是把 Exchange 和 Queue 按照路由规则绑定起来。
    * Queue：消息队列载体，每个消息都会被投入到一个或多个队列。
  * Consumer：消息消费者，即接受消息的程序。
* AMQP 协议中的核心思想就是生产者和消费者的解耦

Exchange 就是根据这个 RoutingKey 和当前 Exchange 所有绑定的 Binding 做匹配，如果满足匹配，就往 Exchange 所绑定的 Queue 发送消息，RoutingKey 的意义依赖于交换机的类型
* Fanout Exchange
  * Fanout Exchange 会忽略 RoutingKey 的设置，直接将 Message 广播到所有绑定的 Queue 中
* Direct Exchange
  * Direct Exchange 是 RabbitMQ 默认的 Exchange，完全根据 RoutingKey 来路由消息。设置 Exchange 和 Queue 的 Binding 时需指定 RoutingKey（一般为 Queue Name），发消息时也指定一样的 RoutingKey，消息就会被路由到对应的Queue。 
* Topic Exchange
  * Topic Exchange 和 Direct Exchange 类似，也需要通过 RoutingKey 来路由消息，区别在于Direct Exchange 对 RoutingKey 是精确匹配，而 Topic Exchange 支持模糊匹配。分别支持*和#通配符，*表示匹配一个单词，#则表示匹配没有或者多个单词 

削峰填谷
* 业务上游队列缓冲，限速发送
* 业务下游队列缓冲，限速执行，使用非自动ack
