# Using the Spark Streaming and kafka to dynamically receive and analyze Big Data in Real Time


## 1.functions of this system.
the system includes these functions:
1. calculates the total payment amounts according per different product ID groups in real time in cluster.
2. saves the result in a Redis database.

## 2.Why Spark and Kafka? 
### the advantages of Spark：
1. Spark supports streaming computing framework, high throughput and fault-tolerant processing.
2. Spark supports cluster manager, which can efficiently scale computing from one to thousands of nodes.
3. Comparing Hadoop, which saves processing data on disk, Spark saves data in memory when processing data. Thus, the computational efficiency has beengreatly improved.

### the advantages of Kafka：
Kafka is a distributed publish subscribe message system with high throughput . Based on zookeeper, it has important functions in real-time computing system.
##### 
## 3.the flowchart of the system

```
graph LR
A(booking system)-->B(Kafka)
B(kafka)-->C(Spark Streaming)
C(Spark Streaming)-->D(Redis)

```

## 4. Project resource structure


```
       
├─src
│  ├─main
│  │  ├─java
│  │  │  └─cn
│  │  │      └─itcast
│  │  │          ├─createorder
│  │  │          │      PaymentInfo.java
│  │  │          │      PaymentInfoProducer.java
│  │  │          │      
│  │  │          └─util
│  │  │                  JedisUtil.java
│  │  │                  
│  │  ├─resources
│  │  │      redis.properties
│  │  │      
│  │  └─scala
│  │      └─cn
│  │          └─itcast
│  │              └─processdata
│  │                      RedisClient.scala
│  │                      StreamingProcessdata.scala
│  │                      
│  └─test
│      └─java
└─target

```


## 5. System construction:
### Spark high available cluster:
Hadoop: version 2.7.4. It is responsible for the data storage and management.

Spark: version 2.3.2. It is responsible for computing framework.

zookeeper：version 3.4.10

Linux: version CentOS_6.7

JDK: version 1.8

### kafka 
version: 2.11-2.000. It is the distributed publish subscribe message system

### Redis
version: 3.2.8. It is responsible to store the dynamic computed data

## 6. Code description
### 6.1 Booking Sytem produces the Payment Information
(PaymentInfo.java)
```

public class PaymentInfo {

    private static final long serialVersionUID = 1L;

    private String orderId;//order Id
    private String productId;//Production ID
    private long productPrice;// Price of this production 

    public PaymentInfo() {
    }
    public static long getSerialVersionUID() {
        return serialVersionUID;
    }

    public String getOrderId() {
        return orderId;
    }

    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }

    public String getProductId() {
        return productId;
    }

    public void setProductId(String productId) {
        this.productId = productId;
    }

    public long getProductPrice() {
        return productPrice;
    }

    public void setProductPrice(long productPrice) {
        this.productPrice = productPrice;
    }
    @Override
    public String toString() {
        return "PaymentInfo{" +
                "orderId='" + orderId + '\'' +
                ", productId='" + productId + '\'' +
                ", productPrice=" + productPrice +
                '}';
    }
    //produces the payment information randomly
    public String random(){
        Random r = new Random();
        this.orderId = UUID.randomUUID().toString().replaceAll("-", "");
        this.productPrice = r.nextInt(1000);
        this.productId = r.nextInt(10)+"";
        JSONObject obj = new JSONObject();
        String jsonString = obj.toJSONString(this);
        return jsonString;
    }
}

```
in the random method above, system uses the UUID to create the payment information object randomly. It includes the price and the related production Id and the order Id. 

Because the data needs to be transferred, the JSONObject class of Fastjson is used to convert the payment information object to a string in JSON format.

### 6.2 Booking Sytem sends payment information data to Kafka cluster producer
(PaymentInfoProducer.java)

```
public class PaymentInfoProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        // 1、assign the hostname and port of the Kafka cluster
        props.put("bootstrap.servers", "hadoop01:9092,hadoop02:9092,hadoop03:9092");
        // 2、Specifies to wait for an answer from all replica nodes   
        props.put("acks", "all");
        // 3、Specify the maximum number of attempts to send a message 
        props.put("retries", 0);
        // 4、Specify the batch message processing size
        props.put("batch.size", 16384);
        // 5、Specify request delay
        props.put("linger.ms", 1);
        // 6、Specify the cache memory size
        props.put("buffer.memory", 33554432);
        // 7、Set key serialization
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        // 8、Set value serialization
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        // 9、new a Producer of Kafka
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<String, String>(props);
        PaymentInfo pay = new PaymentInfo();
        while (true){
            // 10、specify the topic and send the data
            String message = pay.random();
            kafkaProducer.send(new ProducerRecord<String, String>("itcast_order",message));
            System.out.println("the data has been sent to Kafaka："+message);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```
  in these codes above, the most important codes are the 10th processing, the system will specify the topic "itcast_order" and then send the payment information messages to the specified topic in the producer of the assigned hostname and port of the Kafka cluster.
  
  the other codes are related to specify the properties of the Kafka and then to new a producer of Kafka.
  

### 6.3 Spark Streaming receives payment information data in Kafka cluster consumer and dynamically analyzes the data in real time and then stores the computing result in Redis
#### 6.3.1 receiving the data and analyzing the data in real time
(StreamingProcessdata.scala)

```
object StreamingProcessdata {

    // Total amount per product Id group
    val orderTotalKey = "totalAmountPerProductID"
    // Total amount of all payments
    val totalKey = "totalAmountTotalProducts"
    //Redis Database
    val dbIndex = 0

    def main(args: Array[String]): Unit = {
        //1、to new the Sparkconf object
        val sparkConf: SparkConf = new SparkConf()
                .setAppName("KafkaStreamingTest")
                .setMaster("local[4]")
        //2、to new the SparkContext object
        val sc = new SparkContext(sparkConf)
        sc.setLogLevel("WARN")
        //3、to new the StreamingContext object
        val ssc = new StreamingContext(sc, Seconds(3))
        //4、offset of the message will be written to the checkpoint
        ssc.checkpoint("./spark-receiver")
        //5、to setup the Kafka parameters, including to assign the related Kafka cluster
        val kafkaParams = Map("bootstrap.servers" -> "hadoop01:9092,hadoop02:9092,hadoop03:9092",
            "group.id" -> "spark-receiver")
        //6、to specify the Topic
        val topics = Set("itcast_order")
        //7、kafkaDstream will be used to receive the data in the related topic in consumer of the assigned Kafka by using the simple APIs  Createdirectstream
        val kafkaDstream: InputDStream[(String, String)] =
            KafkaUtils.createDirectStream
                    [String, String, StringDecoder, StringDecoder](ssc, kafkaParams, topics)
        //8、receiving the data and begin to parse the data
        val events: DStream[JSONObject] = kafkaDstream.flatMap(line => Some(JSON.parseObject(line._2)))
        //9、to count the total number and total amount by productId
        val orders: DStream[(String, Int, Long)] = events.map(x => (x.getString("productId"), x.getLong("productPrice"))).groupByKey().map(x => (x._1, x._2.size, x._2.reduceLeft(_ + _)))
        orders.foreachRDD(x =>
            x.foreachPartition(partition =>
                partition.foreach(x => {
                    println("productId="
                            + x._1 + " count=" + x._2 + " productPricrice=" + x._3)
                    // to receive the Redis connection
                    val jedis: Jedis = RedisClient.pool.getResource()
                    // to specify the database
                    jedis.select(dbIndex)
                    // 10 、to accumulate amout of each product
                    val sum_perid = jedis.hincrBy(orderTotalKey, x._1, x._3)
                    println("sum_perid is " +sum_perid + " productId="
                      + x._1 + " count=" + x._2 + " productPricrice=" + x._3)
                    //11、 to accumulate the total amount 
                    jedis.incrBy(totalKey, x._3)
                    //12、 disconnect the Redis
                    RedisClient.pool.returnResource(jedis)
                })
            )
        )
        ssc.start()
        ssc.awaitTermination()
    }
}
```
the structure of events in 8th above is like these: 
{"productId":"6","orderId":"0375c5a4e1724fc1877f3eca1759a1ac","productPrice":239}
{"productId":"6","orderId":"e890eaf5135544ee91dca825966f5029","productPrice":848}
{"productId":"2","orderId":"6a6be03bb746428f8a9066660586a92b","productPrice":358}

the structure of orders in 9th above is like these:

(6,2,1087)

(2,1,358)

The creation and destruction of connection objects will cost the time heavily.
Therefore, a connection object such as a connection pool is used for each RDD partition.
so in 9th above, foreachpartition for RDD in dstream is used to create a connection object for each partition in RDD, and use a connection object to write the data in a partition to the Redis. This can greatly reduces the number of connection objects created.




## 7. how to run the system and the result 
### 7.1 start up the zookeeper, the Kafka and the Redis respectively

### 7.2 install the topic
kafka-topics.sh --create \
--topic itcast_order \
--partitions 3 \
--replication-factor 1 \
--zookeeper hadoop01:2181,hadoop02:2181,hadoop03:2181

### 7.3 listen the topic

kafka-console-consumer.sh \
--from-beginning --topic itcast_order \
--bootstrap-server hadoop01:9092, hadoop02:9092, hadoop03:9092 


### 7.4 the booking system produces the data
run the PaymentInfoProducer.main(), the system will produce the payment information data like this:
the data has been sent to Kafaka：{"orderId":"c8878h898h898tg14e2jg989878676h","productID":"9","productPrice":898}
the data has been sent to Kafaka：{"orderId":"a8878h8981098tg14e2ch967676670i","productID":"1","productPrice":178}
the data has been sent to Kafaka：{"orderId":"d8811h898h823g14e2aq988988788h","productID":"4","productPrice":143}

### 7.5 list the data in the Redis
result like this in Redis:

1. hkeys totalAmountPerProductID
 1) "5"
 2) "8"
 3) "1"
 4) "9"
 5) "4"
 6) "0"
 7) "6"
 8) "7"
 9) "3"
10) "2"

2. hvals totalAmountPerProductID
 1) "4093"
 2) "6370"
 3) "9668"
 4) "3250"
 5) "4277"
 6) "4424"
 7) "8147"
 8) "5318"
 9) "9137"
10) "6556"

3. get totalAmountTotalProducts

"61240"






