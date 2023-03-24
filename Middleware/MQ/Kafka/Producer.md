# Producer几个重要配置参数

Kafka 在弹性、容错性以及高吞吐量方面有着很大的优势。想要达到生产环境最优，发挥这些特性，需要我们进行一系列的配置。

## acks

acks参数指定了必须要有多少个分区副本收到消息，生产者才认为该消息是写入成功的，这个参数对于消息是否丢失起着重要作用，该参数的配置具体如下：

- acks=0，表示生产者在成功写入消息之前不会等待任何来自服务器的响应. 换句话说，一旦出现了问题导致服务器没有收到消息，那么生产者就无从得知，消息也就丢失了. 改配置由于不需要等到服务器的响应，所以可以以网络支持的最大速度发送消息，从而达到很高的吞吐量。

- acks=1，表示只要集群的 leader 分区副本接收到了消息，就会向生产者发送一个成功响应的 ack，此时生产者接收到 ack 之后就可以认为该消息是写入成功的. 一旦消息无法写入 leader 分区副本(比如网络原因、leader 节点崩溃)，生产者会收到一个错误响应，当生产者接收到该错误响应之后，为了避免数据丢失，会重新发送数据.这种方式的吞吐量取决于使用的是异步发送还是同步发送。

    > 尖叫提示：如果生产者收到了错误响应，即便是重新发消息，还是会有可能出现丢数据的现象. 比如，如果一个没有收到消息的节点成为了新的Leader，消息就会丢失。

- acks =all (或-1)， 表示只有所有参与复制的节点 (ISR 列表的副本) 全部收到消息时，生产者才会接收到来自服务器的响应. 这种模式是最高级别的，也是最安全的，可以确保不止一个 Broker 接收到了消息. 该模式的延迟会很高。

1. **at-most-once（可能少读） ack=0  发送不管接收成功与否**
2. **at-least-once（可能重复读）ack=all(或-1) 主（leader）从（follower）分区都接收成功事务才成功**
3. **exactly-once （正好）= at-least-once+幂等性**
4. **ack=1 主分区（leader）接收成功事务就成功（kafka默认）**

## min.insync.replicas

上面提到，当 acks=all 时，需要所有的副本都同步了才会发送成功响应到生产者. 其实这里面存在一个问题：如果 Leader 副本是唯一的同步副本时会发生什么呢？此时相当于 acks=1.所以是不安全的。

Kafka 的 Broker 端提供了一个参数 `min.insync.replicas` ，该参数控制的是消息至少被写入到多少个副本才算是”真正写入”，该值默认值为1，生产环境设定为一个大于1的值可以提升消息的持久性. 因为如果同步副本的数量低于该配置值，则生产者会收到错误响应，从而确保消息不丢失。

## replica.lag.time.max.ms

In-sync replica(ISR) 称之为同步副本，ISR 中的副本都是与 Leader 进行同步的副本，所以不在该列表的 follower 会被认为与 Leader 是不同步的. 那么，ISR 中存在是什么副本呢？首先可以明确的是：Leader 副本总是存在于 ISR 中. 而 follower 副本是否在 ISR 中，取决于该 follower 副本是否与 Leader 副本保持了“同步”。

> 尖叫提示：对于 ”follower 副本是否与 Leader 副本保持了同步” 的理解如下：
> (1)上面所说的同步不是指完全的同步，即并不是说一旦 follower 副本同步滞后与 Leader 副本，就会被踢出 ISR 列表。
> (2)Kafka 的 broker 端有一个参数 `replica.lag.time.max.ms`， 该参数表示 follower 副本滞后与 Leader 副本的最长时间间隔，默认是10秒. 这就意味着，只要 follower 副本落后于 leader 副本的时间间隔不超过10秒，就可以认为该 follower 副本与 leader 副本是同步的，所以哪怕当前 follower 副本落后于 Leader 副本几条消息，只要在10秒之内赶上 Leader 副本，就不会被踢出出局。
> （3）如果follower副本被踢出ISR列表，等到该副本追上了Leader副本的进度，该副本会被再次加入到ISR列表中，所以ISR是一个动态列表，并不是静态不变的。

## retries

生产者从服务器收到的错误有可能是临时性的错误（比如分区找不到首领）。在这种情况下， retries参数的值决定了生产者可以重发消息的次数，如果达到这个次数，生产者会放弃重试并返回错误。默认情况下，生产者会在每次重试之间等待100ms ，可以通过retry.backoff.ms 参数来配置时间间隔。

比如，设置了acks=all和min.insync.replicas=2。由于某种原因，所有follower都挂了，由于min.insync.replicas=2，所以生产者无法收到来自Broker端的ack。

此时我们会从Producer端收到一个错误消息：`"Broker: Not enough in-sync replicas"`。这就意味着Kafka不能在Broker上追加生产的消息(数据)了，因为此时的ISR的数量不够。此时在Broker端会有如下的错误消息：

org.apache.kafka.common.errors.NotEnoughReplicasException: The size of the current ISR Set(0) is insufficient to satisfy the min.isr requirement of 2 for partition

默认情况下，Producer不会对此错误进行处理，这就会造成消息丢失，即*_at-most-once*_语义。我们可以通过配置重试次数来让生产者重新发送消息。比如配置retries=3，默认为0

## enable.idempotence

在某些情况下，实际上已将消息提交给了所有同步副本，但是由于网络问题，Broker无法向Producer发送确认ack。由于我们设置`retries=3`，所以producer将重新发送消息3次，这可能会导致topic中消息重复。

比如有一个producer向该topic发送1M消息，并且在提交消息之后但在生产者收到所有确认ack之前，broker失败了。在这种情况下，由于重试机制，最终可能在该topic上收到超过1M的消息，这也称为at-lease-once语义。

当然，我们想要实现的是 exactly-once 语义，即：即便生产者重新发送消息，消费者也应该只收到一次相同的消息。

此时需要进行幂等操作，所谓幂等，即指一次执行一个操作或多次执行一个操作具有相同的效果。配置幂等很简单，通过配置 `enable.idempotence=true` 即可，默认为 false。

那么，幂等是如何实现的呢？由于消息是分 batch(批次)发送的，每个 batch(批次)都有一个序列号。在 Broker 端，会追踪每个分区的最大序列号。如果出现序列号较小或相等的 batch(批次)，broker 将不会将该 batch(批次)写入 topic。这样，除了保证了幂等性，还可以确保 batch(批次)的顺序。

## max.in.flight.requests.per.connection

该参数指定了生产者在收到服务器晌应之前可以发送多少个消息。它的值越高，就会占用越多的内存，不过也会提升吞吐量。把它设为1可以保证消息是按照发送的顺序写入服务器的，即使发生了重试。

因为如果将两个批次发送到单个分区，并且第一个批次失败并被重试，但是，接着第二个批次写入成功，则第二个批次中的记录可能会首先出现，这样就会发生乱序。

如果没有启用幂等功能，但仍然希望按顺序发送消息，则应将此设置配置为1。但是，如果已经启用了幂等，则无需显式定义此配置。

## buffer.memory

该参数用来设置生产者内存缓冲区的大小，生产者用它缓冲要发送到服务器的消息。如果应用程序发送消息的速度超过发送到服务器的速度，会导致生产者空间不足。这个时候，send() 方法调用要么被阻塞，要么抛出异常，取决于如何设置 max.block.ms。

当生产者调用时 `send()`，消息并不会立即发送，而是会添加到内部缓冲区中。默认 `buffer.memory` 值为32MB。如果生产者发送消息的速度超过了将消息发送到 broker 的速度，或者存在网络问题，send() 方法调用会被阻塞 max.block.ms 参数配置的时常，默认1分钟。

## max.block.ms

该参数指定了在调用 send() 方法或使用 partitionsFor() 方法获取元数据时生产者的阻塞时间。当生产者的发送缓冲区已满，或者没有可用的元数据时，这些方法就会被阻塞。在阻塞时间达到 max.block.ms 时，生产者会抛出超时异常。

## linger.ms

该参数指定了生产者在发送批次之前等待更多消息加入批次的时间。kafka 生产者会在批次填满或 linger.ms 达到上限时把批次发送出去。默认情况下，只要有可用的线程，生产者就会把消息发送出去，就算批次里只有一个消息。把 linger.ms 设置成比 0 大的数，让生产者在发送批次之前等待一会儿，使更多的消息加入到这个批次。虽然这样会增加延迟，但也会提升吞吐量（因为一次性发送更多的消息，每个消息的开销就变小了）。

## batch.size

当有多个消息需要被发送到同一个分区时，生产者会把它们放在同一个批次里。该参数指定了一个批次可以使用的内存大小，按照字节数计算（而不是消息个数）。当批次被填满，批次里的所有消息会被发送出去。不过生产者井不一定都会等到批次被填满才发送，这取决于 linger.ms 的配置，比如如果 linger.ms 时间到了，即便批次只包含一个消息，也会被立即发送。所以就算把批次大小设置得很大，也不会造成延迟，只是会占用更多的内存而已。但如果设置得太小，因为生产者需要更频繁地发送消息，会增加一些额外的开销。

可以使用配置使用 linger.ms 和 batch.size。linger.ms 是准备好发送批次之前的延迟时间，默认值为0。这意味着即使批次中只有1条消息，批次也会立即发送。有时，会增加 linger.ms 以减少请求数量并提高吞吐量。但这将导致更多消息保留在内存中。batch.size 是单个批次的最大大小，当满足这两个要求中的任何一个时，将发送批次。

## compression.type

**默认**情况下，消息发送时**不会被压缩**。该参数可以设置为 snappy 、gzip 或 lz4 ，它指定了消息被发送给 broker 之前使用哪一种压缩算也进行压缩。使用压缩可以**降低网络传输开销**和**存储开销**，而这**往往是向 Kafka 发送消息的瓶颈**所在。