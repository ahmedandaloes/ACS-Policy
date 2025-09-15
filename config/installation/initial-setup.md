# ACS Initial Installation Guide

This guide provides step-by-step instructions for installing Red Hat Advanced Cluster Security (ACS) in a production environment.

## Prerequisites

### System Requirements
- Kubernetes 1.19+ or OpenShift 4.6+
- Minimum 4 CPU cores and 8GB RAM for Central
- Minimum 1 CPU core and 2GB RAM per Sensor
- Persistent storage for Central database
- Network connectivity between all components

### Network Requirements
- Port 443 (HTTPS) for Central UI and API
- Port 8443 for Central-to-Sensor communication
- Outbound internet access for image scanning (optional)
- Internal DNS resolution for service discovery

## Installation Methods

### Method 1: Operator Installation (Recommended)

1. **Install the ACS Operator**
   ```bash
   # Create namespace
   kubectl create namespace stackrox
   
   # Install operator via OperatorHub or manual deployment
   kubectl apply -f acs-operator.yaml
   ```

2. **Deploy Central**
   ```yaml
   apiVersion: platform.stackrox.io/v1alpha1
   kind: Central
   metadata:
     name: stackrox-central-services
     namespace: stackrox
   spec:
     central:
       exposure:
         loadBalancer:
           enabled: true
           port: 443
       persistence:
         persistentVolumeClaim:
           claimName: stackrox-db
           size: 100Gi
           storageClass: fast-ssd
     scanner:
       analyzer:
         scaling:
           autoScaling: ENABLED
           maxReplicas: 5
           minReplicas: 2
       db:
         persistence:
           persistentVolumeClaim:
             claimName: scanner-db
             size: 10Gi
   ```

3. **Configure Scanner**
   ```bash
   # Enable image scanning
   kubectl patch central stackrox-central-services -n stackrox --type merge -p '{"spec":{"scanner":{"scannerComponent":"Enabled"}}}'
   ```

### Method 2: Helm Installation

1. **Add Helm Repository**
   ```bash
   helm repo add stackrox https://charts.stackrox.io
   helm repo update
   ```

2. **Install Central**
   ```bash
   helm install stackrox-central-services stackrox/central-services \
     --namespace stackrox --create-namespace \
     --set central.persistence.size=100Gi \
     --set central.exposure.loadBalancer.enabled=true
   ```

## Post-Installation Configuration

### 1. Access Central UI
```bash
# Get LoadBalancer IP
kubectl get svc central-loadbalancer -n stackrox

# Get initial admin password
kubectl get secret central-htpasswd -n stackrox -o jsonpath='{.data.password}' | base64 -d
```

### 2. Generate Sensor Bundle
```bash
# Install roxctl CLI
curl -O https://mirror.openshift.com/pub/rhacs/assets/latest/bin/Linux/roxctl
chmod +x roxctl && sudo mv roxctl /usr/local/bin/

# Generate sensor bundle
roxctl sensor generate k8s --name production-cluster \
  --central-endpoint central.stackrox.svc.cluster.local:443 \
  --insecure-skip-tls-verify \
  --collection-method kernel-module
```

### 3. Deploy Sensor
```bash
# Apply sensor manifests
kubectl apply -R -f sensor-production-cluster/
```

### 4. Configure Authentication

#### SSO Integration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: central-config
  namespace: stackrox
data:
  config.yaml: |
    auth:
      providers:
        - name: "Corporate OIDC"
          type: "oidc"
          oidc:
            issuer: "https://auth.company.com"
            client_id: "stackrox"
            client_secret: "${OIDC_CLIENT_SECRET}"
            disable_offline_access_scope: false
```

#### RBAC Configuration
```bash
# Create role for security team
roxctl central rbac create-role \
  --name "Security Analyst" \
  --permissions "Image,Policy,Deployment:read" \
  --access-scope "Production Clusters"
```

## Security Hardening

### 1. TLS Certificate Configuration
```bash
# Generate or import TLS certificates
kubectl create secret tls central-tls \
  --cert=central.crt \
  --key=central.key \
  -n stackrox

# Update Central to use custom TLS
kubectl patch central stackrox-central-services -n stackrox \
  --type merge -p '{"spec":{"central":{"defaultTLSSecret":{"name":"central-tls"}}}}'
```

### 2. Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: stackrox-central
  namespace: stackrox
spec:
  podSelector:
    matchLabels:
      app: central
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: stackrox
    ports:
    - protocol: TCP
      port: 8443
  - ports:
    - protocol: TCP
      port: 443
  egress:
  - {} # Allow all egress for scanner functionality
```

## Validation and Testing

### 1. Component Health Check
```bash
# Check all pods are running
kubectl get pods -n stackrox

# Verify Central is accessible
curl -k https://central.example.com/v1/ping

# Test sensor connectivity
roxctl sensor get-bundle test-cluster
```

### 2. Scanner Functionality
```bash
# Test image scanning
roxctl image scan --image nginx:latest

# Verify vulnerability database updates
kubectl logs -l app=scanner -n stackrox
```

### 3. Policy Enforcement
```bash
# Import test policies
roxctl policy import --file test-policies.yaml

# Deploy test workload to verify policy enforcement
kubectl apply -f test-deployment.yaml
```

## Backup and Disaster Recovery

### 1. Database Backup
```bash
# Create backup of Central database
kubectl exec central-db-0 -n stackrox -- pg_dump stackrox > central-backup.sql

# Schedule regular backups
kubectl create cronjob central-backup \
  --image=postgres:13 \
  --schedule="0 2 * * *" \
  --restart=OnFailure \
  -- pg_dump -h central-db stackrox
```

### 2. Configuration Backup
```bash
# Export policies and configurations
roxctl policy export --output policies-backup.json
roxctl central backup --output central-config-backup.zip
```

## Troubleshooting

### Common Issues

1. **Central pod stuck in pending state**
   - Check PVC availability and storage class
   - Verify resource requests can be satisfied

2. **Sensor connectivity issues**
   - Verify network policies allow communication
   - Check DNS resolution between clusters
   - Validate TLS certificates

3. **Scanner not functioning**
   - Check internet connectivity for vulnerability updates
   - Verify scanner pod resources and scaling
   - Review scanner logs for errors

### Diagnostic Commands
```bash
# Collect ACS logs
kubectl logs -l app=central -n stackrox --previous
kubectl logs -l app=sensor -n stackrox --previous
kubectl logs -l app=scanner -n stackrox --previous

# Check resource usage
kubectl top pods -n stackrox

# Verify network connectivity
kubectl exec -it deploy/central -n stackrox -- nslookup kubernetes.default.svc.cluster.local
```

## Next Steps

After successful installation:
1. Review and customize security policies
2. Configure integrations with existing tools
3. Set up monitoring and alerting
4. Train team members on ACS usage
5. Establish operational procedures

For detailed configuration options, see the [Component Configuration](../components/) guides.