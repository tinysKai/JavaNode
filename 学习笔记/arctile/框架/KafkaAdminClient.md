## KafkaAdminClient

KafkaAdminClient 不仅可以用来管理 broker、配置和 ACL（Access Control List），还可以用来管理主题。

KafkaAdminClient 继承了 org.apache.kafka.clients.admin.AdminClient 抽象类，并提供了多种方法。

- 创建主题：CreateTopicsResult createTopics(Collection newTopics)。
- 删除主题：DeleteTopicsResult deleteTopics(Collection topics)。
- 列出所有可用的主题：ListTopicsResult listTopics()。
- 查看主题的信息：DescribeTopicsResult describeTopics(Collection topicNames)。
- 查询配置信息：DescribeConfigsResult describeConfigs(Collection resources)。
- 修改配置信息：AlterConfigsResult alterConfigs(Map<ConfigResource, Config> configs)。
- 增加分区：CreatePartitionsResult createPartitions(Map<String, NewPartitions> newPartitions)。

#### 创建主题

```java
//代码清单20-2 使用KafkaAdminClient创建一个主题
String brokerList =  "localhost:9092";
String topic = "topic-admin";

Properties props = new Properties();	①
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);
props.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);
AdminClient client = AdminClient.create(props);		②

NewTopic newTopic = new NewTopic(topic, 4, (short) 1);	③
CreateTopicsResult result = client.
        createTopics(Collections.singleton(newTopic));	④
try {
    result.all().get();											⑤
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
client.close();	
```

#### 查看配置信息

```java
//代码清单20-4 describeConfigs()方法的使用示例
public static void describeTopicConfig() throws ExecutionException,
        InterruptedException {
    String brokerList =  "localhost:9092";
    String topic = "topic-admin";

    Properties props = new Properties();
    props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);
    props.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);
    AdminClient client = AdminClient.create(props);

    ConfigResource resource =
            new ConfigResource(ConfigResource.Type.TOPIC, topic);①
    DescribeConfigsResult result =
            client.describeConfigs(Collections.singleton(resource));②
    Config config = result.all().get().get(resource);③
    System.out.println(config);④
    client.close();
}
```

#### 修改配置

```java
ConfigResource resource = new ConfigResource(ConfigResource.Type.TOPIC, topic);
ConfigEntry entry = new ConfigEntry("cleanup.policy", "compact");
Config config = new Config(Collections.singleton(entry));
Map<ConfigResource, Config> configs = new HashMap<>();
configs.put(resource, config);
AlterConfigsResult result = client.alterConfigs(configs);
result.all().get();
```

#### 创建主题

```java
NewPartitions newPartitions = NewPartitions.increaseTo(5);
Map<String, NewPartitions> newPartitionsMap = new HashMap<>();
newPartitionsMap.put(topic, newPartitions);
CreatePartitionsResult result = client.createPartitions(newPartitionsMap);
result.all().get();
```



#### 合法性校验

Kafka broker 端有一个这样的参数：create.topic.policy.class.name，默认值为 null，它提供了一个入口用来验证主题创建的合法性。使用方式很简单，只需要自定义实现 org.apache.kafka.server.policy.CreateTopicPolicy 接口，比如下面示例中的 PolicyDemo。然后在 broker 端的配置文件 config/server.properties 中配置参数 create.topic.policy.class. name 的值为 org.apache.kafka.server.policy.PolicyDemo，最后启动服务。

```java
//代码清单20-5 主题合法性验证示例
public class PolicyDemo implements CreateTopicPolicy {
    public void configure(Map<String, ?> configs) {
    }

    public void close() throws Exception {
    }

    public void validate(RequestMetadata requestMetadata)
            throws PolicyViolationException {
        if (requestMetadata.numPartitions() != null || 
                requestMetadata.replicationFactor() != null) {
            if (requestMetadata.numPartitions() < 5) {
                throw new PolicyViolationException("Topic should have at " +
                        "least 5 partitions, received: "+ 
                        requestMetadata.numPartitions());
            }
            if (requestMetadata.replicationFactor() <= 1) {
                throw new PolicyViolationException("Topic should have at " +
                        "least 2 replication factor, recevied: "+ 
                        requestMetadata.replicationFactor());
            }
        }
    }
}
```

