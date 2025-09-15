# ACS Policy Templates

This directory contains ready-to-use policy templates for Red Hat Advanced Cluster Security (ACS). These policies can be imported directly into your ACS environment or customized based on your specific requirements.

## Policy Categories

### Build-Time Policies
- [Latest Image Tag Policy](build-time/latest-image-tag-policy.yaml) - Prevents deployment of images with 'latest' tag
- [Dockerfile Security Policy](build-time/dockerfile-security-policy.yaml) - Enforces secure Dockerfile practices
- [Base Image Policy](build-time/base-image-policy.yaml) - Restricts allowed base images

### Deploy-Time Policies
- [Privileged Container Policy](deploy-time/privileged-container-policy.yaml) - Blocks privileged containers
- [Resource Limits Policy](deploy-time/resource-limits-policy.yaml) - Enforces resource limits
- [Security Context Policy](deploy-time/security-context-policy.yaml) - Enforces secure security contexts

### Runtime Policies
- [Process Execution Policy](runtime/process-execution-policy.yaml) - Monitors and controls process execution
- [Network Activity Policy](runtime/network-activity-policy.yaml) - Monitors network connections
- [File System Policy](runtime/filesystem-policy.yaml) - Monitors file system access

## Usage

1. Review each policy template for your environment requirements
2. Customize policy parameters as needed
3. Import policies into ACS using the CLI or Web UI:
   ```bash
   roxctl policy import --file <policy-file.yaml>
   ```

## Policy Management Best Practices

- Test policies in development environments before production deployment
- Use policy exclusions sparingly and document justifications
- Regularly review and update policies based on threat landscape
- Monitor policy violations and adjust thresholds as needed
- Maintain version control for policy changes