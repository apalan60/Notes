# Kafka

## [Is Apache Kafka a Database?](https://www.confluent.io/blog/is-kafka-a-database-with-ksqldb/)

| **Characteristic**          | **Database**                                                                                                                                                     | **Kafka**                                                                                                                                                                                  |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Data Model & Storage**    | Designed to store and manage the current state of data. Supports CRUD operations with updates, deletions, and transactions.                                        | Functions as a distributed, immutable log of events. Data is append-only and retained based on time/size policies rather than explicit updates or deletions.                           |
| **Primary Purpose**         | Serves as the system of record for applications. Optimized for random access, transactional integrity, and ad-hoc queries.                                          | Enables real-time data streaming and event-driven architectures. Excels at decoupling systems, handling high-throughput event ingestion, and supporting event sourcing.              |
| **Querying & Processing**   | Supports complex, ad-hoc queries including joins, aggregations, and multi-record transactions.                                                                    | Offers SQL-like continuous querying capabilities (via ksqlDB) for real-time stream processing and stateful transformations, while underlying data remains an event log.             |
| **Data Representation**     | Represents the current state of data. Historical changes are typically not stored unless using mechanisms like change data capture.                                  | Uses an event-sourced approach by capturing every change as an event. This enables replayability and a full audit trail of how the state evolved over time.                           |
| **Use Cases**               | OLTP applications, static reporting, and transactional systems where data consistency and fast random access are key.                                               | Real-time analytics, event sourcing, change data capture, stream processing, and systems that require audit logs or the ability to replay events.                                      |
| **Key/Index Assignment**    | Keys and indexes are assigned at the table or document level, providing a single, efficient way to reference a record.                                               | Keys are assigned at the individual message level. Each message carries its own key, and the same key can appear in multiple messages across partitions.                              |
| **Fast Lookup Mechanism**   | Utilizes fast index scans to quickly retrieve one or more records based on a key or index.                                                                         | Fast lookup is inherently supported on message offset or message timestamp. Lookup by a message key is not direct since the same key may appear at different offsets.               |
| **Search Behavior**         | A key/index scan quickly returns the relevant record(s) due to efficient indexing structures.                                                                      | Searching for a key without knowing the exact offset requires scanning through a partition, as the key might occur in multiple locations across partitions.                        |

## [Main Concepts and Terminology](https://kafka.apache.org/intro#intro_concepts_and_terms)

1. Events被分類為Topic持續性的儲存
2. data消費後不會刪除，可自訂expired time
3. 數據量上升，性能恆定，所以長時間儲存是完全可以的
4. 每個 Topic 都會被拆分成多個 Partition，而這些 Partition 可以分佈在不同的 Broker 上 -> 客戶端需要讀取或寫入數據時，可以同時訪問多個 Broker，而不是集中在單一服務器上，從而達到負載均衡
5. 相同Event key的events會依序被append到同一個partition，保證消費者以**與寫入事件相同的順序**讀取該partion的events
6. 可執行複製，沒複製的情況下event只會被儲存到某一個partition

## Topic Description

```bash
/opt/kafka/bin $ ./kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092

Topic: quickstart-events        TopicId: 7G3V-GPZQjyVmSsIyLgKrQ PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: quickstart-events        Partition: 0    Leader: 1       Replicas: 1     Isr: 1  Elr:    LastKnownElr:
```
- Topic Detail

| Field             | Value                    | Description                                                      |
|-------------------|--------------------------|------------------------------------------------------------------|
| Topic             | quickstart-events        | Topic 名稱                                                       |
| TopicId           | 7G3V-GPZQjyVmSsIyLgKrQ   | Topic 的唯一識別碼                                               |
| PartitionCount    | 1                        | 分區數量                                                         |
| ReplicationFactor | 1                        | 複製因子。代表只有一份副本（無備援）                              |
| Configs           | segment.bytes=1073741824 | 配置項，設定每個 log segment 的最大尺寸（1GB）                     |

- Partition Detail

| Field        | Value             | Description                                                        |
|--------------|-------------------|--------------------------------------------------------------------|
| Topic        | quickstart-events | Partition 所屬的 Topic                      |
| Partition    | 0                 | Partition 編號                                                     |
| Leader       | 1                 | 該 partition 的 Leader broker 編號                                  |
| Replicas     | [1]               | 存有該 partition 副本的 broker 列表（僅 broker 1）                     |
| Isr          | [1]               | In-Sync Replicas，表示與 Leader 保持同步的 broker 列表（僅 broker 1）   |
| Elr          |                   | End Log Replica                              |
| LastKnownElr |                   | 上一次已知的 End Log Replica                 |


## [Kafka Stream Architecture](https://kafka.apache.org/31/documentation/streams/architecture)

> Steams 是什麼?
> 這段[影片](https://youtu.be/QkdkLdMBuL0?si=uvPH6tRqkLmG2329)提出了一段易懂的解釋
> 
> "Think of a **Stream** as a continous real-time flow of records in key-value paris format，you don't need to explicitly new reocords, just recevie them"

- Kafka 將messages區分成多個topic partition，進行**儲存**以及**傳輸**
- 另外在API也提供了stateful operations (e.g. counts,  aversges, sum, joins...etc)
- Kafka Stream也會區分多個partition，並且會mapping到Kafka topic partition，進行資料處理
- 在Kafka Stream裡的data record會mapping到Kafka裡的messages，兩者之間透過一個*Key*進行對應，也是因此才知道哪個topic要路由到哪個Stream partition

### [Stream Partitions and Tasks](https://kafka.apache.org/31/documentation/streams/architecture#streams_architecture_tasks)

- kafka messages 會進行分區(partitioning)後儲存，根據這個topic partitions的數量，Kafka Stream 會建立對應數量的tasks。進行平行處理
- 假設一個topic有5個partition，就會有5個stream tasks，接著分派給**最多**n個application instances 來運行處理流程（processor topology）
- Kafka Streams 使用 [StreamsPartitionAssignor](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/processor/internals/StreamsPartitionAssignor.java) 這個類別來分配 topic partitions 給tasks，以在多個instances的多個threads之間達到loadbalance以及讓狀態型任務（stateful tasks）維持穩定(sticky, 同個task就在同個thread / instance上執行完)

- 文件裡的範例說明了record如何在kafka stream裡被處理
  傳入了一段"all streams lead to kafka"，每個字元和計數則放至KTable，**"被改變的的Record"**會被送到downstream，即KStream
- 總結來說KTable像是資料表，KStream則是ChangeLog，抓下所有的Stream = 取得所有改變，所以可以回推整張表
![image](https://github.com/user-attachments/assets/460c269b-d7ff-4c7a-90aa-086865afd780)
<br/>*圖源: [kafka stream document](https://kafka.apache.org/39/documentation/streams/quickstart)*

### [Threading Model](https://kafka.apache.org/31/documentation/streams/architecture#streams_architecture_threads)

- 增加更多stream thread / instances = 複製更多處理程序(topology)，來處理不同的partition
- 每個stream thread之間沒有共享狀態，所以不會因為狀態共享導致性能下降
- 每個thread處理*至少一個*Task

- How to scale out?
  - instances 增加，Kafka Streams就會自動處理這些instance之間如何分配topic partition，即應用程式在多個實例中運行時，Kafka Streams 會根據每個任務的需求，自動決定哪些 partition 由哪個任務處理
  - 也可以動態擴縮 stream threads，Kafka Streams 會自動分配 partitions，確保每個運行中的執行緒都有 partition 可以處理，若某些執行緒失效，可以直接新增執行緒來取代它們，而不需要重啟整個客戶端

### [Local State Stores](https://kafka.apache.org/31/documentation/streams/architecture#streams_architecture_state)
- 用於查詢stream process中，資料的中間狀態
- 進行聚合（aggregations）、連接（joins）或窗口運算（windowed computations）等操作，需要在不同的資料進來之間保存某些中間結果。state stores 用來保存這些資料
- 可即時查詢和容錯恢復

### [Fault Tolerance](https://kafka.apache.org/31/documentation/streams/architecture#streams_architecture_recovery)
- 進入kafka的數據在Application出問題並重啟後，仍可以繼續處理數據
- fault-tolerance capability is offered by the Kafka comsumer client
- 執行task的機器如果掛了，會自動在別的運行中的instance重新執行task
- 每個local state store 都有維護一個複製的change log kafka topic 來追蹤狀態更新，並且這些log可以被壓縮清除。以防主無限增長。 故障重啟時，也會以此復原

## Kafka Connect

- 導入plugin connect-file-3.9.0.jar後使用
- 定義資料源(config/connect-file-source.properties)，例如file location、寫入的topic等等，以及定義接收端(config/connect-file-sink.properties)
- connect-standalone/distributed.properties: worker 配置檔，主要用來設定 Kafka Connect 的運行環境（例如連接 Kafka broker、offset 存儲的 topic、plugin 路徑、converter 等等），如果是分散式，還需定義leader vote, task assignment...etc
- 運行流程:
  - Kafka connect process 啟動 -> 從資料源讀取資料 -> produce to Topic ->  sink connector read messages from the topic -> sink connector write messages to the sink file
  - 運行中也可運行其他concumer同時消費這個topic的數據

## [Topic](https://developer.confluent.io/learn-more/kafka-on-the-go/topics/)
- appenend-only logs，寫入後就再也不會改變
- 各種events被分類儲存到到不同topic
- 可複製並持久化儲存到各個nodes
- topic數量沒有上限，取決於每個topic裡的parition數量，使用KRaft mode，數量可達百萬up

- events會被以key-value pairs存入，通常key為null or ID (其他binary data也可以，但通常是這兩種)

## [Partition](https://www.youtube.com/watch?v=y9BStKvVzSs)
當大量的event寫入至topic，並同時被存取，該節點負載會變很重

kafka高度scalable的原因在於可以透過schema design，把資料分到不同的artition
比起建立一個giant node，中小型的節點更加經濟實惠一些
儘管topic可以分散到多節點來管理，但通常不會讓topic過大，取得代之的是將topic partition
![image](https://github.com/user-attachments/assets/6810c16f-b889-40dc-a081-51ed9ba6ff34)

Topic是log，而partition就是將這個log分散成多個logs
給個partition可以佈署到cluster的單一節點上，所以所有的讀寫操作可以平行分配到這些節點上

### messages如何被分配到各個partition

如果進入topic的messages
1. 沒有key，則以round robin均勻分配到各個partition，每個messages的排序就是時間
![image](https://github.com/user-attachments/assets/810cfbd2-aea3-45a1-813f-1e4f5057e312)

2. 有key，通常會把key進行hash，用這個數值除以分區的數量，計算餘數(mod)，得出的數值就是partition number
>  *Note: 所以有key的情況下，kafka保證相同的key，**會被放到同一個aprtition，並且是有順序性的***
>  e.g. key 為customerID，每個customer相關的message可以在每個partition裡循序找到
>  某個key可能會出現量特多的情況，但通常kafka可以應付這些風險，所以還在可管控範圍內


[How to Choose the Number of Topics/Partitions in a Kafka Cluster? By Jun Rao](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/?session_ref=direct&_gl=1*1ofcog2*_gcl_aw*R0NMLjE3Mzk1OTczMjIuQ2p3S0NBaUE4THU5QmhBOEVpd0FhZzE2YnozRFF0S0V3a2VlY2NvMy1pNE5vSlViUWtYUERnTHhUTldmVzRpWllkSEJCZXdaM0gza01Sb0NUWm9RQXZEX0J3RQ..*_gcl_au*Njk2NTM5NzQ3LjE3Mzg4ODgxMTY.*_ga*ODgzNzU4NTY0LjE3Mzg4ODgxMTY.*_ga_D2D3EGKSGD*MTc0MDQ5MjMyOC4xNC4xLjE3NDA0OTM3NjIuNjAuMC4w&_ga=2.131869836.2009217855.1740492328-883758564.1738888116&_gac=1.183067476.1739597321.CjwKCAiA8Lu9BhA8EiwAag16bz3DQtKEwkeecco3-i4NoJUbQkXPDgLxTNWfW4iZYdHBBewZ3H3kMRoCTZoQAvD_BwE)

## [Consumer group](https://developer.confluent.io/learn-more/kafka-on-the-go/consumer-groups/)

當 producer 可以把大量資料分散式地傳遞到topic partition對應的不同節點，單一個consumer 也無法消耗這麼大量的資料
所以也可以使用多個nodes來分散式消費這些messages，而他們統一有一個group.Id，kafka broker會自動化的對這個group進行load balance，新的node出現，則分流給新的node; 反之。node消失，則把剩餘流量轉至既有的node

- 讓consumers 平行處理同一個subset of topic partitions

## Broker

- Kafka 物理儲存數據的Server
- topic partition messages 分散式儲存在各個broker
- 每個partition 會有一個leader broker，其他的會replicate data，故有fault tolerance

---

## kafka v.s. RabbitMQ

- kafka 可以配置retention policy 使可以持久化儲存 / RabbitMQ 在message 在comsumption後被移除
- 因為kafka有持久化儲存，comsumers 可以"重複讀取"，並且有較高的throughput
- [介紹](https://youtu.be/QkdkLdMBuL0?si=ylqp_wYwcZ3qEpTK)中的一個很好的比喻是，kafka 像是隨點即看的Netflix(Streaming)，也可以重任意點重新播放，
  而其他simple message broker像是TV，需要在撥放時同時consume，並且無法持久保存重放(雖然Rabbit MQ有Dead Letter retry的機制，但本質上較不強調persist)


## Contribution Guide

- PR
```mermaid
flowchart LR
    A([Issue]) --> B{Minor?}
    B -- Yes --> C([PR])
    B -- No --> D([JIRA])
    D --> E{Same APIs?}
    E -- Yes --> H(KIP)
    E -- No --> C
    C --> F{LGTM?}
    F -- Yes --> G([Commit])
    F -- No --> C
```

- KIP
```mermaid
flowchart LR
    A([Idea & JIRA]) --> B([Draft & PoC])
    B --> C([Discussion])
    
    C --> D{LGTM?}
    D -- Yes --> E([Voting])
    D -- No --> C
    
    %% 以下這條是虛線箭頭，表示「Discussion」到「Voting」的可選流程
    C -.-> E
    
    E --> F{Accepted?}
    F -- Yes --> G([PR])
    F -- No --> C

```

