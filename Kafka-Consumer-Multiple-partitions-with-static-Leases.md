Yes, if you have multiple partitions and you want to ensure that each partition is handled by a unique leader, you would need to create multiple leases in Kubernetes. Each lease will correspond to a partition, ensuring that only one consumer instance is actively consuming messages from a particular partition at any given time, while others remain on standby.

### Steps to Implement Multiple Leases

1. **Create Lease Resources for Each Partition**:
   - Define a Lease resource in Kubernetes for each Kafka partition.
2. **Modify the Sidecar Logic**:
   - Update the sidecar logic to handle multiple leases, one for each partition.
3. **Modify the Kafka Consumer Logic**:
   - Ensure the Kafka consumer is configured to consume from specific partitions based on the acquired lease.

### Example Implementation

#### 1. Create Lease Resources for Each Partition

**leases.yaml**:
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kafka-lease-partition-0
  namespace: kafka-demo
spec:
  holderIdentity: ""
  leaseDurationSeconds: 15
---
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kafka-lease-partition-1
  namespace: kafka-demo
spec:
  holderIdentity: ""
  leaseDurationSeconds: 15
---
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kafka-lease-partition-2
  namespace: kafka-demo
spec:
  holderIdentity: ""
  leaseDurationSeconds: 15
```

#### 2. Modify the Sidecar Logic to Handle Multiple Leases

**LeaderElectionService.java**:
```java
package com.example.sidecar;

import io.kubernetes.client.extended.leaderelection.LeaderElector;
import io.kubernetes.client.extended.leaderelection.LeaderElectionConfig;
import io.kubernetes.client.extended.leaderelection.resourcelock.LeaseLock;
import io.kubernetes.client.util.Config;
import io.kubernetes.client.openapi.models.V1ObjectMeta;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.time.Duration;
import java.util.Arrays;
import java.util.List;

@Service
public class LeaderElectionService {

    @Value("${NAMESPACE}")
    private String namespace;

    @Value("${POD_NAME}")
    private String podName;

    private final KafkaConsumerService kafkaConsumerService;

    public LeaderElectionService(KafkaConsumerService kafkaConsumerService) {
        this.kafkaConsumerService = kafkaConsumerService;
    }

    @PostConstruct
    public void startLeaderElection() throws Exception {
        var config = Config.defaultClient();
        List<String> partitions = Arrays.asList("kafka-lease-partition-0", "kafka-lease-partition-1", "kafka-lease-partition-2");
        
        for (String leaseName : partitions) {
            var lock = new LeaseLock(namespace, leaseName, new V1ObjectMeta().name(podName));
            var leaderConfig = new LeaderElectionConfig(
                    lock,
                    Duration.ofMillis(15000),  // lease duration
                    Duration.ofMillis(10000),  // renew deadline
                    Duration.ofMillis(2000)    // retry period
            );

            var leaderElector = new LeaderElector(leaderConfig);
            leaderElector.run(
                    () -> setActive(true, leaseName),
                    () -> setActive(false, leaseName),
                    () -> System.out.println("Current leader for " + leaseName + ": " + lock.getHolderIdentity())
            );
        }
    }

    private void setActive(boolean isActive, String leaseName) {
        System.out.println(isActive ? "Became leader for " + leaseName : "Lost leadership for " + leaseName);
        kafkaConsumerService.setActive(isActive, leaseName);
    }
}
```

#### 3. Modify the Kafka Consumer Logic

**KafkaConsumerService.java**:
```java
package com.example.kafkaconsumer;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.MessageListenerContainer;
import org.springframework.kafka.listener.config.ContainerProperties;
import org.springframework.kafka.listener.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.listener.config.KafkaListenerEndpointRegistry;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

@Service
public class KafkaConsumerService {

    @Value("${active:false}")
    private boolean active;

    private final KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

    private Map<String, Boolean> partitionActiveStatus = new HashMap<>();

    public KafkaConsumerService(KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry) {
        this.kafkaListenerEndpointRegistry = kafkaListenerEndpointRegistry;
    }

    @KafkaListener(topics = "your-topic", groupId = "your-group", concurrency = "3")
    public void listen(String message) {
        if (partitionActiveStatus.getOrDefault("kafka-lease-partition-0", false)) {
            System.out.println("Consumed message from partition 0: " + message);
        }
        if (partitionActiveStatus.getOrDefault("kafka-lease-partition-1", false)) {
            System.out.println("Consumed message from partition 1: " + message);
        }
        if (partitionActiveStatus.getOrDefault("kafka-lease-partition-2", false)) {
            System.out.println("Consumed message from partition 2: " + message);
        }
    }

    public void setActive(boolean active, String leaseName) {
        partitionActiveStatus.put(leaseName, active);
        for (MessageListenerContainer container : kafkaListenerEndpointRegistry.getListenerContainers()) {
            if (active) {
                container.start();
            } else {
                container.stop();
            }
        }
    }
}
```

### Deploying the Updated Application

**deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
  namespace: kafka-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kafka-consumer
  template:
    metadata:
      labels:
        app: kafka-consumer
    spec:
      serviceAccountName: kafka-sa
      containers:
      - name: kafka-consumer
        image: your-kafka-consumer-image
        env:
        - name: ACTIVE
          value: "false"
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      - name: leader-election-sidecar
        image: your-sidecar-image
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### Building and Deploying

1. **Build and push the Kafka consumer image**:
   ```sh
   mvn clean package
   docker build -t your-kafka-consumer-image .
   docker push your-kafka-consumer-image
   ```

2. **Build and push the sidecar image**:
   ```sh
   mvn clean package
   docker build -t your-sidecar-image .
   docker push your-sidecar-image
   ```

3. **Apply the Kubernetes configurations**:
   ```sh
   kubectl apply -f namespace.yaml
   kubectl apply -f leases.yaml
   kubectl apply -f rbac.yaml
   kubectl apply -f deployment.yaml
   ```

This setup ensures that each Kafka partition is handled by only one consumer instance at a time, with Kubernetes leases managing the leader election for each partition. This ensures fault tolerance and prevents multiple instances from consuming the same partition simultaneously.
