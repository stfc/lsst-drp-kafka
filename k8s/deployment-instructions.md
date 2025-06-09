# LSST DRP Kafka on Kubernetes with Strimzi (KRaft Mode)

This guide provides step-by-step instructions to deploy the LSST DRP Kafka messaging stack on Kubernetes using Strimzi with KRaft mode (no Zookeeper).

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Directory Structure](#directory-structure)
4. [Kubernetes Manifests](#kubernetes-manifests)
5. [Production Deployment Steps](#production-deployment-steps)
6. [Verification and Testing](#verification-and-testing)
7. [Production Operations](#production-operations)
8. [Troubleshooting](#troubleshooting)

## Architecture Overview

The deployment includes:
- **Kafka Cluster**: 3-node KRaft-based cluster (no Zookeeper)
- **MirrorMaker 2**: For data replication from external sources
- **IngestD Services**: Multiple instances for different datasets (DC2, CCMS1, HSC PDR2)
- **REST Proxy**: HTTP interface to Kafka
- **Monitoring**: Prometheus metrics and exporters
- **Security**: TLS encryption, RBAC, and authentication

## Prerequisites

### Required Tools
```bash
# Install required tools
kubectl version --client  # v1.25+
helm version              # v3.0+
curl --version
base64 --version
```

### Kubernetes Cluster Requirements
- Kubernetes v1.25 or higher
- Minimum 3 worker nodes
- At least 16GB RAM and 8 CPU cores per node
- Storage class supporting persistent volumes (preferably SSD)
- LoadBalancer support for external access

### Network Requirements
- Ingress controller (nginx, traefik, etc.)
- DNS resolution for services
- External connectivity to source Kafka cluster (134.79.23.189:9094)

## Directory Structure

Create the following directory structure:

```bash
mkdir -p k8s/{base,overlays/production}
cd k8s
```

## Kubernetes Manifests

### 1. Namespace Configuration

Create `k8s/base/01-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lsst-drp
  labels:
    name: lsst-drp
```

### 2. KRaft-Based Kafka Cluster

Create `k8s/base/02-kafka-cluster.yaml`:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: lsst-kafka-cluster
  namespace: lsst-drp
  labels:
    app: lsst-kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
        configuration:
          useServiceDnsDomain: true
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
      - name: external
        port: 9094
        type: loadbalancer
        tls: false
        configuration:
          bootstrap:
            loadBalancerIP: "REPLACE_WITH_YOUR_LOAD_BALANCER_IP"
    authorization:
      type: simple
      superUsers:
        - CN=lsst-admin
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.6"
      log.retention.hours: 168
      log.segment.bytes: 1073741824
      log.retention.check.interval.ms: 300000
      num.network.threads: 8
      num.io.threads: 8
      socket.send.buffer.bytes: 102400
      socket.receive.buffer.bytes: 102400
      socket.request.max.bytes: 104857600
      num.partitions: 3
      num.recovery.threads.per.data.dir: 1
      log.flush.interval.messages: 10000
      log.flush.interval.ms: 1000
    storage:
      type: persistent-claim
      size: 500Gi
      class: fast-ssd
      deleteClaim: false
    resources:
      requests:
        memory: 4Gi
        cpu: 1000m
      limits:
        memory: 8Gi
        cpu: 2000m
    jvmOptions:
      -Xms2g
      -Xmx4g
      -XX:+UseG1GC
      -XX:MaxGCPauseMillis=20
      -XX:InitiatingHeapOccupancyPercent=35
    readinessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
    livenessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml
  # KRaft mode - no Zookeeper needed
  entityOperator:
    topicOperator:
      resources:
        requests:
          memory: 512Mi
          cpu: 200m
        limits:
          memory: 512Mi
          cpu: 500m
    userOperator:
      resources:
        requests:
          memory: 512Mi
          cpu: 200m
        limits:
          memory: 512Mi
          cpu: 500m
  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
    resources:
      requests:
        memory: 64Mi
        cpu: 100m
      limits:
        memory: 128Mi
        cpu: 500m
```

### 3. Kafka Metrics Configuration

Create `k8s/base/03-kafka-metrics.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-metrics
  namespace: lsst-drp
data:
  kafka-metrics-config.yml: |
    # Kafka JMX metrics configuration for Prometheus
    rules:
    - pattern: kafka.server<type=(.+), name=(.+)PerSec, topic=(.+)><>Count
      name: kafka_server_$1_$2_total
      type: COUNTER
      labels:
        topic: "$3"
    - pattern: kafka.server<type=(.+), name=(.+)PerSec><>Count
      name: kafka_server_$1_$2_total
      type: COUNTER
    - pattern: kafka.server<type=(.+), name=(.+), topic=(.+), partition=(.+)><>Value
      name: kafka_server_$1_$2
      type: GAUGE
      labels:
        topic: "$3"
        partition: "$4"
    - pattern: kafka.server<type=(.+), name=(.+)><>Value
      name: kafka_server_$1_$2
      type: GAUGE
    - pattern: kafka.controller<type=(.+), name=(.+)><>Value
      name: kafka_controller_$1_$2
      type: GAUGE
    - pattern: kafka.network<type=(.+), name=(.+)><>Value
      name: kafka_network_$1_$2
      type: GAUGE
```

### 4. MirrorMaker 2 Configuration

Create `k8s/base/04-mirrormaker2.yaml`:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: mirrormaker2
  namespace: lsst-drp
spec:
  version: 3.6.0
  replicas: 2
  connectCluster: "target"
  clusters:
    - alias: "source"
      bootstrapServers: "134.79.23.189:9094"
      config:
        config.storage.replication.factor: 3
        offset.storage.replication.factor: 3
        status.storage.replication.factor: 3
        config.storage.topic: mm2-configs
        offset.storage.topic: mm2-offsets
        status.storage.topic: mm2-status
    - alias: "target"
      bootstrapServers: "lsst-kafka-cluster-kafka-bootstrap:9092"
      config:
        config.storage.replication.factor: 3
        offset.storage.replication.factor: 3
        status.storage.replication.factor: 3
        config.storage.topic: mm2-configs
        offset.storage.topic: mm2-offsets
        status.storage.topic: mm2-status
  mirrors:
    - sourceCluster: "source"
      targetCluster: "target"
      sourceConnector:
        tasksMax: 4
        config:
          replication.factor: 3
          offset-syncs.topic.replication.factor: 3
          sync.topic.acls.enabled: "false"
          topics: "RAL_.*"
          topics.exclude: ".*[\\-\\.]internal,.*\\.replica,__consumer_offsets"
          groups.exclude: "console-consumer-.*,connect-.*,__.*"
          emit.heartbeats.enabled: false
          emit.checkpoints.enabled: false
          sync.topic.configs.enabled: true
          key.converter: "org.apache.kafka.connect.converters.ByteArrayConverter"
          value.converter: "org.apache.kafka.connect.converters.ByteArrayConverter"
          refresh.topics.enabled: true
          refresh.topics.interval.seconds: 600
          refresh.groups.enabled: true
          refresh.groups.interval.seconds: 600
      heartbeatConnector:
        config:
          emit.heartbeats.enabled: false
      checkpointConnector:
        config:
          emit.checkpoints.enabled: false
          sync.group.offsets.enabled: true
          sync.group.offsets.interval.seconds: 60
          emit.checkpoints.interval.seconds: 60
  resources:
    requests:
      memory: 2Gi
      cpu: 500m
    limits:
      memory: 4Gi
      cpu: 1000m
  jvmOptions:
    -Xms1g
    -Xmx2g
  metricsConfig:
    type: jmxPrometheusExporter
    valueFrom:
      configMapKeyRef:
        name: connect-metrics
        key: connect-metrics-config.yml
  logging:
    type: inline
    loggers:
      log4j.rootLogger: "INFO, stdout"
      log4j.appender.stdout: "org.apache.log4j.ConsoleAppender"
      log4j.appender.stdout.layout: "org.apache.log4j.PatternLayout"
      log4j.appender.stdout.layout.ConversionPattern: "[%d] %p %m (%c)%n"
      log4j.logger.org.reflections: "ERROR"
      log4j.logger.org.apache.zookeeper: "ERROR"
      log4j.logger.org.I0Itec.zkclient: "ERROR"
      log4j.logger.org.apache.kafka.connect.mirror: "DEBUG"
```

### 5. Connect Metrics Configuration

Create `k8s/base/05-connect-metrics.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: connect-metrics
  namespace: lsst-drp
data:
  connect-metrics-config.yml: |
    # Kafka Connect JMX metrics configuration for Prometheus
    rules:
    - pattern: kafka.connect<type=(.+), client-id=(.+)><>(.+)
      name: kafka_connect_$1_$3
      type: GAUGE
      labels:
        client_id: "$2"
    - pattern: kafka.connect<type=(.+)><>(.+)
      name: kafka_connect_$1_$2
      type: GAUGE
```

### 6. Configuration Files as ConfigMaps

Create `k8s/base/06-configmaps.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingestd-configs
  namespace: lsst-drp
data:
  dc2.yaml: |
    brokers: lsst-kafka-cluster-kafka-bootstrap:9092
    group_id: "ral_ingestd"
    num_messages: 50
    butler:
      repo: panda-test-med-1
    topics:
      usdf.RAL_BUTLER_DISK-dc2:
        rucio_prefix: davs://xrootd.echo.stfc.ac.uk:1094/lsst:datadisk/butler/
        fs_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/butler/job_outputs/
      usdf.RAL_RAW_DISK-dc2:
        rucio_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/raw/
        fs_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/raw/job_outputs/
  ccms1.yaml: |
    brokers: lsst-kafka-cluster-kafka-bootstrap:9092
    group_id: "ral_ingestd"
    num_messages: 50
    butler:
      repo: ccms1
    topics:
      usdf.RAL_BUTLER_DISK-ccms1:
        rucio_prefix: davs://xrootd.echo.stfc.ac.uk:1094/lsst:datadisk/butler/
        fs_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/butler/job_outputs/
      usdf.RAL_RAW_DISK-ccms1:
        rucio_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/raw/
        fs_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/raw/job_outputs/
  hsc_pdr2_multisite.yaml: |
    brokers: lsst-kafka-cluster-kafka-bootstrap:9092
    group_id: "ral_ingestd"
    num_messages: 50
    butler:
      repo: hsc_pdr2_multisite
    topics:
      usdf.RAL_BUTLER_DISK-hsc_pdr2_multisite:
        rucio_prefix: davs://xrootd.echo.stfc.ac.uk:1094/lsst:datadisk/butler/
        fs_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/butler/job_outputs/
      usdf.RAL_RAW_DISK-hsc_pdr2_multisite:
        rucio_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/raw/
        fs_prefix: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/raw/job_outputs/
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: butler-repos-index
  namespace: lsst-drp
data:
  butler-repos-index.yaml: |
    panda-test-med-1: /etc/panda-test-med-1-lancs/butler.yaml
    hsc_pdr2_multisite: /etc/hsc-pdr2-multisite/butler.yaml
    dc2: /etc/panda-test-med-1-lancs/butler.yaml
    ccms1: /etc/ccms1/butler.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: butler-repos
  namespace: lsst-drp
data:
  panda-test-med-1-lancs-butler.yaml: |
    datastore:
      cls: lsst.daf.butler.datastores.fileDatastore.FileDatastore
      records:
        table: file_datastore_records
      root: <butlerRoot>
    registry:
      db: postgresql://dbspgha03.fds.rl.ac.uk:5432/ral_butler
      managers:
        attributes: lsst.daf.butler.registry.attributes.DefaultButlerAttributeManager
        collections: lsst.daf.butler.registry.collections.synthIntKey.SynthIntKeyCollectionManager
        datasets: lsst.daf.butler.registry.datasets.byDimensions.ByDimensionsDatasetRecordStorageManagerUUID
        datastores: lsst.daf.butler.registry.bridge.monolithic.MonolithicDatastoreRegistryBridgeManager
        dimensions: lsst.daf.butler.registry.dimensions.static.StaticDimensionRecordStorageManager
        opaque: lsst.daf.butler.registry.opaque.ByNameOpaqueTableStorageManager
      namespace: panda_test_med_1
  hsc-pdr2-multisite-butler.yaml: |
    datastore:
      name: hsc_pdr2_multisite
      cls: lsst.daf.butler.datastores.fileDatastore.FileDatastore
      records:
        table: file_datastore_records
      root: https://webdav.echo.stfc.ac.uk:1094/lsst:datadisk/butler/repos/hsc_pdr2_multisite
    registry:
      db: postgresql://dbspgha03.fds.rl.ac.uk:5432/ral_butler
      managers:
        attributes: lsst.daf.butler.registry.attributes.DefaultButlerAttributeManager
        collections: lsst.daf.butler.registry.collections.synthIntKey.SynthIntKeyCollectionManager
        datasets: lsst.daf.butler.registry.datasets.byDimensions.ByDimensionsDatasetRecordStorageManagerUUID
        datastores: lsst.daf.butler.registry.bridge.monolithic.MonolithicDatastoreRegistryBridgeManager
        dimensions: lsst.daf.butler.registry.dimensions.static.StaticDimensionRecordStorageManager
        opaque: lsst.daf.butler.registry.opaque.ByNameOpaqueTableStorageManager
      namespace: hsc_pdr2_multisite
  ccms1-butler.yaml: |
    datastore:
      cls: lsst.daf.butler.datastores.fileDatastore.FileDatastore
      records:
        table: file_datastore_records
      root: <butlerRoot>
    registry:
      db: postgresql://dbspgha03.fds.rl.ac.uk:5432/ral_butler
      managers:
        attributes: lsst.daf.butler.registry.attributes.DefaultButlerAttributeManager
        collections: lsst.daf.butler.registry.collections.synthIntKey.SynthIntKeyCollectionManager
        datasets: lsst.daf.butler.registry.datasets.byDimensions.ByDimensionsDatasetRecordStorageManagerUUID
        datastores: lsst.daf.butler.registry.bridge.monolithic.MonolithicDatastoreRegistryBridgeManager
        dimensions: lsst.daf.butler.registry.dimensions.static.StaticDimensionRecordStorageManager
        opaque: lsst.daf.butler.registry.opaque.ByNameOpaqueTableStorageManager
      namespace: ccms1
  dc2-butler.yaml: |
    datastore:
      cls: lsst.daf.butler.datastores.fileDatastore.FileDatastore
      records:
        table: file_datastore_records
      root: <butlerRoot>
    registry:
      db: postgresql://dbspgha03.fds.rl.ac.uk:5432/ral_butler
      managers:
        attributes: lsst.daf.butler.registry.attributes.DefaultButlerAttributeManager
        collections: lsst.daf.butler.registry.collections.synthIntKey.SynthIntKeyCollectionManager
        datasets: lsst.daf.butler.registry.datasets.byDimensions.ByDimensionsDatasetRecordStorageManagerUUID
        datastores: lsst.daf.butler.registry.bridge.monolithic.MonolithicDatastoreRegistryBridgeManager
        dimensions: lsst.daf.butler.registry.dimensions.static.StaticDimensionRecordStorageManager
        opaque: lsst.daf.butler.registry.opaque.ByNameOpaqueTableStorageManager
      namespace: panda_test_med_1
```

### 7. Secrets Configuration

Create `k8s/base/07-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-auth
  namespace: lsst-drp
type: Opaque
data:
  # Replace with base64 encoded content of your db-auth.yaml
  db-auth.yaml: <BASE64_ENCODED_DB_AUTH_CONTENT>
---
# TLS certificates for Kafka (you'll need to generate these)
apiVersion: v1
kind: Secret
metadata:
  name: lsst-kafka-cluster-cluster-ca-cert
  namespace: lsst-drp
type: Opaque
data:
  ca.crt: <BASE64_ENCODED_CA_CERT>
---
apiVersion: v1
kind: Secret
metadata:
  name: lsst-admin-client-cert
  namespace: lsst-drp
type: Opaque
data:
  user.crt: <BASE64_ENCODED_CLIENT_CERT>
  user.key: <BASE64_ENCODED_CLIENT_KEY>
```

### 8. IngestD Deployments

Create `k8s/base/08-ingestd-deployments.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingestd-dc2
  namespace: lsst-drp
  labels:
    app: ingestd-dc2
    component: ingestd
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: ingestd-dc2
  template:
    metadata:
      labels:
        app: ingestd-dc2
        component: ingestd
    spec:
      serviceAccountName: ingestd-service-account
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: ingestd
        image: ghcr.io/lsst-dm/ctrl_ingestd:1.10
        imagePullPolicy: IfNotPresent
        env:
        - name: CTRL_INGESTD_CONFIG
          value: /tmp/ingestd-configs/dc2.yaml
        - name: LSST_DB_AUTH
          value: /tmp/db-auth.yaml
        - name: DAF_BUTLER_REPOSITORY_INDEX
          value: /etc/butler-repos-index.yaml
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 1000m
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "ps aux | grep '[i]ngestd' || exit 1"
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "ps aux | grep '[i]ngestd' || exit 1"
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        volumeMounts:
        - name: ingestd-configs
          mountPath: /tmp/ingestd-configs
          readOnly: true
        - name: butler-repos-index
          mountPath: /etc/butler-repos-index.yaml
          subPath: butler-repos-index.yaml
          readOnly: true
        - name: butler-repos-panda
          mountPath: /etc/panda-test-med-1-lancs/butler.yaml
          subPath: panda-test-med-1-lancs-butler.yaml
          readOnly: true
        - name: butler-repos-hsc
          mountPath: /etc/hsc-pdr2-multisite/butler.yaml
          subPath: hsc-pdr2-multisite-butler.yaml
          readOnly: true
        - name: butler-repos-ccms1
          mountPath: /etc/ccms1/butler.yaml
          subPath: ccms1-butler.yaml
          readOnly: true
        - name: butler-repos-dc2
          mountPath: /etc/panda-test-med-1/butler.yaml
          subPath: dc2-butler.yaml
          readOnly: true
        - name: db-auth
          mountPath: /tmp/db-auth.yaml
          subPath: db-auth.yaml
          readOnly: true
      volumes:
      - name: ingestd-configs
        configMap:
          name: ingestd-configs
      - name: butler-repos-index
        configMap:
          name: butler-repos-index
      - name: butler-repos-panda
        configMap:
          name: butler-repos
          items:
          - key: panda-test-med-1-lancs-butler.yaml
            path: panda-test-med-1-lancs-butler.yaml
      - name: butler-repos-hsc
        configMap:
          name: butler-repos
          items:
          - key: hsc-pdr2-multisite-butler.yaml
            path: hsc-pdr2-multisite-butler.yaml
      - name: butler-repos-ccms1
        configMap:
          name: butler-repos
          items:
          - key: ccms1-butler.yaml
            path: ccms1-butler.yaml
      - name: butler-repos-dc2
        configMap:
          name: butler-repos
          items:
          - key: dc2-butler.yaml
            path: dc2-butler.yaml
      - name: db-auth
        secret:
          secretName: db-auth
      nodeSelector:
        node-type: compute
      tolerations:
      - key: "compute-node"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
---
# Similar deployments for CCMS1 and HSC PDR2 (abbreviated for brevity)
# Copy the above deployment and modify:
# - metadata.name: ingestd-ccms1 / ingestd-hsc-pdr2-multisite
# - labels.app: ingestd-ccms1 / ingestd-hsc-pdr2-multisite
# - CTRL_INGESTD_CONFIG: ccms1.yaml / hsc_pdr2_multisite.yaml
```

### 9. REST Proxy and Services

Create `k8s/base/09-services.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rest-proxy
  namespace: lsst-drp
  labels:
    app: rest-proxy
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: rest-proxy
  template:
    metadata:
      labels:
        app: rest-proxy
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: rest-proxy
        image: confluentinc/cp-kafka-rest:7.5.0
        ports:
        - containerPort: 8082
          name: rest-proxy
        env:
        - name: KAFKA_REST_HOST_NAME
          value: rest-proxy
        - name: KAFKA_REST_BOOTSTRAP_SERVERS
          value: lsst-kafka-cluster-kafka-bootstrap:9092
        - name: KAFKA_REST_LISTENERS
          value: http://0.0.0.0:8082
        - name: KAFKA_REST_SCHEMA_REGISTRY_URL
          value: http://schema-registry:8081
        resources:
          requests:
            memory: 512Mi
            cpu: 200m
          limits:
            memory: 1Gi
            cpu: 500m
        livenessProbe:
          httpGet:
            path: /
            port: 8082
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 8082
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: rest-proxy
  namespace: lsst-drp
  labels:
    app: rest-proxy
spec:
  type: LoadBalancer
  ports:
  - port: 8082
    targetPort: 8082
    name: rest-proxy
  selector:
    app: rest-proxy
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-exporter-external
  namespace: lsst-drp
  labels:
    app: kafka-exporter-external
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-exporter-external
  template:
    metadata:
      labels:
        app: kafka-exporter-external
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: kafka-exporter
        image: danielqsj/kafka-exporter:v1.7.0
        args:
        - "--kafka.server=lsst-kafka-cluster-kafka-bootstrap:9092"
        - "--web.listen-address=:9308"
        - "--log.level=info"
        ports:
        - containerPort: 9308
          name: metrics
        resources:
          requests:
            memory: 64Mi
            cpu: 100m
          limits:
            memory: 128Mi
            cpu: 200m
        livenessProbe:
          httpGet:
            path: /metrics
            port: 9308
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /metrics
            port: 9308
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-exporter-external
  namespace: lsst-drp
  labels:
    app: kafka-exporter-external
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9308"
    prometheus.io/path: "/metrics"
spec:
  ports:
  - port: 9308
    targetPort: 9308
    name: metrics
  selector:
    app: kafka-exporter-external
```

### 10. RBAC Configuration

Create `k8s/base/10-rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingestd-service-account
  namespace: lsst-drp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: lsst-drp
  name: ingestd-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ingestd-role-binding
  namespace: lsst-drp
subjects:
- kind: ServiceAccount
  name: ingestd-service-account
  namespace: lsst-drp
roleRef:
  kind: Role
  name: ingestd-role
  apiGroup: rbac.authorization.k8s.io
```

### 11. Monitoring Configuration

Create `k8s/base/11-monitoring.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kafka-metrics
  namespace: lsst-drp
  labels:
    app: kafka
spec:
  selector:
    matchLabels:
      strimzi.io/name: lsst-kafka-cluster-kafka
  endpoints:
  - port: tcp-prometheus
    interval: 30s
    path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kafka-exporter-metrics
  namespace: lsst-drp
  labels:
    app: kafka-exporter
spec:
  selector:
    matchLabels:
      app: kafka-exporter-external
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mirrormaker2-metrics
  namespace: lsst-drp
  labels:
    app: mirrormaker2
spec:
  selector:
    matchLabels:
      strimzi.io/name: mirrormaker2-connect
  endpoints:
  - port: tcp-prometheus
    interval: 30s
    path: /metrics
```

## Production Deployment Steps

### Step 1: Prerequisites Setup

1. **Install Strimzi Operator**
```bash
# Create the operator namespace
kubectl create namespace kafka-operator

# Install Strimzi operator
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka-operator' -n kafka-operator

# Wait for operator to be ready
kubectl wait --for=condition=Ready pod -l name=strimzi-cluster-operator -n kafka-operator --timeout=300s
```

2. **Install Cert-Manager (for TLS certificates)**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

3. **Verify Storage Class**
```bash
# Check available storage classes
kubectl get storageclass

# If 'fast-ssd' doesn't exist, create one or modify the Kafka spec
# Example for creating a fast-ssd storage class:
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true
EOF
```

### Step 2: Prepare Configuration

1. **Create DB Auth Secret**
```bash
# Base64 encode your db-auth.yaml
base64 -i db-auth.yaml > db-auth-base64.txt

# Edit k8s/base/07-secrets.yaml and replace <BASE64_ENCODED_DB_AUTH_CONTENT>
# with the content from db-auth-base64.txt
```

2. **Update Load Balancer IP**
```bash
# Edit k8s/base/02-kafka-cluster.yaml
# Replace REPLACE_WITH_YOUR_LOAD_BALANCER_IP with your actual IP
# Or remove the loadBalancerIP line to use dynamic assignment
```

### Step 3: Deploy Infrastructure

1. **Create Namespace**
```bash
kubectl apply -f k8s/base/01-namespace.yaml
```

2. **Deploy Secrets and ConfigMaps**
```bash
kubectl apply -f k8s/base/07-secrets.yaml
kubectl apply -f k8s/base/06-configmaps.yaml
kubectl apply -f k8s/base/03-kafka-metrics.yaml
kubectl apply -f k8s/base/05-connect-metrics.yaml
```

3. **Deploy RBAC**
```bash
kubectl apply -f k8s/base/10-rbac.yaml
```

### Step 4: Deploy Kafka Cluster

1. **Deploy Kafka with KRaft**
```bash
kubectl apply -f k8s/base/02-kafka-cluster.yaml

# Wait for Kafka cluster to be ready (this may take 10-15 minutes)
kubectl wait --for=condition=Ready kafka/lsst-kafka-cluster -n lsst-drp --timeout=900s

# Check cluster status
kubectl get kafka -n lsst-drp
kubectl get pods -n lsst-drp -l strimzi.io/cluster=lsst-kafka-cluster
```

2. **Verify Kafka Cluster**
```bash
# Check if all Kafka pods are running
kubectl get pods -n lsst-drp -l strimzi.io/name=lsst-kafka-cluster-kafka

# Check Kafka services
kubectl get svc -n lsst-drp -l strimzi.io/cluster=lsst-kafka-cluster

# Test basic Kafka operations
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### Step 5: Deploy MirrorMaker 2

1. **Deploy MirrorMaker 2**
```bash
kubectl apply -f k8s/base/04-mirrormaker2.yaml

# Wait for MirrorMaker to be ready
kubectl wait --for=condition=Ready kafkamirrormaker2/mirrormaker2 -n lsst-drp --timeout=600s

# Check MirrorMaker status
kubectl get kafkamirrormaker2 -n lsst-drp
kubectl get pods -n lsst-drp -l strimzi.io/kind=KafkaMirrorMaker2
```

2. **Verify MirrorMaker 2**
```bash
# Check MirrorMaker logs
kubectl logs -n lsst-drp -l strimzi.io/name=mirrormaker2-connect

# Check if topics are being replicated
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --list | grep -E "^usdf\."
```

### Step 6: Deploy Applications

1. **Deploy IngestD Services**
```bash
kubectl apply -f k8s/base/08-ingestd-deployments.yaml

# Check deployment status
kubectl get deployments -n lsst-drp -l component=ingestd
kubectl get pods -n lsst-drp -l component=ingestd

# Check logs
kubectl logs -n lsst-drp -l app=ingestd-dc2
```

2. **Deploy Supporting Services**
```bash
kubectl apply -f k8s/base/09-services.yaml

# Check services
kubectl get svc -n lsst-drp
kubectl get deployments -n lsst-drp
```

### Step 7: Deploy Monitoring (Optional)

```bash
# Deploy monitoring if Prometheus operator is installed
kubectl apply -f k8s/base/11-monitoring.yaml

# Check ServiceMonitors
kubectl get servicemonitor -n lsst-drp
```

## Verification and Testing

### Health Checks

1. **Check All Pods**
```bash
kubectl get pods -n lsst-drp
kubectl get all -n lsst-drp
```

2. **Kafka Cluster Health**
```bash
# Check cluster status
kubectl describe kafka lsst-kafka-cluster -n lsst-drp

# List topics
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# Check cluster metadata
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

3. **MirrorMaker 2 Health**
```bash
# Check connector status
kubectl exec -n lsst-drp deployment/mirrormaker2-connect -- curl -s localhost:8083/connectors

# Check connector health
kubectl exec -n lsst-drp deployment/mirrormaker2-connect -- curl -s localhost:8083/connectors/source->target.MirrorSourceConnector/status
```

### Test Kafka Functionality

1. **Create Test Topic**
```bash
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-topics.sh \
  --create --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 3
```

2. **Test Producer/Consumer**
```bash
# Producer test
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-console-producer.sh \
  --topic test-topic --bootstrap-server localhost:9092 \
  --property "parse.key=true" --property "key.separator=:"

# Consumer test (in another terminal)
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-console-consumer.sh \
  --topic test-topic --bootstrap-server localhost:9092 \
  --from-beginning --property print.key=true
```

3. **Test REST Proxy**
```bash
# Get REST Proxy service endpoint
kubectl get svc rest-proxy -n lsst-drp

# Test REST API (replace with actual external IP)
REST_PROXY_IP=$(kubectl get svc rest-proxy -n lsst-drp -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://${REST_PROXY_IP}:8082/topics
```

## Production Operations

### Monitoring and Alerting

1. **Metrics Collection**
```bash
# Check metrics endpoints
kubectl get svc -n lsst-drp -l "prometheus.io/scrape=true"

# Test metrics endpoint
kubectl port-forward -n lsst-drp svc/kafka-exporter-external 9308:9308
curl http://localhost:9308/metrics
```

2. **Log Aggregation**
```bash
# View application logs
kubectl logs -n lsst-drp -l app=ingestd-dc2 --tail=100 -f

# View Kafka logs
kubectl logs -n lsst-drp -l strimzi.io/name=lsst-kafka-cluster-kafka --tail=100 -f

# View MirrorMaker logs
kubectl logs -n lsst-drp -l strimzi.io/name=mirrormaker2-connect --tail=100 -f
```

### Scaling Operations

1. **Scale Kafka Cluster**
```bash
# Scale Kafka brokers
kubectl patch kafka lsst-kafka-cluster -n lsst-drp --type='merge' -p='{"spec":{"kafka":{"replicas":5}}}'

# Wait for scaling to complete
kubectl wait --for=condition=Ready kafka/lsst-kafka-cluster -n lsst-drp --timeout=600s
```

2. **Scale IngestD Services**
```bash
# Scale IngestD deployments
kubectl scale deployment ingestd-dc2 -n lsst-drp --replicas=5
kubectl scale deployment ingestd-ccms1 -n lsst-drp --replicas=3
kubectl scale deployment ingestd-hsc-pdr2-multisite -n lsst-drp --replicas=3

# Check scaling status
kubectl get deployments -n lsst-drp -l component=ingestd
```

3. **Scale MirrorMaker 2**
```bash
# Scale MirrorMaker 2
kubectl patch kafkamirrormaker2 mirrormaker2 -n lsst-drp --type='merge' -p='{"spec":{"replicas":4}}'
```

### Backup and Recovery

1. **Backup Strategies**
```bash
# Backup Kafka topics (using Kafka tools)
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic usdf.RAL_BUTLER_DISK-dc2 \
  --from-beginning \
  --max-messages 1000 > backup-topic-data.json

# Backup persistent volumes
kubectl get pvc -n lsst-drp
# Use your cloud provider's snapshot functionality
```

2. **Configuration Backup**
```bash
# Export current configurations
kubectl get kafka lsst-kafka-cluster -n lsst-drp -o yaml > kafka-cluster-backup.yaml
kubectl get kafkamirrormaker2 mirrormaker2 -n lsst-drp -o yaml > mirrormaker2-backup.yaml

# Export all configurations
kubectl get all -n lsst-drp -o yaml > lsst-drp-full-backup.yaml
```

### Updates and Maintenance

1. **Rolling Updates**
```bash
# Update IngestD image
kubectl set image deployment/ingestd-dc2 -n lsst-drp ingestd=ghcr.io/lsst-dm/ctrl_ingestd:1.11

# Monitor rollout
kubectl rollout status deployment/ingestd-dc2 -n lsst-drp

# Rollback if needed
kubectl rollout undo deployment/ingestd-dc2 -n lsst-drp
```

2. **Kafka Cluster Updates**
```bash
# Update Kafka version
kubectl patch kafka lsst-kafka-cluster -n lsst-drp --type='merge' -p='{"spec":{"kafka":{"version":"3.7.0"}}}'

# Monitor the update process
kubectl get kafka lsst-kafka-cluster -n lsst-drp -w
```

3. **Configuration Updates**
```bash
# Update ConfigMaps
kubectl apply -f k8s/base/06-configmaps.yaml

# Restart deployments to pick up new config
kubectl rollout restart deployment/ingestd-dc2 -n lsst-drp
kubectl rollout restart deployment/ingestd-ccms1 -n lsst-drp
kubectl rollout restart deployment/ingestd-hsc-pdr2-multisite -n lsst-drp
```

## Troubleshooting

### Common Issues

1. **Kafka Cluster Not Starting**
```bash
# Check events
kubectl describe kafka lsst-kafka-cluster -n lsst-drp
kubectl get events -n lsst-drp --sort-by='.lastTimestamp'

# Check storage
kubectl get pvc -n lsst-drp
kubectl describe pvc -n lsst-drp

# Check resource limits
kubectl top pods -n lsst-drp
kubectl describe nodes
```

2. **MirrorMaker 2 Connection Issues**
```bash
# Check connectivity to source cluster
kubectl exec -n lsst-drp deployment/mirrormaker2-connect -- nc -zv 134.79.23.189 9094

# Check connector logs
kubectl logs -n lsst-drp -l strimzi.io/name=mirrormaker2-connect | grep ERROR

# Reset connector
kubectl exec -n lsst-drp deployment/mirrormaker2-connect -- curl -X DELETE localhost:8083/connectors/source--target.MirrorSourceConnector
```

3. **IngestD Pod Issues**
```bash
# Check logs
kubectl logs -n lsst-drp -l app=ingestd-dc2 --previous

# Check configuration
kubectl exec -n lsst-drp deployment/ingestd-dc2 -- cat /tmp/ingestd-configs/dc2.yaml

# Check database connectivity
kubectl exec -n lsst-drp deployment/ingestd-dc2 -- nc -zv dbspgha03.fds.rl.ac.uk 5432
```

4. **Network Issues**
```bash
# Check service endpoints
kubectl get endpoints -n lsst-drp

# Test internal connectivity
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- nc -zv lsst-kafka-cluster-kafka-bootstrap 9092

# Check DNS resolution
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- nslookup lsst-kafka-cluster-kafka-bootstrap
```

### Diagnostic Commands

1. **Cluster Overview**
```bash
# Get all resources
kubectl get all -n lsst-drp

# Check resource usage
kubectl top pods -n lsst-drp
kubectl top nodes

# Check persistent volumes
kubectl get pv,pvc -n lsst-drp
```

2. **Kafka Diagnostics**
```bash
# Check Kafka cluster info
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Check consumer groups
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Check topic details
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic usdf.RAL_BUTLER_DISK-dc2
```

3. **Performance Monitoring**
```bash
# Check JVM metrics
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- jstat -gc 1

# Check disk usage
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- df -h

# Check network connections
kubectl exec -n lsst-drp lsst-kafka-cluster-kafka-0 -- netstat -an | grep :9092
```

## Security Considerations

1. **Network Policies**
```yaml
# Example network policy to restrict access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lsst-drp-network-policy
  namespace: lsst-drp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: lsst-drp
    - namespaceSelector:
        matchLabels:
          name: monitoring
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

2. **Pod Security Standards**
```yaml
# Add to deployment specs
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault
containers:
- name: container-name
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop:
      - ALL
```

## Key Production Features

✅ **KRaft Mode**: No Zookeeper dependency  
✅ **High Availability**: 3 Kafka replicas with proper replication  
✅ **Security**: TLS encryption, authentication, and RBAC  
✅ **Monitoring**: Prometheus metrics and health checks  
✅ **Resource Management**: CPU/memory limits and requests  
✅ **Storage**: Persistent volumes with proper storage classes  
✅ **Graceful Updates**: Rolling update strategies  
✅ **Production Logging**: Structured logging configuration  
✅ **Backup Strategy**: Persistent volume configuration for data retention  
✅ **Scalability**: Horizontal scaling support for all components  

This production deployment provides enterprise-grade reliability, security, and operational excellence for the LSST DRP Kafka messaging infrastructure.