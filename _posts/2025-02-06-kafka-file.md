---
layout: post
title: Kafka topic的存储结构
categories: [ kafka, storage ]
description: Kafka topic 在 Broker 中的存储结构与文件作用
keywords: kafka, storage, partition, log, index, metadata
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---


# Kafka 主题 (Topic) 在 Broker 存储中的文件目录结构

Kafka 以分区 (Partition) 为单位存储数据，每个主题的分区会对应一个独立的目录，其中包含多个日志文件、索引文件和元数据文件。下面是对各个文件的详细解释：

## 目录结构

- `test-topic-0/`：这是 `test-topic` 主题的第 0 号分区的存储目录。
- `test-topic-1/`，`test-topic-2/`，`test-topic-3/`：这些是 `test-topic` 主题的其他分区目录，每个分区都存储在独立的文件夹中。

## 主要文件解释

### `00000000000000000000.log`

- 这是 Kafka 主要的日志数据文件，存储了从 `offset=0` 开始的消息记录。
- `.log` 文件是 Kafka 的存储核心，每个分区对应多个 `.log` 文件，文件名中的数字表示起始 `offset`。

### `00000000000000000000.index`

- 这是索引文件，用于加速 Kafka 在 `.log` 文件中查找特定 `offset` 的消息。
- Kafka 通过这个索引可以快速定位到 `.log` 文件中的具体偏移量（`offset`）。

### `00000000000000000000.timeindex`

- 这是时间索引文件，存储了 Kafka 消息的时间戳与 `offset` 之间的映射。
- 这个文件可以帮助 Kafka 通过时间快速查找消息，而不需要遍历整个 `.log` 文件。

### `00000000000000001120.snapshot`

- 这是快照文件，存储了 Kafka 对该分区的某些元数据的快照信息（比如已提交的 `offset`）。
- 可能用于恢复 Kafka 的分区状态，避免崩溃后重新计算。

### `leader-epoch-checkpoint`

- 记录了该分区的 `leader` 变更历史信息（`epoch`）。
- 这对于 Kafka 副本同步和分区主从关系切换至关重要。

### `partition.metadata`

- 存储该分区的元数据信息，例如分区编号、ISR（同步副本集）等。
- 这对 Kafka 的分区管理至关重要。

## 总结

- Kafka 每个 `topic` 都由多个 `partition` 组成，每个 `partition` 有一个独立的存储目录。
- `.log` 文件存储消息，`.index` 和 `.timeindex` 文件加速查找，`leader-epoch-checkpoint` 和 `partition.metadata` 记录元数据信息。
- 这些文件保证了 Kafka 的高效存储和高可用性。