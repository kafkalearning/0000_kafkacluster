# 0000_kafkacluster


How to build Kafka cluster



topic 配置复制因子

Partition的个数 在server.properties中配置


isr leader中保持的follow的列表 (in-sync replica)基本同步， 差的太远会从列表中踢出

producer和consumer都是基于一个分区的leader来工作的（读写都是基于leader, 有点像redis）

topic 的每个分区的leader是在哪里? metadata都存在ZK中 

leader和follower是通过follower去拉同步的数据


leader写完本地磁盘之后等待follower拉数据，放到follower本地内存。并返回给leader确认消息（提升性能）。 然后leader就会给client一个commit.则表示数据安全
leader一旦确认commit，则此数据就可以对外提供数据


Partition数据一致性
kafka中producer发送消息到broker，broker有三种返回方式，分别是
1.noack
2.leader commit成功就ack
3.leader和follower同时commit成功才返回ack
第三种方式是数据强一致性。一致性增强，但是吞吐量就会下降  --但是kafka要高吞吐量



启动
./kafka-server-start.sh –daemon config/server.properties & 


2.Kafka UI Manager
git clone https://github.com/yahoo/kafka-manager
cd kafka-manager 
./sbt clean dist
cd  kafka-manager/target/universal
unzip kafka-manager-1.3.0.8.zip 
cd kafka-manager-1.3.0.8
bin/kafka-manager

kafka monitor
java -Xms512M -Xmx512M -Xss1024K -XX:PermSize=512m -XX:MaxPermSize=1024m -cp KafkaOffsetMonitor-assembly-0.2.1.jar com.quantifind.kafka.offsetapp.OffsetGetterWeb --zk 192.168.102.132:2181,192.168.102.130:2181,192.168.102.131:2181/kafka_root --port 8086 --refresh 10.seconds --retain 7.days > stdout.log 2> stderr.log &
创建topic
./kafka-topics.sh --create --zookeeper 192.168.102.132:2181,192.168.102.130:2181,192.168.102.131:2181/kafka_root --replication-factor 3 --partitions 1 --topic test --config min.insync.replicas=2
启动生产者
./kafka-console-producer.sh --broker-list 192.168.102.132:9092,192.168.102.130:9092,192.168.102.131:9092 --request-required-acks -1 --topic test
启动消费者
./kafka-console-consumer.sh --zookeeper 192.168.102.132:2181,192.168.102.130:2181,192.168.102.131:2181/kafka_root --topic test --from-beginning  
NotEnoughReplicasException: Message are rejected since there are fewer in-sync replicas than required


consumer group
consumer balance
1个partition会分配给一个consumer, 一个consumer可以有多个partition
consumer数量变化了 则会触发一次新的rebalance


consumer的负载均衡

producer的负载均衡（分发策略）在producer函数上可以实现RR算法


Kafka源码中的Producer Record定义
2.内部数据结构：
-- Topic （名字）
-- PartitionID ( 可选)
-- Key[( 可选 )
-- Value
3.生产者记录（简称PR）的发送逻辑:
<1> 若指定Partition ID,则PR被发送至指定Partition
<2> 若未指定Partition ID,但指定了Key, PR会按照hasy(key)发送至对应Partition
<3> 若既未指定Partition ID也没指定Key，PR会按照round-robin模式发送到每个Partition
<4> 若同时指定了Partition ID和Key, PR只会发送到指定的Partition (Key不起作用，代码逻辑决定)




how to create one kafkatemplate

prepare producer config(broker information) -> ProducerFactory -> KafkaTemplate

@Bean
public ProducerFactory<Integer, String> producerFactory() {
    return new DefaultKafkaProducerFactory<>(producerConfigs());
}

@Bean
public Map<String, Object> producerConfigs() {
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    // See https://kafka.apache.org/documentation/#producerconfigs for more properties
    return props;
}

@Bean
public KafkaTemplate<Integer, String> kafkaTemplate() {
    return new KafkaTemplate<Integer, String>(producerFactory());
}

https://docs.spring.io/spring-kafka/docs/2.1.4.RELEASE/reference/htmlsingle/