# ACS Security Quick Reference

This quick reference provides essential commands, configurations, and checklists for Red Hat Advanced Cluster Security (ACS) operations.

## Essential CLI Commands

### Installation and Setup
```bash
# Download roxctl CLI
curl -O https://mirror.openshift.com/pub/rhacs/assets/latest/bin/Linux/roxctl
chmod +x roxctl && sudo mv roxctl /usr/local/bin/

# Configure authentication
export ROX_CENTRAL_ENDPOINT=central.example.com:443
export ROX_API_TOKEN=your-api-token

# Test connectivity
roxctl central whoami
```

### Image Security
```bash
# Scan an image
roxctl image scan --image nginx:latest

# Scan with specific output format
roxctl image scan --image nginx:latest --output table
roxctl image scan --image nginx:latest --output json

# Check image for specific vulnerabilities
roxctl image scan --image nginx:latest --severity CRITICAL,IMPORTANT

# Scan image and save results
roxctl image scan --image nginx:latest --output json > scan-results.json
```

### Policy Management
```bash
# List all policies
roxctl policy list

# Import a policy
roxctl policy import --file policy.yaml

# Export policies
roxctl policy export --output policies.json

# Disable a policy
roxctl policy patch --name "Policy Name" --disabled true

# Check deployment against policies
roxctl deployment check --file deployment.yaml
```

### Cluster and Sensor Management
```bash
# Generate sensor bundle
roxctl sensor generate k8s --name cluster-name \
  --central-endpoint central.example.com:443 \
  --output-dir ./sensor-bundle

# List clusters
roxctl cluster list

# Delete a cluster
roxctl cluster delete --name cluster-name

# Check sensor status
kubectl get pods -n stackrox -l app=sensor
```

### Compliance and Reporting
```bash
# Run compliance scan
roxctl compliance run --standard CIS_Kubernetes_v1_5

# Export compliance results
roxctl compliance export --standard CIS_Kubernetes_v1_5 --output results.json

# Generate vulnerability report
roxctl vuln-mgmt export --output vulns.csv
```

## Critical Security Checklist (5-Minute Review)

### ✅ Essential Security Checks
- [ ] ACS Central is accessible via HTTPS with valid certificate
- [ ] Multi-factor authentication is enabled
- [ ] Critical vulnerability policies are enforced
- [ ] Privileged container policies are active
- [ ] Network policies are blocking unauthorized traffic
- [ ] Runtime monitoring is collecting data
- [ ] Backup procedures are tested and functional

### ⚠️ High-Risk Configuration Issues
- [ ] Default passwords are changed
- [ ] Admin users have MFA enabled
- [ ] Service accounts have minimal permissions
- [ ] Sensor-to-Central communication is encrypted
- [ ] External registry access is restricted
- [ ] Policy violations are being monitored
- [ ] Audit logs are being collected

## Common Policy Templates

### Block Latest Tags
```yaml
# Prevent deployment of images with 'latest' tag
apiVersion: v1
kind: Policy
metadata:
  name: "No Latest Image Tag"
spec:
  severity: "HIGH_SEVERITY"
  lifecycleStages: ["BUILD", "DEPLOY"]
  policySections:
  - policyGroups:
    - fieldName: "Image Tag"
      values: [{"value": "latest"}]
```

### Require Resource Limits
```yaml
# Enforce CPU and memory limits
apiVersion: v1
kind: Policy
metadata:
  name: "Required Resource Limits"
spec:
  severity: "MEDIUM_SEVERITY"
  lifecycleStages: ["DEPLOY"]
  policySections:
  - policyGroups:
    - fieldName: "Memory Limit"
      negate: true
      values: [{"value": ".*"}]
```

### Block Privileged Containers
```yaml
# Prevent privileged container execution
apiVersion: v1
kind: Policy
metadata:
  name: "No Privileged Containers"
spec:
  severity: "HIGH_SEVERITY"
  lifecycleStages: ["DEPLOY"]
  policySections:
  - policyGroups:
    - fieldName: "Privileged Container"
      values: [{"value": "true"}]
```

## Network Security Quick Config

### Basic Network Policy Template
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### ACS Component Network Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: stackrox-network-policy
  namespace: stackrox
spec:
  podSelector:
    matchLabels:
      app: central
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 8443
```

## Troubleshooting Quick Reference

### Common Issues and Solutions

| Issue | Symptoms | Quick Fix |
|-------|----------|-----------|
| Sensor offline | No data in dashboard | Check sensor pods: `kubectl get pods -n stackrox` |
| Policy violations not showing | No alerts generated | Verify admission controller: `kubectl get validatingwebhookconfiguration` |
| Image scans failing | Scanner errors | Check scanner logs: `kubectl logs -l app=scanner -n stackrox` |
| Central UI inaccessible | Connection refused | Check Central service: `kubectl get svc -n stackrox` |
| Database connection issues | Central startup failures | Verify database pod: `kubectl get pods -l app=central-db -n stackrox` |

### Log Collection Commands
```bash
# Collect all ACS logs
kubectl logs -l app=central -n stackrox --previous > central.log
kubectl logs -l app=sensor -n stackrox --previous > sensor.log
kubectl logs -l app=scanner -n stackrox --previous > scanner.log
kubectl logs -l app=collector -n stackrox --previous > collector.log

# Get resource usage
kubectl top pods -n stackrox

# Check events
kubectl get events -n stackrox --sort-by='.lastTimestamp'
```

## Performance Optimization

### Resource Recommendations

| Component | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|-------------|----------------|-----------|--------------|
| Central | 1000m | 4Gi | 4000m | 8Gi |
| Scanner | 500m | 2Gi | 2000m | 4Gi |
| Sensor | 100m | 500Mi | 500m | 1Gi |
| Collector | 50m | 100Mi | 200m | 500Mi |

### Scaling Guidelines
```bash
# Scale scanner for high-volume scanning
kubectl patch deployment scanner -n stackrox -p '{"spec":{"replicas":3}}'

# Horizontal Pod Autoscaler for scanner
kubectl autoscale deployment scanner -n stackrox --cpu-percent=70 --min=2 --max=10
```

## Security Hardening Checklist

### Authentication & Authorization
- [ ] SSO/OIDC configured with corporate identity provider
- [ ] Default admin password changed
- [ ] Service accounts follow principle of least privilege
- [ ] API tokens have appropriate expiration
- [ ] Role-based access control (RBAC) implemented

### Network Security
- [ ] TLS certificates are valid and properly configured
- [ ] Network policies restrict inter-pod communication
- [ ] Firewall rules limit external access
- [ ] Load balancer configured with security headers
- [ ] Internal traffic is encrypted

### Data Protection
- [ ] Database encryption at rest enabled
- [ ] Backup encryption configured
- [ ] Secrets stored in external secret management
- [ ] Audit logs protected and retained
- [ ] Data classification policies applied

### Monitoring & Alerting
- [ ] Security metrics exported to monitoring system
- [ ] Critical alerts configured and tested
- [ ] Log forwarding to SIEM implemented
- [ ] Incident response procedures documented
- [ ] Regular security reviews scheduled

## Emergency Response

### Incident Response Commands
```bash
# Quickly disable a problematic policy
roxctl policy patch --name "Policy Name" --disabled true

# Get all active violations
roxctl alert list --output json | jq '.alerts[] | select(.state == "ACTIVE")'

# Emergency cluster isolation
kubectl patch networkpolicy default-deny-all -p '{"spec":{"ingress":[],"egress":[]}}'

# Collect forensic data
kubectl get pods --all-namespaces -o wide > pods-snapshot.txt
kubectl get events --all-namespaces --sort-by='.lastTimestamp' > events-snapshot.txt
```

### Emergency Contacts Template
```
Security Team: security@company.com
On-call Engineer: +1-XXX-XXX-XXXX
Incident Commander: +1-XXX-XXX-XXXX
Legal/Compliance: legal@company.com
```

## Useful URLs and References

### Official Documentation
- [Red Hat ACS Documentation](https://docs.openshift.com/acs/)
- [ACS API Reference](https://docs.openshift.com/acs/operating/manage-user-access/configure-short-lived-access.html)
- [ACS CLI Reference](https://docs.openshift.com/acs/cli/roxctl.html)

### Security Standards
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [OWASP Kubernetes Security](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)

### Tools and Utilities
- [Kubernetes Security Tools](https://kubernetes.io/docs/concepts/security/)
- [YAML Lint](https://www.yamllint.com/)
- [JSON Formatter](https://jsonformatter.org/)

---

*Keep this reference handy for quick ACS operations and security checks.*