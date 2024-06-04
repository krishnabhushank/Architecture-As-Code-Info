Sure, let's set up a Kubernetes deployment with Kafka consumers, using Kubernetes leases to manage active and standby instances for a specific partition. We'll deploy the Kafka consumer and a sidecar container to handle leader election using leases.

Here's the plan:
1. **Set up a Kubernetes Cluster**: Ensure you have a running Kubernetes cluster.
2. **Create a Namespace**: Create a separate namespace for our demonstration.
3. **Define a Lease Resource**: Create a Lease resource in Kubernetes.
4. **Create a Service Account and Role Binding**: Allow the pods to access and modify the Lease resource.
5. **Deploy the Kafka Consumers with Sidecar**: Deploy pods with Kafka consumers and a sidecar for lease management.
6. **Implement Leader Election Logic in the Sidecar**: The sidecar will manage lease acquisition and release.

### Step-by-Step Code Implementation

1. **Create a Namespace**

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kafka-demo
```

2. **Define a Lease Resource**

```yaml
# lease.yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kafka-lease
  namespace: kafka-demo
spec:
  holderIdentity: ""
  leaseDurationSeconds: 15
```

3. **Create a Service Account and Role Binding**

```yaml
# rbac.yaml
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

4. **Deploy the Kafka Consumers with Sidecar**

```yaml
# deployment.yaml
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

5. **Implement Leader Election Logic in the Sidecar**

Create a Spring Boot application for the sidecar to handle leader election.

**Dependencies in `pom.xml`:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.kubernetes</groupId>
        <artifactId>client-java</artifactId>
        <version>15.0.1</version>
    </dependency>
</dependencies>
```

**LeaderElectionService.java:**
```java
package com.example.kafka;

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
        // Implement logic to set the environment variable ACTIVE for Kafka Consumer
        System.out.println(isActive ? "Became leader" : "Lost leadership");
        // Here you would implement the logic to notify the Kafka consumer
        // For example, write to a shared config or use a signal
    }
}
```

**Dockerfile for the sidecar:**
```Dockerfile
FROM openjdk:11-jre-slim
VOLUME /tmp
COPY target/kafka-sidecar-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

Build and push the Docker image for the sidecar:
```sh
mvn clean package
docker build -t your-sidecar-image .
docker push your-sidecar-image
```

### Deployment

Deploy the resources to your Kubernetes cluster:
```sh
kubectl apply -f namespace.yaml
kubectl apply -f lease.yaml
kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
```

This setup ensures that only one Kafka consumer instance is active at a time for a specific partition, with Kubernetes leases managing the leader election.

**Next Steps:**
**a.** Implement the logic in the Kafka consumer to start consuming messages only if the `ACTIVE` environment variable is `true`.
**b.** Add health checks to ensure timely failover if the active consumer becomes unresponsive.
