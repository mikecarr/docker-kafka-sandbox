# Kafka Development Environment with Zookeeper and Kafka UI

This project sets up a local Apache Kafka development environment using Docker Compose. It includes:
*   **Zookeeper:** For Kafka coordination (in Zookeeper mode).
*   **Kafka:** A single Kafka broker.
*   **Kafka UI (Provectus):** A web-based UI to browse and manage your Kafka cluster.

This setup is configured for Kafka to run in Zookeeper mode (not KRaft mode).

## Components

1.  **`zookeeper` (Bitnami Zookeeper)**
    *   **Image:** `bitnami/zookeeper:latest` (or a pinned version like `bitnami/zookeeper:3.9`)
    *   **Purpose:** Apache Zookeeper is used by Kafka (when not in KRaft mode) for distributed coordination, including managing broker metadata, controller election, and tracking consumer offsets.
    *   **Configuration:** `ALLOW_ANONYMOUS_LOGIN=yes` allows Kafka and Kafka UI to connect without complex ACLs for local development.
    *   **Port:** `2181` (standard Zookeeper port)

2.  **`kafka` (Bitnami Kafka)**
    *   **Image:** `bitnami/kafka:latest` (or a pinned version like `bitnami/kafka:3.6`)
    *   **Purpose:** The Kafka message broker responsible for receiving, storing, and delivering messages.
    *   **Key Configuration:**
        *   `KAFKA_ENABLE_KRAFT=no`: **Crucial.** Explicitly tells the Bitnami Kafka image to run in Zookeeper mode, not KRaft (Zookeeper-less) mode.
        *   `KAFKA_CFG_BROKER_ID=1`: Unique ID for this Kafka broker.
        *   `KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181`: Tells Kafka where to find Zookeeper (service name `zookeeper` on port `2181`).
        *   `KAFKA_CFG_LISTENERS=INSIDE://0.0.0.0:9093,OUTSIDE://0.0.0.0:9092`:
            *   `INSIDE`: For communication *within* the Docker network (e.g., Kafka UI to Kafka, Kafka CLI tools inside the container to itself). Binds to all interfaces inside the container on port `9093`.
            *   `OUTSIDE`: For communication *from your host machine* (or external clients) to Kafka. Binds to all interfaces inside the container on port `9092`.
        *   `KAFKA_CFG_ADVERTISED_LISTENERS=INSIDE://kafka:9093,OUTSIDE://localhost:9092`:
            *   `INSIDE://kafka:9093`: How other services *inside* the Docker network (like Kafka UI) should connect to this broker (using the service name `kafka`).
            *   `OUTSIDE://localhost:9092`: How clients *outside* Docker (on your host machine) should connect to this broker (using `localhost` and the mapped host port).
        *   `KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT`: Defines PLAINTEXT for both listeners.
        *   `KAFKA_CFG_INTER_BROKER_LISTENER_NAME=INSIDE`: Specifies which listener to use for internal Kafka broker-to-broker communication (relevant if you had multiple brokers).
        *   `KAFKA_CREATE_TOPICS="my-topic:1:1"`: Automatically creates a topic named `my-topic` with 1 partition and a replication factor of 1 on startup.
        *   `KAFKA_ALLOW_PLAINTEXT_LISTENER=yes`: Acknowledges the use of PLAINTEXT for development.
    *   **Ports:**
        *   `9092:9092`: Maps host port `9092` to container port `9092` (for the `OUTSIDE` listener).

3.  **`kafka-ui` (Provectus Kafka UI)**
    *   **Image:** `provectuslabs/kafka-ui:latest` (or a pinned version like `provectuslabs/kafka-ui:v0.7.1`)
    *   **Purpose:** Provides a web interface to view Kafka brokers, topics, messages, consumer groups, and perform some administrative tasks.
    *   **Key Configuration:**
        *   `KAFKA_CLUSTERS_0_NAME=local-kafka-cluster`: Display name for the cluster in the UI.
        *   `KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9093`: Tells Kafka UI to connect to the Kafka service named `kafka` on port `9093` (the `INSIDE` listener).
        *   `KAFKA_CLUSTERS_0_ZOOKEEPER=zookeeper:2181`: Provides Zookeeper address for Kafka UI features that might use it.
    *   **Port:** `8080:8080`: Maps host port `8080` to container port `8080` for accessing the UI.

## Prerequisites

*   [Docker](https://www.docker.com/get-started)
*   [Docker Compose](https://docs.docker.com/compose/install/)

## How to Run

1.  **Save the `docker-compose.yml` file** (provided in previous interactions or create one based on the description above) in a directory on your machine.
2.  **Navigate to the directory** containing the `docker-compose.yml` file in your terminal.
3.  **Start the services:**
    ```bash
    docker-compose up -d
    ```
    The `-d` flag runs the containers in detached mode (in the background). You can omit it to see logs directly in your terminal.

    It might take a minute or two for Kafka to fully initialize and create topics, especially on the first run.

## How to Test

### 1. Check Kafka UI

*   Open your web browser and go to: `http://localhost:8080`
*   You should see the Kafka UI dashboard.
*   It should display your cluster (`local-kafka-cluster`).
*   Navigate to "Topics". You should see `my-topic` listed. Initially, it will have 0 messages.

### 2. Produce Messages (using Kafka Console Producer)

*   Find your Kafka container name (e.g., `projectname_kafka_1` or `kafka-1`):
    ```bash
    docker ps
    ```
*   Open a terminal and run the console producer (replace `your_kafka_container_name` with the actual name):
    ```bash
    docker exec -it your_kafka_container_name /opt/bitnami/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9093 --topic my-topic
    ```
    *   `localhost:9093` is used because we are connecting from *within* the Kafka container to its `INSIDE` listener.
*   Type some messages, pressing Enter after each:
    ```
    >Hello from Kafka!
    >This is a test message.
    >One more message.
    ```
*   Press `Ctrl+C` to stop the producer.

### 3. Consume Messages (using Kafka Console Consumer)

*   Open a **new** terminal window.
*   Run the console consumer (replace `your_kafka_container_name`):
    ```bash
    docker exec -it your_kafka_container_name /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9093 --topic my-topic --from-beginning
    ```
*   You should see the messages you produced:
    ```
    Hello from Kafka!
    This is a test message.
    One more message.
    ```
*   Press `Ctrl+C` to stop the consumer.

### 4. Verify in Kafka UI Again

*   Refresh the "Topics" page for `my-topic` in Kafka UI, or click on the "Messages" tab for that topic.
*   You should now see the messages you produced, along with their details (offset, timestamp, etc.).

## Stopping the Stack

To stop and remove the containers:
```bash
docker-compose down
```

To stop, remove containers, AND remove associated volumes (for a clean slate):
```bash
docker-compose down -v
```

## Troubleshooting
* **Errors during startup:** Check the logs for each service:
    ```bash
    docker-compose logs zookeeper
    docker-compose logs kafka
    docker-compose logs kafka-ui
    ```
* **Port conflicts:** If ports 2181, 9092, or 8080 are already in use on your host machine, docker-compose up will fail. You can change the host-side port mapping in docker-compose.yml (e.g., change 9092:9092 to 9992:9092 and then connect to Kafka from your host on port 9992).
* **Kafka still trying KRaft mode:** Ensure KAFKA_ENABLE_KRAFT=no is present and correctly spelled in the kafka service's environment variables in docker-compose.yml. Also ensure no conflicting KRaft-specific variables like KAFKA_CFG_NODE_ID or KAFKA_CFG_PROCESS_ROLES are set if you intend to use Zookeeper.

## References
* Apache Kafka: https://kafka.apache.org/
* Bitnami Kafka Docker Image: https://hub.docker.com/r/bitnami/kafka/
* Bitnami Zookeeper Docker Image: https://hub.docker.com/r/bitnami/zookeeper/
* Provectus Kafka UI: https://github.com/provectus/kafka-ui
* Docker Compose: https://docs.docker.com/compose/

```
Remember to replace `your_kafka_container_name` in the testing commands with the actual name of your Kafka container as shown by `docker ps`. You can also often use the service name if your Docker Compose version and Docker engine support it directly with `docker exec`, like `docker exec -it kafka ...` if your service is named `kafka` and you run `docker-compose exec kafka ...`. However, using the full container name obtained from `docker ps` is generally more reliable for `docker exec`.
```