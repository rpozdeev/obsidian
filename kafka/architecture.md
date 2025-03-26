# Архитектура Kafka

## Основные компоненты

1. **Producer (Продюсер)** – отправляет данные в Kafka.    
    - Пример: веб-приложение, которое пишет события (клики, логины).
	
2. **Consumer (Консьюмер)** – читает данные из Kafka.
    - Пример: система аналитики, которая обрабатывает эти события.
	
3. **Broker (Брокер)** – сервер, хранящий данные.
    - В кластере Kafka обычно **3+ брокера** для отказоустойчивости.
	
4. **Topic (Топик)** – «тема» или категория для сообщений (например, `user_actions`).
    - Сообщения в топике хранятся **упорядоченно** и **не удаляются** сразу после чтения.
	
5. **Partition (Партиция)** – топик делится на партиции для параллельной обработки.
    - Каждая партиция **реплицируется** (обычно 3 копии).
	
6. **ZooKeeper / KRaft** – управляет метаданными (в новых версиях ZooKeeper заменён на KRaft).

## Как данные хранятся?
- Сообщения записываются в **commit log** (файлы на диске).
- У каждого сообщения есть **offset** (уникальный номер в партиции).
- Данные **не удаляются** сразу, а живут **N дней** (настраивается).

## Запуск Kafka локально

```yaml
version: "3.9"
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - 22181:2181

  kafka1:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - 29092:29092
    hostname: kafka
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 2

  kafka2:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - 29093:29092
    hostname: kafka2
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:29093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 2

  kafka3:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - 29094:29092
    hostname: kafka2
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:29094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 2
      
  kafka-ui:
    image: provectuslabs/kafka-ui
    container_name: kafka-ui
    ports:
      - 8090:8080
    restart: always
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:29092,kafka2:29093,kafka3:29094
      - KAFKA_CLUSTERS_0_ZOOKEEPER=zookeeper:2181
    links:
      - kafka
      - kafka2
      - kafka3
      - zookeeper
```