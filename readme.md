# Kubernetes Elasticsearch & Kibana Stack

This repository contains Kubernetes manifests to deploy an Elasticsearch and Kibana stack with persistent storage and configurable authentication.

## 📋 Prerequisites

- Kubernetes cluster (tested with Docker Desktop)
- `kubectl` configured to connect to your cluster
- At least 4GB RAM available for Elasticsearch

## 🏗️ Architecture

```
┌─────────────────┐    ┌─────────────────┐
│     Kibana      │────│  Elasticsearch  │
│   (Port 5601)   │    │   (Port 9200)   │
└─────────────────┘    └─────────────────┘
         │                       │
         └───────────────────────┘
              Persistent Volume
```

## 📁 Project Structure

```
k8s-es/
├── elasticsearch/
│   ├── 00-elastic-config-map.yaml    # Elasticsearch configuration
│   ├── 01-elastic-pvc.yaml          # Persistent Volume Claim
│   ├── 02-elastic-deployment.yaml   # Elasticsearch deployment
│   └── 03-elastic-service.yaml      # Elasticsearch service
├── kibana/
│   ├── 00-kibana-secret.yaml        # Authentication secrets
│   ├── 01-kibana-deployment.yaml    # Kibana deployment
│   └── 02-kibana-service.yaml       # Kibana service
└── README.md
```

## 🚀 Quick Start

### 1. Deploy Elasticsearch

```bash
# Apply Elasticsearch manifests
kubectl apply -f elasticsearch/

# Wait for Elasticsearch to be ready
kubectl rollout status deployment/elasticsearch
```

### 2. Deploy Kibana

```bash
# Apply Kibana manifests
kubectl apply -f kibana/

# Wait for Kibana to be ready
kubectl rollout status deployment/kibana
```

### 3. Verify Deployment

```bash
# Check pod status
kubectl get pods

# Check services
kubectl get svc
```

## 🔐 Authentication Options

This stack supports multiple authentication configurations:

### Option 1: No Authentication (Development)

For development environments, you can disable authentication:

**Elasticsearch Configuration:**
```yaml
# In elasticsearch/00-elastic-config-map.yaml
xpack.security.enabled: false
```

**Kibana Configuration:**
```yaml
# In kibana/01-kibana-deployment.yaml
env:
  - name: ELASTICSEARCH_HOSTS
    value: "http://elasticsearch:9200"
```

### Option 2: Username/Password Authentication

**Elasticsearch:** Keep security enabled with default passwords.

**Kibana Configuration:**
```yaml
# In kibana/01-kibana-deployment.yaml
env:
  - name: ELASTICSEARCH_HOSTS
    value: "http://elasticsearch:9200"
  - name: ELASTICSEARCH_USERNAME
    value: "kibana_system"
  - name: ELASTICSEARCH_PASSWORD
    value: "your_password_here"
```

### Option 3: Service Account Token (Recommended)

**Create a service token:**
```bash
# Port forward to Elasticsearch
kubectl port-forward svc/elasticsearch 9200:9200 &

# Create service token
curl -u elastic:testingPassword -X POST \
  "http://localhost:9200/_security/service/elastic/kibana/credential/token/kibana-token"
```

**Update Kibana secret:**
```yaml
# In kibana/00-kibana-secret.yaml
stringData:
  token: "YOUR_TOKEN_HERE"
```

**Kibana Configuration:**
```yaml
# In kibana/01-kibana-deployment.yaml
env:
  - name: ELASTICSEARCH_HOSTS
    value: "http://elasticsearch:9200"
  - name: ELASTICSEARCH_SERVICEACCOUNT_TOKEN
    valueFrom:
      secretKeyRef:
        name: kibana-service-token
        key: token
```

## 🌐 Accessing the Services

### Local Access (Port Forwarding)

**Elasticsearch:**
```bash
kubectl port-forward svc/elasticsearch 9200:9200
# Access at: http://localhost:9200
```

**Kibana:**
```bash
kubectl port-forward svc/kibana 5601:5601
# Access at: http://localhost:5601
```

### Authentication for API Access

**With Username/Password:**
```bash
curl -u elastic:testingPassword http://localhost:9200
```

**With Service Token:**
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:9200
```

## 📊 Resource Configuration

### Elasticsearch Resources

- **Memory:** 2Gi request, 4Gi limit
- **CPU:** 1 core request, 2 cores limit
- **Storage:** 1Gi persistent volume
- **JVM Heap:** 2GB (-Xms2g -Xmx2g)

### Kibana Resources

- **Memory:** Best effort (no limits set)
- **CPU:** Best effort (no limits set)

## 🔧 Configuration Customization

### Elasticsearch Settings

Edit `elasticsearch/00-elastic-config-map.yaml`:

```yaml
elasticsearch.yml: |
  xpack.security.enabled: true
  discovery.type: single-node
  network.host: 0.0.0.0
  cluster.name: "elasticsearch"
  node.name: "elasticsearch-node"
  # Add your custom settings here
```

### Storage Configuration

Edit `elasticsearch/01-elastic-pvc.yaml`:

```yaml
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi  # Increase storage as needed
  storageClassName: hostpath  # Change for your storage class
```

## 🐛 Troubleshooting

### Common Issues

**1. Elasticsearch won't start:**
```bash
# Check logs
kubectl logs -l app=elasticsearch

# Common issues:
# - Insufficient memory
# - Storage issues
# - Configuration errors
```

**2. Kibana authentication errors:**
```bash
# Check Kibana logs
kubectl logs -l app=kibana

# Check if Elasticsearch is accessible from Kibana
kubectl exec -it deployment/kibana -- curl http://elasticsearch:9200
```

**3. Pod in Pending state:**
```bash
# Check pod events
kubectl describe pod <pod-name>

# Common causes:
# - Resource constraints
# - Storage provisioning issues
# - Node selector issues
```

### Health Checks

**Elasticsearch Cluster Health:**
```bash
curl -u elastic:testingPassword http://localhost:9200/_cluster/health
```

**Kibana Status:**
```bash
curl http://localhost:5601/api/status
```

## 🔄 Maintenance

### Scaling

```bash
# Scale Elasticsearch (single-node mode - not recommended for production)
kubectl scale deployment elasticsearch --replicas=1

# Scale Kibana
kubectl scale deployment kibana --replicas=2
```

### Updates

```bash
# Update Elasticsearch image
kubectl set image deployment/elasticsearch elasticsearch=docker.elastic.co/elasticsearch/elasticsearch:9.0.5

# Update Kibana image
kubectl set image deployment/kibana kibana=docker.elastic.co/kibana/kibana:9.0.5
```

### Backup

```bash
# Backup persistent volume data
kubectl exec -it deployment/elasticsearch -- tar -czf /tmp/backup.tar.gz /usr/share/elasticsearch/data

# Copy backup out of container
kubectl cp <pod-name>:/tmp/backup.tar.gz ./elasticsearch-backup.tar.gz
```

## ⚠️ Production Considerations

This setup is designed for development and testing. For production use:

1. **Enable TLS/SSL** for all communications
2. **Use proper secrets management** (not plain text passwords)
3. **Configure resource limits** appropriately
4. **Set up monitoring** and alerting
5. **Implement proper backup** strategies
6. **Use multiple Elasticsearch nodes** for high availability
7. **Configure proper storage classes** for your environment
8. **Set up network policies** for security

## 📚 Additional Resources

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Elastic on Kubernetes](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)

## 📝 License

This project is provided as-is for educational and development purposes.

### To create kibana credentials in ES
```
curl -u elastic:testingPassword -X POST "http://localhost:9200/_security/user/kibana_system/_password" -H "Content-Type: application/json" -d '{"password":"kibana_password_123"}'
```