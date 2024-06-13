Yes, you can dynamically create the leases within your application or sidecar container rather than manually creating them. This can be done during the initialization phase of your application.

Here's how you can modify the sidecar to dynamically create leases for each Kafka partition if they don't already exist.

### Modifying the Sidecar for Dynamic Lease Creation

1. **Modify the LeaderElectionService.java to Create Leases Dynamically**:
   
```java
package com.example.sidecar;

import io.kubernetes.client.extended.leaderelection.LeaderElector;
import io.kubernetes.client.extended.leaderelection.LeaderElectionConfig;
import io.kubernetes.client.extended.leaderelection.resourcelock.LeaseLock;
import io.kubernetes.client.util.Config;
import io.kubernetes.client.openapi.models.V1Lease;
import io.kubernetes.client.openapi.models.V1LeaseSpec;
import io.kubernetes.client.openapi.models.V1ObjectMeta;
import io.kubernetes.client.openapi.models.V1OwnerReference;
import io.kubernetes.client.openapi.models.V1Pod;
import io.kubernetes.client.openapi.models.V1PodList;
import io.kubernetes.client.openapi.models.V1PodSpec;
import io.kubernetes.client.openapi.models.V1PodTemplateSpec;
import io.kubernetes.client.openapi.models.V1ServiceAccount;
import io.kubernetes.client.openapi.models.V1ServiceAccountList;
import io.kubernetes.client.openapi.models.V1ServiceAccountSpec;
import io.kubernetes.client.openapi.models.V1beta1Lease;
import io.kubernetes.client.openapi.models.V1beta1LeaseSpec;
import io.kubernetes.client.openapi.apis.CoordinationV1Api;
import io.kubernetes.client.openapi.apis.CoreV1Api;
import io.kubernetes.client.openapi.models.V1OwnerReference;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.time.Duration;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

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
        var coordinationApi = new CoordinationV1Api(config);
        List<String> partitions = Arrays.asList("kafka-lease-partition-0", "kafka-lease-partition-1", "kafka-lease-partition-2");
        
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

### Steps to Implement

1. **Build and Push the Sidecar Image**:
   ```sh
   mvn clean package
   docker build -t your-sidecar-image .
   docker push your-sidecar-image
   ```

2. **Deploy the Application**:

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

3. **Apply the Kubernetes Configurations**:
   ```sh
   kubectl apply -f namespace.yaml
   kubectl apply -f rbac.yaml
   kubectl apply -f deployment.yaml
   ```

### Summary

By modifying the sidecar to dynamically create leases if they do not already exist, you eliminate the need to manually create them. This approach simplifies the management of leases, especially in environments where partitions may change frequently. The sidecar checks for the existence of each lease and creates it if necessary before proceeding with the leader election process. This ensures that each Kafka partition is managed by a unique leader, providing fault tolerance and preventing multiple instances from consuming the same partition simultaneously.
