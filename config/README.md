# ACS Configuration Guidelines

This directory contains best practice configuration guides for Red Hat Advanced Cluster Security (ACS) components.

## Configuration Guides

### Installation and Setup
- [Initial Installation Guide](installation/initial-setup.md) - Step-by-step ACS installation
- [High Availability Setup](installation/ha-setup.md) - Configuring ACS for high availability
- [Upgrade Procedures](installation/upgrade-guide.md) - Safe upgrade practices

### Component Configuration
- [Central Configuration](components/central-config.md) - ACS Central best practices
- [Sensor Configuration](components/sensor-config.md) - Sensor deployment and tuning
- [Scanner Configuration](components/scanner-config.md) - Image scanner optimization
- [Collector Configuration](components/collector-config.md) - Runtime data collection tuning

### Security Hardening
- [TLS Configuration](security/tls-config.md) - Certificate management and TLS setup
- [Network Security](security/network-config.md) - Network policies and firewall rules
- [Authentication Setup](security/auth-config.md) - SSO and RBAC configuration
- [Secrets Management](security/secrets-config.md) - Secure secrets handling

### Performance and Scaling
- [Resource Planning](performance/resource-planning.md) - Capacity planning guidelines
- [Performance Tuning](performance/tuning-guide.md) - Optimization recommendations
- [Scaling Guidelines](performance/scaling-guide.md) - Horizontal and vertical scaling

### Integration
- [CI/CD Integration](integration/cicd-integration.md) - DevSecOps pipeline integration
- [SIEM Integration](integration/siem-integration.md) - Log forwarding and alerting
- [Monitoring Integration](integration/monitoring-setup.md) - Metrics and dashboards

## Quick Start Configurations

### Minimal Production Setup
```yaml
# Basic Central configuration for production
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
  scanner:
    analyzer:
      scaling:
        autoScaling: ENABLED
        maxReplicas: 5
        minReplicas: 2
```

### Security-Hardened Configuration
```yaml
# Enhanced security configuration
apiVersion: platform.stackrox.io/v1alpha1
kind: Central
metadata:
  name: stackrox-central-services
  namespace: stackrox
spec:
  central:
    defaultTLSSecret:
      name: central-tls
    exposure:
      route:
        enabled: true
    persistence:
      persistentVolumeClaim:
        claimName: central-db
        storageClass: encrypted-ssd
  scanner:
    scannerComponent: Enabled
```

## Configuration Validation

Use the following commands to validate your configuration:

```bash
# Validate Central installation
roxctl central check

# Test sensor connectivity
roxctl sensor get-bundle <cluster-name>

# Verify scanner functionality
roxctl image scan --image <test-image>

# Check policy status
roxctl policy list
```

## Environment-Specific Configurations

### Development Environment
- Reduced resource allocation
- Relaxed security policies for testing
- Enhanced logging for debugging

### Staging Environment
- Production-like security policies
- Full monitoring and alerting
- Performance testing configurations

### Production Environment
- Maximum security hardening
- High availability setup
- Optimized performance settings
- Comprehensive backup strategy

## Troubleshooting

Common configuration issues and solutions:

1. **Sensor connectivity issues**: Check network policies and firewall rules
2. **Scanner performance**: Adjust resource limits and scaling parameters
3. **Central UI access**: Verify ingress/route configuration and certificates
4. **Policy violations**: Review exclusions and policy tuning

For detailed troubleshooting guides, see the individual configuration files in each subdirectory.