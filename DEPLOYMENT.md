# Design76 Kubernetes Deployment Guide

## 📁 Folder Structure
```
k8s/
├── 01-storage/           # Persistent Volume Claims
├── 02-configmaps/       # Configuration and Environment
├── 03-database/         # MongoDB Database
├── 04-messaging/        # RabbitMQ, Kafka, Zookeeper
├── 05-cache/           # Hazelcast Distributed Cache
├── 06-observability/   # Prometheus, Grafana, Elasticsearch, Kibana
└── 07-managers/        # Schema Manager Application
```

## 🚀 Step-by-Step Deployment Process

### Prerequisites
- Kubernetes cluster running and accessible
- `kubectl` configured and connected to your cluster
- Sufficient cluster resources for all services

### Phase 1: Environment Setup
Create namespace and environment configuration first:

```bash
# Deploy namespace and environment variables
kubectl apply -f k8s/01-namespace/namespace.yaml

# Verify namespace creation
kubectl get namespace design74-infrastructure
```

### Phase 2: Storage Infrastructure
Deploy persistent volume claims for data storage:

```bash
# Deploy all storage claims
kubectl apply -f k8s/01-storage/persistent-volumes.yaml

# Verify PVCs are created
kubectl get pvc -n design74-infrastructure
```

### Phase 3: Shared Configuration
Deploy shared configuration maps:

```bash
# Deploy shared configuration
kubectl apply -f k8s/02-configmaps/shared-config.yaml

# Verify ConfigMaps
kubectl get configmaps -n design74-infrastructure
```

### Phase 4: Database Layer
Deploy MongoDB database:

```bash
# Deploy MongoDB
kubectl apply -f k8s/03-database/mongodb.yaml

# Wait for MongoDB to be ready
kubectl wait --for=condition=ready pod -l app=design74-mongodb -n design74-infrastructure --timeout=300s

# Verify MongoDB deployment
kubectl get pods -l app=design74-mongodb -n design74-infrastructure
kubectl logs -l app=design74-mongodb -n design74-infrastructure --tail=10
```

### Phase 5: Messaging Infrastructure
Deploy messaging services in correct order:

```bash
# Deploy Zookeeper first (Kafka dependency)
kubectl apply -f k8s/04-messaging/zookeeper.yaml

# Wait for Zookeeper to be ready
kubectl wait --for=condition=ready pod -l app=design74-zookeeper -n design74-infrastructure --timeout=300s

# Deploy Kafka (depends on Zookeeper)
kubectl apply -f k8s/04-messaging/kafka.yaml

# Wait for Kafka to be ready
kubectl wait --for=condition=ready pod -l app=design74-kafka -n design74-infrastructure --timeout=300s

# Deploy RabbitMQ (independent)
kubectl apply -f k8s/04-messaging/rabbitmq.yaml

# Deploy Kafka UI (depends on Kafka)
kubectl apply -f k8s/04-messaging/kafka-ui.yaml

# Verify messaging services
kubectl get pods -l component=messaging -n design74-infrastructure
kubectl get pods -l component=coordination -n design74-infrastructure
kubectl get pods -l component=streaming -n design74-infrastructure
```

### Phase 6: Cache Layer
Deploy distributed cache:

```bash
# Deploy Hazelcast
kubectl apply -f k8s/05-cache/hazelcast.yaml

# Wait for Hazelcast to be ready
kubectl wait --for=condition=ready pod -l app=design74-hazelcast -n design74-infrastructure --timeout=300s

# Verify cache deployment
kubectl get pods -l app=design74-hazelcast -n design74-infrastructure
```

### Phase 7: Observability Stack
Deploy monitoring and logging infrastructure:

```bash
# Deploy Elasticsearch first (required by Kibana)
kubectl apply -f k8s/06-observability/elasticsearch.yaml

# Wait for Elasticsearch (takes longer to start)
kubectl wait --for=condition=ready pod -l app=design74-elasticsearch -n design74-infrastructure --timeout=600s

# Deploy OpenTelemetry Collector
kubectl apply -f k8s/06-observability/otel-collector.yaml

# Deploy Prometheus
kubectl apply -f k8s/06-observability/prometheus.yaml

# Deploy Grafana
kubectl apply -f k8s/06-observability/grafana.yaml

# Deploy Kibana (depends on Elasticsearch)
kubectl apply -f k8s/06-observability/kibana.yaml

# Verify observability stack
kubectl get pods -l component=search -n design74-infrastructure
kubectl get pods -l component=metrics -n design74-infrastructure
kubectl get pods -l component=dashboard -n design74-infrastructure
kubectl get pods -l component=telemetry -n design74-infrastructure
```

### Phase 8: Application Layer
Deploy the Schema Manager application:

```bash
# Deploy Schema Manager ConfigMap
kubectl apply -f k8s/07-managers/schema-manager/configmap.yaml

# Deploy Schema Manager Deployment
kubectl apply -f k8s/07-managers/schema-manager/deployment.yaml

# Deploy Schema Manager Service
kubectl apply -f k8s/07-managers/schema-manager/service-internal.yaml

# Deploy Horizontal Pod Autoscaler
kubectl apply -f k8s/07-managers/schema-manager/hpa.yaml

# Deploy Network Policy
kubectl apply -f k8s/07-managers/schema-manager/networkpolicy.yaml

# Wait for Schema Manager to be ready
kubectl wait --for=condition=ready pod -l app=design74-schema-manager -n design74-infrastructure --timeout=300s

# Verify application deployment
kubectl get pods -l app=design74-schema-manager -n design74-infrastructure
kubectl logs -l app=design74-schema-manager -n design74-infrastructure --tail=10
```

## 🔍 Verification Commands

### Overall System Status
```bash
# Check all pods
kubectl get pods -n design74-infrastructure

# Check all services
kubectl get services -n design74-infrastructure

# Check all PVCs
kubectl get pvc -n design74-infrastructure

# Check recent events
kubectl get events -n design74-infrastructure --sort-by='.lastTimestamp' | tail -20
```

### Individual Service Health
```bash
# MongoDB
kubectl logs -l app=design74-mongodb -n design74-infrastructure --tail=20

# RabbitMQ
kubectl logs -l app=design74-rabbitmq -n design74-infrastructure --tail=20

# Kafka
kubectl logs -l app=design74-kafka -n design74-infrastructure --tail=20

# Elasticsearch
kubectl logs -l app=design74-elasticsearch -n design74-infrastructure --tail=20

# Schema Manager
kubectl logs -l app=design74-schema-manager -n design74-infrastructure --tail=20
```

### Resource Usage
```bash
# Check resource usage
kubectl top pods -n design74-infrastructure

# Check node resources
kubectl top nodes
```

## 🌐 Accessing Services

### Port Forwarding for Local Access
```bash
# MongoDB
kubectl port-forward svc/design74-mongodb 27017:27017 -n design74-infrastructure

# RabbitMQ Management
kubectl port-forward svc/design74-rabbitmq 15672:15672 -n design74-infrastructure

# Kafka UI
kubectl port-forward svc/design74-kafka-ui 8082:8080 -n design74-infrastructure

# Elasticsearch
kubectl port-forward svc/design74-elasticsearch 9200:9200 -n design74-infrastructure

# Kibana
kubectl port-forward svc/design74-kibana 5601:5601 -n design74-infrastructure

# Prometheus
kubectl port-forward svc/design74-prometheus 9090:9090 -n design74-infrastructure

# Grafana
kubectl port-forward svc/design74-grafana 3000:3000 -n design74-infrastructure

# Schema Manager (if external service exists)
kubectl port-forward svc/design74-schema-manager 5160:5160 -n design74-infrastructure
```

### Service URLs (after port forwarding)
- **MongoDB**: `mongodb://localhost:27017`
- **RabbitMQ Management**: `http://localhost:15672` (guest/guest)
- **Kafka UI**: `http://localhost:8082`
- **Elasticsearch**: `http://localhost:9200`
- **Kibana**: `http://localhost:5601`
- **Prometheus**: `http://localhost:9090`
- **Grafana**: `http://localhost:3000` (admin/admin)
- **Schema Manager**: `http://localhost:5160`

## 🛠️ Troubleshooting

### Common Issues
```bash
# Check pod status and events
kubectl describe pod <pod-name> -n design74-infrastructure

# Check service endpoints
kubectl get endpoints -n design74-infrastructure

# Check storage class
kubectl get storageclass

# Check cluster resources
kubectl describe nodes
```

### Cleanup (if needed)
```bash
# Delete entire namespace (removes everything)
kubectl delete namespace design74-infrastructure

# Or delete individual components
kubectl delete -f k8s/07-managers/schema-manager/
kubectl delete -f k8s/06-observability/
kubectl delete -f k8s/05-cache/
kubectl delete -f k8s/04-messaging/
kubectl delete -f k8s/03-database/
kubectl delete -f k8s/02-configmaps/
kubectl delete -f k8s/01-storage/
```

## 📝 Notes

- **Deployment Order**: Follow the exact order as dependencies exist between services
- **Wait Times**: Allow sufficient time for each service to start before proceeding
- **Resource Requirements**: Ensure your cluster has adequate CPU and memory
- **Storage**: Verify storage class availability for PVCs
- **Networking**: Ensure cluster networking allows inter-pod communication
- **Environment Variables**: All configuration is handled via ConfigMaps
- **Scaling**: Schema Manager is configured with HPA for auto-scaling

## ✅ Success Criteria

Deployment is successful when:
- All pods are in `Running` state
- All PVCs are `Bound`
- Services have valid endpoints
- Applications respond to health checks
- Port forwarding works for external access
