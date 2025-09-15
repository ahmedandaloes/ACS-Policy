# ACS Security Audit Configuration

This directory contains security-focused configuration guidelines for auditing Red Hat Advanced Cluster Security (ACS) deployments.

## Security Audit Configuration

### Security Hardening Verification
- [Security Hardening Checklist](security/security-hardening.md) - Security configuration verification

## Security Audit Quick Configurations

### Security Validation Commands
```bash
# Verify Central security configuration
roxctl central whoami

# Check TLS configuration
roxctl central db status

# Validate policy compliance
roxctl policy list --enabled

# Check for security violations
roxctl violation list --severity CRITICAL,HIGH
```

## Security Audit Validation

Use the following commands to validate security configurations during audit:

```bash
# Check Central installation security
roxctl central check

# Validate sensor security posture
roxctl sensor status

# Verify image scanning is enabled
roxctl image scan --image <test-image>

# Check policy enforcement status
roxctl policy list --enabled
```

## Security Audit Focus

This configuration is specifically designed for security auditing:

### Security Audit Areas
- TLS/SSL certificate validation
- Authentication and authorization verification  
- Network security configuration review
- Policy enforcement verification
- Compliance status checking

## Common Security Audit Issues

Security misconfigurations found during audits:

1. **Weak TLS configuration**: Verify certificate validity and encryption strength
2. **Inadequate access controls**: Review RBAC and authentication mechanisms  
3. **Policy gaps**: Check for missing or disabled security policies
4. **Violation monitoring**: Ensure security violations are properly tracked and remediated

For detailed security audit procedures, see the [ACS Audit Checklist](../ACS-Audit-Checklist.md).