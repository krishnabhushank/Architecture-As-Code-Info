Certainly! Below are the `application.properties` files for both the Kafka consumer application and the sidecar application.

### Kafka Consumer Application

**application.properties:**
```properties
# Spring Boot application configuration
spring.application.name=kafka-consumer

# Kafka configuration
spring.kafka.bootstrap-servers=your-kafka-bootstrap-server
spring.kafka.consumer.group-id=your-group-id
spring.kafka.consumer.auto-offset-reset=earliest

# Active flag, this will be managed by the sidecar
active=false
```

### Sidecar Application

**application.properties:**
```properties
# Spring Boot application configuration
spring.application.name=leader-election-sidecar

# Kubernetes namespace and pod name (these will be injected by Kubernetes)
NAMESPACE=default
POD_NAME=default

# Lease names for partitions
lease.names=kafka-lease-partition-0,kafka-lease-partition-1,kafka-lease-partition-2
```

### Additional Code Modifications

#### Sidecar Application: Reading Lease Names from Properties

Modify the `LeaderElectionService` to read lease names from the `application.properties` file.

**LeaderElectionService.java:**
```java
package com.example.sidecar;

import io.kubernetes.client.extended.leaderelection.LeaderElector;
import io.kubernetes.client.extended.leaderelection.LeaderElectionConfig;
import io.kubernetes.client.extended.leaderelection.resourcelock.LeaseLock;
import io.kubernetes.client.util.Config;
import io.kubernetes.client.openapi.models.V1Lease;
import io.kubernetes.client.openapi.models.V1LeaseSpec;
import io.kubernetes.client.openapi.models.V1ObjectMeta;
import io.kubernetes.client.openapi.apis.CoordinationV1Api;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.time.Duration;
import java.util.List;
import java.util.Arrays;

@Service
public class LeaderElectionService {

    @Value("${NAMESPACE}")
    private String namespace;

    @Value("${POD_NAME}")
    private String podName;

    @Value("${lease.names}")
    private String leaseNames;

    private final KafkaConsumerService kafkaConsumerService;

    public LeaderElectionService(KafkaConsumerService kafkaConsumerService) {
        this.kafkaConsumerService = kafkaConsumerService;
    }

    @PostConstruct
    public void startLeaderElection() throws Exception {
        var config = Config.defaultClient();
        var coordinationApi = new CoordinationV1Api(config);
        List<String> partitions = Arrays.asList(leaseNames.split(","));

        // Create leases dynamically
        createLeasesIfNotExist(coordinationApi, partitions);

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

    private void createLeasesIfNotExist(CoordinationV1Api coordinationApi, List<String> leaseNames) throws Exception {
        for (String leaseName : leaseNames) {
            try {
                coordinationApi.readNamespacedLease(leaseName, namespace, null);
            } catch (Exception e) {
                System.out.println("Lease not found, creating lease: " + leaseName);
                V1Lease lease = new V1Lease()
                        .metadata(new V1ObjectMeta().name(leaseName).namespace(namespace))
                        .spec(new V1LeaseSpec().holderIdentity("").leaseDurationSeconds(15));
                coordinationApi.createNamespacedLease(namespace, lease, null, null, null);
            }
        }
    }

    private void setActive(boolean isActive, String leaseName) {
        System.out.println(isActive ? "Became leader for " + leaseName : "Lost leadership for " + leaseName);
        kafkaConsumerService.setActive(isActive, leaseName);
    }
}
```

With these modifications, the sidecar will dynamically read the lease names from the `application.properties` file and create them if they do not exist. The Kafka consumer application will rely on the sidecar to manage the `active` flag, ensuring that only one consumer instance is active for each partition at a time.

### Deploying the Updated Application

1. **Build and Push the Kafka Consumer Image**:
   ```sh
   mvn clean package
   docker build -t your-kafka-consumer-image .
   docker push your-kafka-consumer-image
   ```

2. **Build and Push the Sidecar Image**:
   ```sh
   mvn clean package
   docker build -t your-sidecar-image .
   docker push your-sidecar-image
   ```

3. **Apply the Kubernetes Configurations**:
   ```sh
   kubectl apply -f namespace.yaml
   kubectl apply -f rbac.yaml
   kubectl apply -f deployment.yaml
   ```

This setup ensures that your Kafka consumer application and sidecar will dynamically create and manage leases for each partition, providing fault tolerance and ensuring that only one consumer instance is active for each partition at a time.
