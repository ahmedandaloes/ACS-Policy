# ACS Security Hardening Configuration

This guide provides specific configuration recommendations for hardening Red Hat Advanced Cluster Security (ACS) deployments.

## TLS and Certificate Management

### Central TLS Configuration
```yaml
# Custom TLS certificate for Central
apiVersion: v1
kind: Secret
metadata:
  name: central-tls
  namespace: stackrox
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # Base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi... # Base64 encoded private key
---
apiVersion: platform.stackrox.io/v1alpha1
kind: Central
metadata:
  name: stackrox-central-services
  namespace: stackrox
spec:
  central:
    defaultTLSSecret:
      name: central-tls
    additionalCAs:
    - name: corporate-ca
      content: |
        -----BEGIN CERTIFICATE-----
        MIIDXTCCAkWgAwIBAgIJAKoK/hXAVrSzMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
        # ... corporate CA certificate content
        -----END CERTIFICATE-----
```

### Certificate Rotation
```bash
#!/bin/bash
# rotate-central-tls.sh

# Generate new certificate
openssl req -x509 -newkey rsa:4096 -keyout central.key -out central.crt \
    -days 365 -nodes -subj "/C=US/ST=CA/L=San Francisco/O=Company/CN=central.company.com"

# Create new secret
kubectl create secret tls central-tls-new \
    --cert=central.crt \
    --key=central.key \
    -n stackrox

# Update Central to use new certificate
kubectl patch central stackrox-central-services -n stackrox \
    --type merge -p '{"spec":{"central":{"defaultTLSSecret":{"name":"central-tls-new"}}}}'

# Wait for rollout
kubectl rollout status deployment/central -n stackrox

# Clean up old secret
kubectl delete secret central-tls -n stackrox
kubectl patch secret central-tls-new -n stackrox --type merge -p '{"metadata":{"name":"central-tls"}}'
```

## Authentication and Authorization

### OIDC Configuration
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
            client_id: "stackrox-central"
            client_secret_file: "/run/secrets/oidc/client-secret"
            disable_offline_access_scope: false
            callback_mode: "fragment"
            ui_endpoint: "https://central.company.com"
          claim_mappings:
            email: "email"
            name: "name"
            groups: "groups"
          required_attributes:
            - key: "department"
              value: "security|engineering"
---
apiVersion: v1
kind: Secret
metadata:
  name: oidc-client-secret
  namespace: stackrox
type: Opaque
data:
  client-secret: <base64-encoded-secret>
```

### RBAC Configuration
```bash
# Create custom roles
roxctl central rbac create-role \
    --name "Security Analyst" \
    --permissions "Alert,Deployment,Image,NetworkPolicy,Secret:read" \
    --permissions "Policy:read,write" \
    --access-scope "Production Clusters"

roxctl central rbac create-role \
    --name "DevOps Engineer" \
    --permissions "Deployment,Image:read" \
    --permissions "Policy:read" \
    --access-scope "Development Clusters"

roxctl central rbac create-role \
    --name "Compliance Officer" \
    --permissions "Alert,Compliance,Policy:read" \
    --access-scope "All Clusters"

# Create access scopes
roxctl central rbac create-access-scope \
    --name "Production Clusters" \
    --included-clusters "prod-cluster-1,prod-cluster-2" \
    --included-namespaces "default,production"

roxctl central rbac create-access-scope \
    --name "Development Clusters" \
    --included-clusters "dev-cluster-1,staging-cluster-1" \
    --excluded-namespaces "kube-system,kube-public"
```

## Network Security

### Network Policies for ACS Components
```yaml
# Central network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: central-network-policy
  namespace: stackrox
spec:
  podSelector:
    matchLabels:
      app: central
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow UI access
  - from: []
    ports:
    - protocol: TCP
      port: 443
  # Allow sensor connections
  - from:
    - namespaceSelector:
        matchLabels:
          name: stackrox
    ports:
    - protocol: TCP
      port: 8443
  egress:
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # Allow HTTPS for vulnerability feeds
  - to: []
    ports:
    - protocol: TCP
      port: 443
  # Allow database connections
  - to:
    - podSelector:
        matchLabels:
          app: central-db
    ports:
    - protocol: TCP
      port: 5432

---
# Sensor network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sensor-network-policy
  namespace: stackrox
spec:
  podSelector:
    matchLabels:
      app: sensor
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow collector connections
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 8443
  egress:
  # Allow central connections
  - to:
    - podSelector:
        matchLabels:
          app: central
    ports:
    - protocol: TCP
      port: 8443
  # Allow API server access
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 6443
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

### Admission Controller Configuration
```yaml
# ValidatingAdmissionWebhook for ACS
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: stackrox
webhooks:
- name: policyeval.stackrox.io
  clientConfig:
    service:
      name: admission-control
      namespace: stackrox
      path: "/validate"
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps", ""]
    apiVersions: ["v1", "v1beta1", "v1beta2"]
    resources: ["deployments", "replicasets", "replicationcontrollers", "statefulsets", "daemonsets", "pods"]
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail
  timeoutSeconds: 10
```

## Resource Security

### Security Contexts
```yaml
# Secure Central deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: central
  namespace: stackrox
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 4000
        runAsGroup: 4000
        fsGroup: 4000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: central
        image: stackrox/main:latest
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 4000
          runAsGroup: 4000
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            memory: "4Gi"
            cpu: "1500m"
          limits:
            memory: "8Gi"
            cpu: "4000m"
        livenessProbe:
          httpGet:
            path: /v1/ping
            port: 8080
            scheme: HTTPS
          initialDelaySeconds: 30
          timeoutSeconds: 10
        readinessProbe:
          httpGet:
            path: /v1/ping
            port: 8080
            scheme: HTTPS
          initialDelaySeconds: 10
          timeoutSeconds: 5
```

### Pod Security Standards
```yaml
# Restricted pod security policy for ACS namespace
apiVersion: v1
kind: Namespace
metadata:
  name: stackrox
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Pod security policy for ACS components
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: stackrox-restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  runAsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1000
        max: 65535
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1000
        max: 65535
  readOnlyRootFilesystem: true
```

## Data Protection

### Database Encryption
```yaml
# Encrypted PostgreSQL for Central
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: central-db
  namespace: stackrox
spec:
  template:
    spec:
      containers:
      - name: db
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: stackrox
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD_FILE
          value: /run/secrets/db-password/password
        - name: POSTGRES_INITDB_ARGS
          value: "--auth-host=scram-sha-256 --auth-local=scram-sha-256"
        command:
        - docker-entrypoint.sh
        - postgres
        - -c
        - ssl=on
        - -c
        - ssl_cert_file=/run/secrets/tls/tls.crt
        - -c
        - ssl_key_file=/run/secrets/tls/tls.key
        - -c
        - ssl_ca_file=/run/secrets/ca/ca.crt
        - -c
        - log_statement=all
        - -c
        - log_directory=/var/log/postgresql
        - -c
        - logging_collector=on
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: tls
          mountPath: /run/secrets/tls
        - name: ca
          mountPath: /run/secrets/ca
        - name: db-password
          mountPath: /run/secrets/db-password
      volumes:
      - name: tls
        secret:
          secretName: central-db-tls
      - name: ca
        secret:
          secretName: central-ca
      - name: db-password
        secret:
          secretName: central-db-password
```

### Backup Encryption
```bash
#!/bin/bash
# encrypted-backup.sh

# Create encrypted backup of Central database
export PGPASSWORD=$(kubectl get secret central-db-password -n stackrox -o jsonpath='{.data.password}' | base64 -d)

# Dump database
kubectl exec central-db-0 -n stackrox -- pg_dump -U postgres stackrox | \
    gpg --symmetric --cipher-algo AES256 --compress-algo 1 --s2k-mode 3 \
        --s2k-digest-algo SHA512 --s2k-count 65536 --force-mdc \
        --passphrase-file /run/secrets/backup-key/passphrase > \
        "central-backup-$(date +%Y%m%d-%H%M%S).sql.gpg"

# Upload to secure storage
aws s3 cp "central-backup-$(date +%Y%m%d-%H%M%S).sql.gpg" \
    s3://company-acs-backups/ \
    --server-side-encryption aws:kms \
    --ssekms-key-id alias/acs-backup-key
```

## Monitoring and Auditing

### Audit Logging Configuration
```yaml
# Fluent Bit configuration for ACS log collection
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: stackrox
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Path              /var/log/stackrox/*.log
        Parser            json
        Tag               stackrox.*
        Refresh_Interval  5

    [OUTPUT]
        Name              forward
        Match             stackrox.*
        Host              splunk-forwarder.logging.svc.cluster.local
        Port              24224
        tls               on
        tls.verify        on
        tls.ca_file       /fluent-bit/ssl/ca.crt
        tls.crt_file      /fluent-bit/ssl/client.crt
        tls.key_file      /fluent-bit/ssl/client.key

  parsers.conf: |
    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

### Security Metrics Collection
```yaml
# ServiceMonitor for Prometheus metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: stackrox-central
  namespace: stackrox
spec:
  selector:
    matchLabels:
      app: central
  endpoints:
  - port: monitoring
    interval: 30s
    path: /metrics
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
---
# PrometheusRule for ACS alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: stackrox-alerts
  namespace: stackrox
spec:
  groups:
  - name: stackrox
    rules:
    - alert: StackRoxCentralDown
      expr: up{job="stackrox-central"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "StackRox Central is down"
        description: "StackRox Central has been down for more than 5 minutes"

    - alert: HighVulnerabilityCount
      expr: stackrox_vulnerability_count{severity="CRITICAL"} > 10
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High number of critical vulnerabilities detected"
        description: "More than 10 critical vulnerabilities found in running workloads"
```

## Secrets Management

### External Secrets Integration
```yaml
# External Secrets Operator configuration
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: stackrox
spec:
  provider:
    vault:
      server: "https://vault.company.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "stackrox"
          serviceAccountRef:
            name: "external-secrets-sa"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: central-secrets
  namespace: stackrox
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: central-secrets
    creationPolicy: Owner
  data:
  - secretKey: admin-password
    remoteRef:
      key: stackrox/central
      property: admin-password
  - secretKey: db-password
    remoteRef:
      key: stackrox/database
      property: password
  - secretKey: oidc-client-secret
    remoteRef:
      key: stackrox/auth
      property: oidc-secret
```

## Compliance Configuration

### CIS Benchmark Compliance
```bash
#!/bin/bash
# cis-compliance-check.sh

# Run CIS Kubernetes benchmark compliance check
roxctl compliance run --standard CIS_Kubernetes_v1_5 \
    --cluster production-cluster \
    --output json > cis-compliance-report.json

# Parse results and generate summary
PASSED=$(jq '.results[].checks[] | select(.state == "PASS") | length' cis-compliance-report.json | wc -l)
FAILED=$(jq '.results[].checks[] | select(.state == "FAIL") | length' cis-compliance-report.json | wc -l)
TOTAL=$((PASSED + FAILED))

echo "CIS Kubernetes Benchmark Compliance Summary:"
echo "Passed: $PASSED"
echo "Failed: $FAILED"
echo "Total: $TOTAL"
echo "Compliance Rate: $(echo "scale=2; $PASSED / $TOTAL * 100" | bc)%"

# Fail pipeline if compliance rate is below threshold
THRESHOLD=95
COMPLIANCE_RATE=$(echo "scale=0; $PASSED / $TOTAL * 100" | bc)

if [ "$COMPLIANCE_RATE" -lt "$THRESHOLD" ]; then
    echo "FAILED: Compliance rate ($COMPLIANCE_RATE%) below threshold ($THRESHOLD%)"
    exit 1
fi

echo "PASSED: Compliance rate meets requirements"
```

This hardening configuration provides comprehensive security measures for ACS deployments across authentication, network security, resource protection, and compliance monitoring.