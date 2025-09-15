# ACS Security Audit Quick Reference

This quick reference provides essential commands and checklists specifically for security auditing of Red Hat Advanced Cluster Security (ACS) deployments.

## Security Audit CLI Commands

### Authentication and Access Verification
```bash
# Download roxctl CLI for auditing
curl -O https://mirror.openshift.com/pub/rhacs/assets/latest/bin/Linux/roxctl
chmod +x roxctl && sudo mv roxctl /usr/local/bin/

# Configure audit session
export ROX_CENTRAL_ENDPOINT=central.example.com:443
export ROX_API_TOKEN=your-audit-token

# Verify audit access
roxctl central whoami

# Check authentication status  
roxctl auth status
```

### Security Audit - Image Vulnerability Assessment
```bash
# Audit image vulnerabilities
roxctl image scan --image nginx:latest --severity CRITICAL,HIGH

# Generate audit report for image
roxctl image scan --image nginx:latest --output json > image-audit-report.json

# Check for specific CVEs during audit
roxctl image scan --image nginx:latest --output table | grep CVE-

# Audit multiple images for compliance
for image in $(cat image-list.txt); do
  roxctl image scan --image $image --severity CRITICAL,HIGH >> audit-results.txt
done
```

### Security Audit - Policy Compliance Review
```bash
# Audit all security policies
roxctl policy list --output json > policy-audit.json

# Check policy enforcement status
roxctl policy list --enabled

# Audit policy violations
roxctl violation list --severity CRITICAL,HIGH --output json

# Review policy exclusions (audit risk)
roxctl policy exclusions list

# Validate policy configuration
roxctl policy check --file policy.yaml
```

# Check deployment against policies
roxctl deployment check --file deployment.yaml
```

### Security Audit - Cluster and Sensor Status
```bash
# Audit cluster security status  
roxctl cluster list --output json > cluster-audit.json

# Verify sensor deployment security
kubectl get pods -n stackrox -l app=sensor -o wide

# Check sensor communication security
roxctl sensor status --cluster cluster-name

# Audit collector security status
kubectl get daemonset -n stackrox collector
```

### Security Audit - Compliance and Vulnerability Assessment
```bash
# Run security compliance audit
roxctl compliance run --standard CIS_Kubernetes_v1_5

# Export compliance audit results
roxctl compliance export --standard CIS_Kubernetes_v1_5 --output compliance-audit.json

# Generate vulnerability audit report
roxctl vuln-mgmt export --output vulnerability-audit.csv

# Audit network security violations
roxctl violation list --type NETWORK_POLICY --output json

# Check runtime security violations
roxctl violation list --type RUNTIME --severity HIGH,CRITICAL
```

## Security Audit Checklist (Quick Assessment)

### ✅ Critical Security Audit Points
- [ ] ACS Central accessible only via HTTPS with valid TLS certificate
- [ ] Multi-factor authentication enabled for all admin accounts
- [ ] No default passwords or weak authentication mechanisms
- [ ] Critical and high severity vulnerability policies enforced
- [ ] Privileged container policies active and blocking violations
- [ ] Network policies configured and blocking unauthorized traffic
- [ ] Runtime monitoring active and collecting security events
- [ ] Security violation alerts configured and functional

### ⚠️ Security Audit Red Flags
- [ ] Admin accounts without MFA enabled
- [ ] Service accounts with excessive permissions
- [ ] Unencrypted sensor-to-central communication
- [ ] Unrestricted external registry access
- [ ] High number of unresolved security violations
- [ ] Missing or disabled critical security policies
- [ ] Audit logging disabled or incomplete
- [ ] Backup and recovery procedures untested

## Security Audit - Key Policy Validation

### Critical Security Policies to Audit
Use these commands to verify essential security policies are in place:

```bash
# Verify latest tag policy is enforced
roxctl policy list | grep -i "latest.*tag"

# Check privileged container blocking
roxctl policy list | grep -i "privileged"

# Validate resource limit enforcement  
roxctl policy list | grep -i "resource.*limit"

# Verify image scanning policies
roxctl policy list | grep -i "image.*scan"
```

### Audit Policy Examples for Verification

#### 1. Latest Image Tag Prevention (Critical)
Verify this policy exists and is enabled:
```bash
roxctl policy get --name "No Latest Image Tag"
```

#### 2. Privileged Container Blocking (Critical)
Verify this policy exists and blocks privileged containers:
```bash
roxctl policy get --name "Privileged Container"
```

#### 3. Resource Limits Enforcement (High)
Verify this policy requires resource limits:
```bash
roxctl policy get --name "Required Resource Limits"
## Security Audit - Network Security Verification

### Network Policy Audit Commands
```bash
# Audit network policies 
kubectl get networkpolicies --all-namespaces -o wide

# Check ACS component network isolation
kubectl get networkpolicy -n stackrox

# Verify default deny policies exist
kubectl get networkpolicy default-deny-all

# Audit network violations
roxctl violation list --type NETWORK_POLICY --output json
```

### Critical Network Security Audit Points
- [ ] Default deny-all network policy implemented
- [ ] ACS components have restrictive network policies
- [ ] Ingress traffic is controlled and monitored
- [ ] Inter-namespace communication is restricted
- [ ] External network access is limited and audited

## Security Audit - Troubleshooting and Log Analysis

### Security-Focused Troubleshooting

| Security Issue | Symptoms | Audit Command |
|---|---|---|
| Policy bypass | Violations not blocked | `roxctl policy list --disabled` |
| Authentication failure | Access denied errors | `roxctl auth status` |
| Scanner vulnerabilities | Missing CVE data | `kubectl logs -l app=scanner -n stackrox` |
| Sensor disconnection | No runtime data | `roxctl sensor status --cluster <name>` |
| TLS certificate issues | Connection errors | `roxctl central db status` |

### Security Log Collection for Audit
```bash
# Collect security violation logs
roxctl violation list --output json > security-violations.json

# Collect policy enforcement logs  
kubectl logs -l app=admission-controller -n stackrox > policy-enforcement.log

# Collect authentication audit logs
kubectl logs -l app=central -n stackrox | grep -i auth > auth-audit.log

# Check runtime security events
kubectl logs -l app=collector -n stackrox | grep -i security > runtime-security.log
```

## Security Audit - Critical Security Validation

### Authentication & Authorization Audit
- [ ] SSO/OIDC properly configured and enforced
- [ ] No default admin passwords in use
- [ ] Service accounts follow principle of least privilege
- [ ] API tokens have appropriate expiration and rotation
- [ ] Role-based access control (RBAC) properly implemented
- [ ] Audit logs capture all authentication events

### Network Security Audit
- [ ] TLS certificates are valid and properly configured
- [ ] Network policies restrict inter-pod communication appropriately
- [ ] External access is properly controlled and monitored
- [ ] Internal traffic encryption is enforced
- [ ] Network segmentation is implemented correctly

### Data Protection Audit  
- [ ] Database encryption at rest is enabled and verified
- [ ] Backup encryption is configured and tested
- [ ] Secrets are stored securely (not in plain text)
- [ ] Audit logs are protected and retained per policy
- [ ] Data classification policies are applied and enforced

### Policy Enforcement Audit
- [ ] Critical security policies are enabled and enforcing
- [ ] Policy violations are properly detected and blocked
- [ ] Policy exclusions are justified and documented
- [ ] Runtime security monitoring is active and alerting
- [ ] Compliance frameworks are properly implemented

## Security Incident Response for Auditors

### Security Incident Detection Commands
```bash
# Check for active security violations
roxctl violation list --state ACTIVE --severity CRITICAL,HIGH

# Audit runtime security events
roxctl alert list --type RUNTIME --output json

# Review policy enforcement failures
roxctl violation list --type POLICY_VIOLATION --output table

# Check for compliance violations
roxctl compliance run --standard CIS_Kubernetes_v1_5
```

### Security Audit Documentation

For comprehensive security audit procedures, refer to:
- [ACS Audit Checklist](ACS-Audit-Checklist.md) - Complete 200+ point security audit
- [Security Hardening Guide](config/security/security-hardening.md) - Security configuration verification
- [Compliance Report Template](templates/compliance-report-template.md) - Audit reporting format

### Security Audit References

#### Standards and Frameworks
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [OWASP Kubernetes Security](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)

#### Official ACS Documentation  
- [Red Hat ACS Security Guide](https://docs.openshift.com/acs/operating/manage-user-access/configure-short-lived-access.html)
- [ACS CLI Reference](https://docs.openshift.com/acs/cli/roxctl.html)

---

*This reference provides essential commands and checklists for conducting comprehensive security audits of Red Hat Advanced Cluster Security deployments.*