To demonstrate the concept of Kubernetes leases for a Kafka consumer written in Java Spring Boot, we'll use a sidecar container to manage the lease and ensure that only one Kafka consumer instance is active at any given time. The active instance will consume messages from a specific Kafka partition, while other instances will remain on standby.

Here's how you can achieve this:

1. **Create the necessary Kubernetes resources** (Namespace, Lease, Service Account, Role, and Role Binding).
2. **Set up a Kafka Consumer application** in Java Spring Boot.
3. **Create a sidecar container** for leader election using Kubernetes leases.
4. **Deploy the Kafka Consumer and the sidecar** to Kubernetes.

### Step-by-Step Implementation

#### 1. Create Kubernetes Resources

**namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kafka-demo
```

**lease.yaml:**
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kafka-lease
  namespace: kafka-demo
spec:
  holderIdentity: ""
  leaseDurationSeconds: 15
```

**rbac.yaml:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafka-sa
  namespace: kafka-demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kafka-demo
  name: kafka-role
rules:
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "watch", "list", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kafka-rolebinding
  namespace: kafka-demo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kafka-role
subjects:
- kind: ServiceAccount
  name: kafka-sa
  namespace: kafka-demo
```

#### 2. Set up the Kafka Consumer in Java Spring Boot

**pom.xml:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
</dependencies>
```

**KafkaConsumerService.java:**
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

@Service
public class KafkaConsumerService {

    @Value("${active:false}")
    private boolean active;

    private final KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

    public KafkaConsumerService(KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry) {
        this.kafkaListenerEndpointRegistry = kafkaListenerEndpointRegistry;
    }

    @KafkaListener(topics = "your-topic", groupId = "your-group")
    public void listen(String message) {
        if (active) {
            System.out.println("Consumed message: " + message);
        }
    }

    public void setActive(boolean active) {
        this.active = active;
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

**KafkaConsumerApplication.java:**
```java
package com.example.kafkaconsumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(KafkaConsumerApplication.class, args);
    }
}
```

#### 3. Create the Sidecar Container for Leader Election

**LeaderElectionService.java:**
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

@Service
public class LeaderElectionService {

    @Value("${NAMESPACE}")
    private String namespace;

    @Value("${LEASE_NAME}")
    private String leaseName;

    @Value("${POD_NAME}")
    private String podName;

    private final KafkaConsumerService kafkaConsumerService;

    public LeaderElectionService(KafkaConsumerService kafkaConsumerService) {
        this.kafkaConsumerService = kafkaConsumerService;
    }

    @PostConstruct
    public void startLeaderElection() throws Exception {
        var config = Config.defaultClient();
        var lock = new LeaseLock(namespace, leaseName, new V1ObjectMeta().name(podName));
        var leaderConfig = new LeaderElectionConfig(
                lock,
                Duration.ofMillis(15000),  // lease duration
                Duration.ofMillis(10000),  // renew deadline
                Duration.ofMillis(2000)    // retry period
        );

        var leaderElector = new LeaderElector(leaderConfig);
        leaderElector.run(
                () -> setActive(true),
                () -> setActive(false),
                () -> System.out.println("Current leader: " + lock.getHolderIdentity())
        );
    }

    private void setActive(boolean isActive) {
        System.out.println(isActive ? "Became leader" : "Lost leadership");
        kafkaConsumerService.setActive(isActive);
    }
}
```

**Dockerfile for the Sidecar:**
```Dockerfile
FROM openjdk:11-jre-slim
VOLUME /tmp
COPY target/sidecar-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

#### 4. Deploy the Kafka Consumer and Sidecar to Kubernetes

**deployment.yaml:**
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
        - name: LEASE_NAME
          value: "kafka-lease"
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
        - name: LEASE_NAME
          value: "kafka-lease"
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
   kubectl apply -f lease.yaml
   kubectl apply -f rbac.yaml
   kubectl apply -f deployment.yaml
   ```

This setup ensures that only one Kafka consumer instance is active at a time for a specific partition, with Kubernetes leases managing the leader election. The active consumer instance will consume messages, while others remain on standby.
