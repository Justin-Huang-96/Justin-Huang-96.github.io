---
layout: post
title: kafka-base-consumer
categories: [ kafka, consumer ]
description: some word here
keywords: keyword1, keyword2
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 设计亮点

## 动态线程管理：

采用 ExecutorService 创建动态线程池 (CachedThreadPool)，根据需要随时创建和销毁线程，避免了固定线程池资源浪费或不足的问题。
每个Kafka分区独立维护一个线程和队列，确保分区之间的并行处理能力，提升了吞吐量和消费效率。

## 分区隔离和动态扩展：

使用 ConcurrentHashMap 维护分区的消息队列和线程任务，动态为不同的分区创建线程。
分区线程在无新消息时自动关闭，减少空闲线程占用资源。

## 线程空闲回收机制：

如果分区线程在规定时间内（IDLE_TIMEOUT_MS）没有处理到消息，则主动关闭线程并移除分区队列，避免线程长期空闲占用资源。

## 线程池监控机制：

定期通过调度任务 (ScheduledExecutorService) 监控线程池状态，打印当前活跃线程数和队列大小，便于系统运行状态观测和问题排查。

## 事件监听与自动关闭：

监听 ConsumerStoppedEvent，当Kafka消费者停止时，自动触发关闭流程，确保线程池和资源及时释放，防止资源泄漏。

## 灵活的消息处理机制：

采用抽象方法 process()子类实现自定义的Kafka消息处理逻辑，确保代码具备良好的扩展性，适用于不同业务场景。


# 后续改进思路
线程管理优化（避免线程膨胀）：

问题： 如果分区数量较多或消费高峰期，可能会导致线程池中线程数量快速膨胀，造成内存和CPU压力。
改进建议：
引入线程数量上限控制，限制分区线程数量（例如，使用 ThreadPoolExecutor 设置最大线程数），防止无限制线程增长。
或者使用分区批量消费方式，在一个线程中处理多个分区的消息，降低线程数量。
任务取消机制优化：

问题： 线程关闭时，Future 仅从 partitionFutures 中移除，未调用 cancel 方法中断线程。
改进建议： 在分区空闲或手动关闭时，通过 future.cancel(true) 主动中断线程执行，避免不必要的线程阻塞。
消息丢失问题处理：

问题： 如果分区队列已满，超出的消息会直接丢弃，可能造成数据丢失。
改进建议：
增加丢弃消息的告警或持久化机制，例如将丢弃的消息写入日志或外部存储，供后续补偿处理。
或者引入队列扩容机制，动态调整分区队列大小以适应高峰流量。
负载均衡和分区重分配：

问题： 当前每个分区独立维护线程，如果分区负载不均匀，可能导致某些线程非常繁忙，而其他线程空闲。
改进建议： 引入分区动态分配机制，将不同分区的消息分配到空闲线程中，平衡负载。例如，可以在多个分区间共享一个线程池。
持久化未处理消息：

问题： 当系统关闭时，分区队列中未处理的消息可能丢失。
改进建议： 在系统关闭前，将分区队列中未消费的消息存入磁盘或Redis等持久化存储中，待系统恢复后重新处理，提升系统的可靠性。
线程健康检查与重启机制：

问题： 如果某个分区的线程意外崩溃或抛出异常，可能导致分区消息积压而无法恢复处理。
改进建议： 引入健康检查机制，如果线程意外终止，自动重启消费者线程，保障分区消息持续消费。
监控和报警机制：

增强监控功能，在线程或分区长期积压时，触发告警或自动扩容机制。
引入Prometheus或类似工具，暴露线程池状态、消息消费延迟等指标，便于实时监控系统运行状况。

# Base-Consumer代码示例(自动ack版本)

```java

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationListener;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.kafka.event.ConsumerStoppedEvent;

import javax.annotation.PostConstruct;
import java.lang.reflect.Field;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.*;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * BaseConsumer 类是Kafka消费者的基础实现，支持动态创建和关闭线程，处理不同分区的数据。
 * 它确保在分区长时间空闲时关闭对应线程，从而提高系统资源利用率。
 */
public abstract class BaseConsumer implements ApplicationListener<ConsumerStoppedEvent> {
    private static final Logger logger = LoggerFactory.getLogger(BaseConsumer.class);

    private volatile ExecutorService executorService;
    private static final int QUEUE_CAPACITY = 1000;
    private static final long IDLE_TIMEOUT_MS = 5 * 60 * 1000; // 5分钟无消息则关闭线程

    // 存储每个分区对应的消息队列
    private final ConcurrentHashMap<Integer, BlockingQueue<ConsumerRecord<String, String>>> partitionQueues = new ConcurrentHashMap<>();
    // 记录每个分区对应的线程任务，便于线程管理和关闭
    private final ConcurrentHashMap<Integer, Future<?>> partitionFutures = new ConcurrentHashMap<>();
    private final Lock lock = new ReentrantLock();

    private final String topicName;
    @Autowired
    private KafkaAdmin kafkaAdmin;

    public BaseConsumer(String topicName) {
        this.topicName = topicName;
    }

    @PostConstruct
    private void init() {
       /* Properties adminConfig = getKafkaAdminProperties();
        int partitionCount;
        try (AdminClient adminClient = AdminClient.create(adminConfig)) {
            // 获取Kafka中指定topic的分区数量
            ListTopicsResult topics = adminClient.listTopics();
            if (topics.names().get().contains(topicName)) {
                Map<String, TopicDescription> descriptions =
                        adminClient.describeTopics(java.util.Collections.singletonList(topicName)).all().get();
                TopicDescription description = descriptions.get(topicName);
                partitionCount = description.partitions().size();
            } else {
                logger.warn("Topic {} not found. 使用默认分区数量1", topicName);
                partitionCount = 1;
            }
        } catch (Exception e) {
            logger.error("获取分区数量失败，使用默认分区数量1。Topic: {}", topicName, e);
            partitionCount = 1;
        }*/

        // 创建动态线程池，根据需要随时创建和销毁线程
        this.executorService = Executors.newCachedThreadPool(runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName(topicName + "-Consumer-Thread-" + thread.getId());
            thread.setDaemon(true); // 守护线程，确保不会阻止应用程序关闭
            return thread;
        });
        logger.info("线程池初始化完成，Topic: {}", topicName);
        startThreadPoolMonitor();
    }

    // 获取KafkaAdmin的配置属性，确保能够创建AdminClient管理Kafka
    private Properties getKafkaAdminProperties() {
        try {
            Field configField = KafkaAdmin.class.getDeclaredField("configs");
            configField.setAccessible(true);
            @SuppressWarnings("unchecked")
            Map<String, Object> configMap = (Map<String, Object>) configField.get(kafkaAdmin);

            Properties properties = new Properties();
            properties.putAll(configMap);
            return properties;
        } catch (Exception e) {
            logger.error("获取KafkaAdmin配置失败，使用空配置。", e);
            return new Properties();
        }
    }

    // 抽象方法，子类需实现具体的消息处理逻辑
    public abstract void process(ConsumerRecord<String, String> record);

    /**
     * 启动分区线程并处理消息
     */
    public void start(ConsumerRecord<String, String> record) {
        int partition = record.partition();
        lock.lock();
        try {
            // 如果该分区还没有对应的线程和队列，则创建新队列并启动线程
            BlockingQueue<ConsumerRecord<String, String>> queue = partitionQueues.computeIfAbsent(partition, key -> {
                BlockingQueue<ConsumerRecord<String, String>> newQueue = new LinkedBlockingQueue<>(QUEUE_CAPACITY);
                startConsumerThread(partition, newQueue);
                return newQueue;
            });
            // 将消息放入分区对应的队列，队列满则丢弃消息
            if (!queue.offer(record)) {
                logger.warn("分区 {} 的队列已满，消息被丢弃", partition);
            }
        } finally {
            lock.unlock();
        }
    }

    /**
     * 启动消费者线程，处理分区队列中的消息
     */
    private void startConsumerThread(int partition, BlockingQueue<ConsumerRecord<String, String>> queue) {
        Future<?> future = executorService.submit(() -> {
            long lastProcessedTime = System.currentTimeMillis();
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    // 每隔1秒从队列中拉取一条消息进行处理
                    ConsumerRecord<String, String> record = queue.poll(1, TimeUnit.SECONDS);
                    if (record != null) {
                        process(record);
                        lastProcessedTime = System.currentTimeMillis(); // 更新最后处理时间
                    } else if (System.currentTimeMillis() - lastProcessedTime > IDLE_TIMEOUT_MS) {
                        // 如果队列中长时间没有新消息，停止该分区线程
                        logger.info("分区 {} 长时间空闲，关闭对应线程", partition);
                        partitionQueues.remove(partition);
                        partitionFutures.remove(partition);
                        break;
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (Exception e) {
                    logger.error("处理分区 {} 的消息时出错", partition, e);
                }
            }
        });
        partitionFutures.put(partition, future);
    }

    /**
     * 定期监控线程池状态，打印线程池的活跃线程数和队列大小
     */
    private void startThreadPoolMonitor() {
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleAtFixedRate(() -> {
            logger.info("线程池状态 - 活跃线程: {}, 队列大小: {}",
                    ((ThreadPoolExecutor) executorService).getActiveCount(),
                    ((ThreadPoolExecutor) executorService).getQueue().size());
        }, 0, 5, TimeUnit.SECONDS);
    }

    /**
     * Kafka消费者停止事件监听器
     */
    @Override
    public void onApplicationEvent(ConsumerStoppedEvent event) {
        logger.info("Topic {} 的消费者已停止，启动关闭流程", topicName);
        shutdown();
    }

    /**
     * 关闭线程池和消费者线程
     */
    public void shutdown() {
        logger.info("关闭Topic {} 的消费者线程池...", topicName);
        if (executorService != null) {
            executorService.shutdown();
            try {
                if (!executorService.awaitTermination(10, TimeUnit.SECONDS)) {
                    executorService.shutdownNow();
                }
            } catch (InterruptedException e) {
                executorService.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
    }
}

```

# DemoConsumer示例

```java

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

/**
 * @author: Justin Huang
 * @description: 一个consumer实现类只针对一个topic
 * @date: 2025/1/7 9:54
 */
@Service
public class DemoConsumer extends BaseConsumer {

    private static final Logger logger = LoggerFactory.getLogger(DemoConsumer.class);
    public static final String TOPIC = "test-topic";

    public DemoConsumer() {
        //固定写法
        super(TOPIC);
    }


    @KafkaListener(topics = TOPIC, groupId = "test-group")
    public void listen(ConsumerRecord<String, String> record) {
        //固定写法
        start(record);
    }


    @Override
    public void process(ConsumerRecord<String, String> record) {
        //具体业务逻辑
        logger.info("Processing message with key: {}, value: {}  partition:{}", record.key(), record.value(), record.partition());
    }
}

```



# 手动Ack版本示例
## BaseConsumer
```java
import javafx.util.Pair;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.header.Headers;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationListener;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.kafka.event.ConsumerStoppedEvent;
import org.springframework.kafka.support.Acknowledgment;

import javax.annotation.PostConstruct;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.lang.reflect.Field;
import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.*;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.zip.GZIPInputStream;

/**
 * BaseConsumer 类是Kafka消费者的基础实现，支持动态创建和关闭线程，处理不同分区的数据。
 * 它确保在分区长时间空闲时关闭对应线程，从而提高系统资源利用率。
 */
public abstract class BaseConsumer implements ApplicationListener<ConsumerStoppedEvent> {
    private static final Logger logger = LoggerFactory.getLogger(BaseConsumer.class);

    private volatile ExecutorService executorService;
    private static final int QUEUE_CAPACITY = 1000;
    private static final long IDLE_TIMEOUT_MS = 5 * 60 * 1000; // 5分钟无消息则关闭线程

    // 存储每个分区对应的消息队列
    private final ConcurrentHashMap<Integer, BlockingQueue<Pair<ConsumerRecord<String, String>, Acknowledgment>>> partitionQueues = new ConcurrentHashMap<>();
    // 记录每个分区对应的线程任务，便于线程管理和关闭
    private final ConcurrentHashMap<Integer, Future<?>> partitionFutures = new ConcurrentHashMap<>();
    private final Lock lock = new ReentrantLock();

    private final String topicName;
    @Autowired
    private KafkaAdmin kafkaAdmin;

    public BaseConsumer(String topicName) {
        this.topicName = topicName;
    }

    @PostConstruct
    private void init() {
       /* Properties adminConfig = getKafkaAdminProperties();
        int partitionCount;
        try (AdminClient adminClient = AdminClient.create(adminConfig)) {
            // 获取Kafka中指定topic的分区数量
            ListTopicsResult topics = adminClient.listTopics();
            if (topics.names().get().contains(topicName)) {
                Map<String, TopicDescription> descriptions =
                        adminClient.describeTopics(java.util.Collections.singletonList(topicName)).all().get();
                TopicDescription description = descriptions.get(topicName);
                partitionCount = description.partitions().size();
            } else {
                logger.warn("Topic {} not found. 使用默认分区数量1", topicName);
                partitionCount = 1;
            }
        } catch (Exception e) {
            logger.error("获取分区数量失败，使用默认分区数量1。Topic: {}", topicName, e);
            partitionCount = 1;
        }*/

        // 创建动态线程池，根据需要随时创建和销毁线程
        this.executorService = Executors.newCachedThreadPool(runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName(topicName + "-Consumer-Thread-" + thread.getId());
            thread.setDaemon(true); // 守护线程，确保不会阻止应用程序关闭
            return thread;
        });
        logger.info("线程池初始化完成，Topic: {}", topicName);
        startThreadPoolMonitor();
    }

    // 获取KafkaAdmin的配置属性，确保能够创建AdminClient管理Kafka
    private Properties getKafkaAdminProperties() {
        try {
            Field configField = KafkaAdmin.class.getDeclaredField("configs");
            configField.setAccessible(true);
            @SuppressWarnings("unchecked")
            Map<String, Object> configMap = (Map<String, Object>) configField.get(kafkaAdmin);

            Properties properties = new Properties();
            properties.putAll(configMap);
            return properties;
        } catch (Exception e) {
            logger.error("获取KafkaAdmin配置失败，使用空配置。", e);
            return new Properties();
        }
    }

    // 抽象方法，子类需实现具体的消息处理逻辑
    public abstract void process(ConsumerRecord<String, String> record);


    /**
     * 启动分区线程并处理消息
     */
    public void start(ConsumerRecord<String, String> record, Acknowledgment ack) {

        int partition = record.partition();
        lock.lock();
        try {
            // 如果该分区还没有对应的线程和队列，则创建新队列并启动线程
            BlockingQueue<Pair<ConsumerRecord<String, String>, Acknowledgment>> pairs = partitionQueues.computeIfAbsent(partition, key -> {
                BlockingQueue<Pair<ConsumerRecord<String, String>, Acknowledgment>> newQueue = new LinkedBlockingQueue<>(QUEUE_CAPACITY);
                startConsumerThread(partition, newQueue);
                return newQueue;
            });
            // 将消息放入分区对应的队列，队列满则丢弃消息
            if (!pairs.offer(new Pair<>(record, ack))) {
                logger.warn("分区 {} 的队列已满，消息被丢弃", partition);
            }
        } finally {
            lock.unlock();
        }
    }


    /**
     * 启动消费者线程，处理分区队列中的消息
     */
    private void startConsumerThread(int partition, BlockingQueue<Pair<ConsumerRecord<String, String>, Acknowledgment>> queue) {
        Future<?> future = executorService.submit(() -> {
            long lastProcessedTime = System.currentTimeMillis();
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    // 每隔1秒从队列中拉取一条消息进行处理
                    Pair<ConsumerRecord<String, String>, Acknowledgment> pair = queue.poll(1, TimeUnit.SECONDS);
                    if (pair != null) {
                        ConsumerRecord<String, String> record = pair.getKey();
                        Acknowledgment acknowledgment = pair.getValue();
                        String decompressedValue = decompress(record);
                        ConsumerRecord<String, String> decompressedRecord = new ConsumerRecord<>(
                                record.topic(), record.partition(), record.offset(), record.key(), decompressedValue);
                        process(decompressedRecord);
                        //手动ack
                        acknowledgment.acknowledge();
                        lastProcessedTime = System.currentTimeMillis(); // 更新最后处理时间
                    } else if (System.currentTimeMillis() - lastProcessedTime > IDLE_TIMEOUT_MS) {
                        // 如果队列中长时间没有新消息，停止该分区线程
                        logger.info("分区 {} 长时间空闲，关闭对应线程", partition);
                        partitionQueues.remove(partition);
                        partitionFutures.remove(partition);
                        break;
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (Exception e) {
                    logger.error("处理分区 {} 的消息时出错", partition, e);
                }
            }
        });
        partitionFutures.put(partition, future);
    }

    private String decompress(ConsumerRecord<String, String> record) throws Exception {
        Headers headers = record.headers();
        String compressionType = headers.lastHeader("compression") != null ? new String(headers.lastHeader("compression").value(), StandardCharsets.UTF_8) : "none";
        if ("gzip".equalsIgnoreCase(compressionType)) {
            return decompressGzip(record.value());
        }
        //后续扩展多种压缩方式
        return record.value();
    }

    private String decompressGzip(String data) throws Exception {
        byte[] compressedData = Base64.getDecoder().decode(data);
        try (GZIPInputStream gis = new GZIPInputStream(new ByteArrayInputStream(compressedData));
             ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[1024];
            int len;
            while ((len = gis.read(buffer)) != -1) {
                baos.write(buffer, 0, len);
            }
            return baos.toString("UTF-8");
        }
    }

    /**
     * 定期监控线程池状态，打印线程池的活跃线程数和队列大小
     */
    private void startThreadPoolMonitor() {
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleAtFixedRate(() -> {
            logger.info("线程池状态 - 活跃线程: {}, 队列大小: {}",
                    ((ThreadPoolExecutor) executorService).getActiveCount(),
                    ((ThreadPoolExecutor) executorService).getQueue().size());
        }, 0, 5, TimeUnit.SECONDS);
    }

    /**
     * Kafka消费者停止事件监听器
     */
    @Override
    public void onApplicationEvent(ConsumerStoppedEvent event) {
        logger.info("Topic {} 的消费者已停止，启动关闭流程", topicName);
        shutdown();
    }

    /**
     * 关闭线程池和消费者线程
     */
    public void shutdown() {
        logger.info("关闭Topic {} 的消费者线程池...", topicName);
        if (executorService != null) {
            executorService.shutdown();
            try {
                if (!executorService.awaitTermination(10, TimeUnit.SECONDS)) {
                    executorService.shutdownNow();
                }
            } catch (InterruptedException e) {
                executorService.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
    }
}

```

## ConcurrentKafkaListenerContainerFactory
```java
@EnableKafka
@Configuration
public class KafkaConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(ConsumerFactory<String, String> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        // 设置 ackMode 为 MANUAL，表示手动确认
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        return factory;
    }
}
```

## DemoConsumer
```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

/**
 * @author: Justin Huang
 * @description: 一个consumer实现类只针对一个topic
 * @date: 2025/1/7 9:54
 */
@Service
public class DemoConsumer extends BaseConsumer {

    private static final Logger logger = LoggerFactory.getLogger(DemoConsumer.class);
    public static final String TOPIC = "test-topic";

    public DemoConsumer() {
        //固定写法
        super(TOPIC);
    }


    @KafkaListener(topics = TOPIC, groupId = "test-group")
    public void listen(ConsumerRecord<String, String> record, Acknowledgment acknowledgment) {
        //固定写法
        start(record, acknowledgment);
    }


    @Override
    public void process(ConsumerRecord<String, String> record) {
        //具体业务逻辑
        logger.info("Processing message with key: {}, value: {}  partition:{}", record.key(), record.value(), record.partition());
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

    }
}

```