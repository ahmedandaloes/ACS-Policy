# ACS Security Policy Reference Templates

This directory contains reference security policy templates for Red Hat Advanced Cluster Security (ACS) security audit verification. These policies serve as examples of critical security configurations that should be validated during security audits.

## Security Policy Audit Categories

### Build-Time Security Policies (Audit Verification)
- [Latest Image Tag Policy](build-time/latest-image-tag-policy.yaml) - Verify prevention of 'latest' tag deployments
- [Dockerfile Security Policy](build-time/dockerfile-security-policy.yaml) - Validate secure Dockerfile practices enforcement
- [Base Image Policy](build-time/base-image-policy.yaml) - Audit base image restriction controls

### Deploy-Time Security Policies (Audit Verification)  
- [Privileged Container Policy](deploy-time/privileged-container-policy.yaml) - Verify privileged container blocking
- [Resource Limits Policy](deploy-time/resource-limits-policy.yaml) - Audit resource limit enforcement
- [Security Context Policy](deploy-time/security-context-policy.yaml) - Validate security context requirements

### Runtime Security Policies (Audit Verification)
- [Process Execution Policy](runtime/process-execution-policy.yaml) - Audit runtime process monitoring
- [Network Activity Policy](runtime/network-activity-policy.yaml) - Verify network connection monitoring  
- [File System Policy](runtime/filesystem-policy.yaml) - Validate file system access controls

## Security Audit Usage

### Policy Validation Commands
```bash
# Verify policy exists and is enabled
roxctl policy get --name "Policy Name"

# Check policy enforcement status
roxctl policy list --enabled | grep "Policy Name"

# Validate policy configuration
roxctl policy check --file <policy-file.yaml>

# Audit policy violations
roxctl violation list --policy "Policy Name"
```

## Security Audit Best Practices

### Critical Policy Audit Checks
- Verify all critical security policies are enabled and enforcing
- Validate policy configurations match security requirements
- Review policy exclusions for security risks and proper justification
- Audit policy violation patterns and remediation status
- Ensure policy enforcement is consistent across all environments

### Audit Documentation
- Document all security policy verification results
- Track any missing or misconfigured security policies
- Record policy exclusions and their security justifications  
- Maintain audit trail of policy compliance status
- Report policy gaps and remediation recommendations