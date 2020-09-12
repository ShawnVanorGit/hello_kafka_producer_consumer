###### 单机Kafka安装到CentOS7

```
#进入指定目录
cd /usr/local
#下载安装包
wget http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.2.0/kafka_2.12-2.2.0.tgz tar -zxvf kafka_2.12-2.2.0.tgz
#解压
tar -zxvf kafka_2.12-2.2.0.tgz
```

###### 在CentOS上启动生产者和消费者进行测试

```
#进入Kafka目录
cd /usr/local/kafka/kafka_2.12-2.2.0
#进入Zookeeper目录，启动 Kafka之前要先启动 Zookeeper
cd /usr/local/zookeeper/zookeeper-3.4.11/
sh bin/zkServer.sh start
#启动Kafka服务
bin/kafka-server-start.sh config/server.properties &
```
确认服务是否启动？输入 `jps`命令可以查看Kafka和Zookeeper进程

重新打开两个ssh窗口，分别启动生产者和消费者
```
#启动生产者(Kafka的默认端口号是9092)
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
#启动消费者(Zookeeper的默认端口号是2181)
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
```
当生产者输入信息后并回车，在消费者窗口立刻可以看到对应的消息，说明Kafka环境搭建成功！

###### 外网访问不了Kafka？`listeners`和 `advertised.listeners`傻傻分不清！

> 需要注意的是，在本地开发之前，需要保证服务器上的 `Zookeeeper` 和 `Kafka` 服务端口都已经开放，分别需要开放的端口对应是 `2181` 和 `9092`。如果是阿里云的服务器，除了防火墙外，可能还需要修改安全组访问策略配置。

接下来在本地使用Java客户端连接Kafka进行测试：
```
#配置pom文件
<dependency>  
    <groupId>org.apache.kafka</groupId>  
    <artifactId>kafka-clients</artifactId>  
    <version>0.11.0.0</version>  
</dependency>  
  
<dependency>  
    <groupId>org.slf4j</groupId>  
    <artifactId>slf4j-simple</artifactId>  
    <version>1.7.25</version>  
    <scope>compile</scope>  
</dependency>
```

```
#编写Kafka的生产者
import java.util.Properties;  
  
import org.apache.kafka.clients.producer.KafkaProducer;  
import org.apache.kafka.clients.producer.Producer;  
import org.apache.kafka.clients.producer.ProducerRecord;  
  
public class ProducerDemo {  
  
    public static void main(String[] args){  
        Properties properties = new Properties();  
        properties.put("bootstrap.servers", "120.27.233.226:9092");  
        properties.put("acks", "all");  
        properties.put("retries", 0);  
        properties.put("batch.size", 16384);  
        properties.put("linger.ms", 1);  
        properties.put("buffer.memory", 33554432);  
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");  
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");  
        Producer<String, String> producer = null;  
        try {  
            producer = new KafkaProducer<String, String>(properties);  
            for (int i = 0; i < 100; i++) {  
                String msg = "This is Message " + i;  
                producer.send(new ProducerRecord<String, String>("test", 0,null, msg));  
                System.out.println("Sent:" + msg);  
            }  
        } catch (Exception e) {  
            e.printStackTrace();  
  
        } finally {  
            producer.close();  
        }  
    }  
}
```

```
#编写Kafka的消费者
import java.util.Arrays;  
import java.util.Properties;  
  
import org.apache.kafka.clients.consumer.ConsumerRecord;  
import org.apache.kafka.clients.consumer.ConsumerRecords;  
import org.apache.kafka.clients.consumer.KafkaConsumer;  
import org.apache.kafka.common.TopicPartition;  
  
public class ConsumerDemo {  
  
    public static void main(String[] args) throws InterruptedException {  
        Properties properties = new Properties();  
        properties.put("bootstrap.servers", "120.27.233.226:9092");  
        properties.put("group.id", "test");  
        properties.put("enable.auto.commit", "true");  
        properties.put("auto.commit.interval.ms", "1000");  
        properties.put("auto.offset.reset", "earliest");  
        properties.put("session.timeout.ms", "30000");  
        properties.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");  
        properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");  
  
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<String, String>(properties);  
        kafkaConsumer.assign(Arrays.asList(new TopicPartition("test",0)));  
        while (true) {  
            ConsumerRecords<String, String> records = kafkaConsumer.poll(100);  
            for (ConsumerRecord<String, String> record : records) {  
                System.out.printf("offset = %d, value = %s", record.offset(), record.value());  
                System.out.println("=====================>");  
            }  
        }  
    }  
}
```
启动生产者就和消费者，发现一直在报 `broker` 不可用？
```
[main] WARN org.apache.kafka.clients.NetworkClient - Connection to node 0 could not be established. Broker may not be available.
```

这是因为服务器没对外网提供服务，这时需要配置`config/server.properties` 里的两个参数：`listeners`和 `advertised.listeners`

> 1. 内网连接服务的话，就只需要 listener 配置，一般用 0.0.0.0 表示不限制;
> 2. 对公网提供服务就需要配置 advertied.listener，也就是公网ip;

再去看一下本地客户端是否成功连接。到此，已成功解决本地不能访问Kafka服务的问题！

**同理，在 `docker` 中或者在阿里云主机上部署 `kafka` 集群时，也是需要用到 `advertised_listeners`。**

图片网页版请移步：[外网访问不了Kafka？listeners和advertised.listeners傻傻分不清!][https://segmentfault.com/a/1190000024434277]
