# CI/CD Integration Guide

This guide demonstrates how to integrate Red Hat Advanced Cluster Security (ACS) into your CI/CD pipelines for comprehensive DevSecOps security.

## Overview

ACS provides multiple integration points in the software development lifecycle:
- **Build Stage**: Image vulnerability scanning and policy checks
- **Deploy Stage**: Deployment-time policy validation
- **Runtime Stage**: Continuous monitoring and threat detection

## Integration Methods

### 1. CLI Integration (roxctl)

#### Installation
```bash
# Download roxctl CLI
curl -O https://mirror.openshift.com/pub/rhacs/assets/latest/bin/Linux/roxctl
chmod +x roxctl && sudo mv roxctl /usr/local/bin/

# Configure authentication
export ROX_CENTRAL_ENDPOINT=central.example.com:443
export ROX_API_TOKEN=<your-api-token>
```

#### Image Scanning in CI Pipeline
```bash
#!/bin/bash
# scan-image.sh

set -e

IMAGE_NAME=$1
SEVERITY_THRESHOLD=${2:-IMPORTANT}

echo "Scanning image: $IMAGE_NAME"

# Perform image scan
SCAN_RESULT=$(roxctl image scan --image="$IMAGE_NAME" --output=json)

# Check for vulnerabilities above threshold
CRITICAL_COUNT=$(echo "$SCAN_RESULT" | jq '.scan.components[].vulns[] | select(.severity == "CRITICAL") | length' | wc -l)
HIGH_COUNT=$(echo "$SCAN_RESULT" | jq '.scan.components[].vulns[] | select(.severity == "IMPORTANT") | length' | wc -l)

echo "Found $CRITICAL_COUNT critical and $HIGH_COUNT high severity vulnerabilities"

# Fail pipeline if critical vulnerabilities found
if [ "$CRITICAL_COUNT" -gt 0 ]; then
    echo "FAILED: Critical vulnerabilities found"
    exit 1
elif [ "$HIGH_COUNT" -gt 5 ] && [ "$SEVERITY_THRESHOLD" = "IMPORTANT" ]; then
    echo "FAILED: Too many high severity vulnerabilities"
    exit 1
fi

echo "PASSED: Image scan successful"
```

#### Deployment Policy Check
```bash
#!/bin/bash
# check-deployment.sh

DEPLOYMENT_FILE=$1

echo "Checking deployment policies for: $DEPLOYMENT_FILE"

# Check deployment against policies
roxctl deployment check --file="$DEPLOYMENT_FILE" --output=json > policy_check.json

# Parse results
VIOLATIONS=$(jq '.alerts | length' policy_check.json)

if [ "$VIOLATIONS" -gt 0 ]; then
    echo "FAILED: $VIOLATIONS policy violations found"
    jq '.alerts[] | .policy.name' policy_check.json
    exit 1
fi

echo "PASSED: No policy violations found"
```

### 2. GitHub Actions Integration

#### Workflow Example
```yaml
# .github/workflows/security-scan.yml
name: ACS Security Scan

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
        
    - name: Install roxctl
      run: |
        curl -O https://mirror.openshift.com/pub/rhacs/assets/latest/bin/Linux/roxctl
        chmod +x roxctl
        sudo mv roxctl /usr/local/bin/
        
    - name: Scan image with ACS
      env:
        ROX_CENTRAL_ENDPOINT: ${{ secrets.ROX_CENTRAL_ENDPOINT }}
        ROX_API_TOKEN: ${{ secrets.ROX_API_TOKEN }}
      run: |
        roxctl image scan --image=myapp:${{ github.sha }} \
          --output=table \
          --severity=CRITICAL,IMPORTANT
          
    - name: Check deployment policies
      env:
        ROX_CENTRAL_ENDPOINT: ${{ secrets.ROX_CENTRAL_ENDPOINT }}
        ROX_API_TOKEN: ${{ secrets.ROX_API_TOKEN }}
      run: |
        roxctl deployment check \
          --file=k8s/deployment.yaml \
          --output=table
          
    - name: Generate security report
      if: always()
      env:
        ROX_CENTRAL_ENDPOINT: ${{ secrets.ROX_CENTRAL_ENDPOINT }}
        ROX_API_TOKEN: ${{ secrets.ROX_API_TOKEN }}
      run: |
        roxctl image scan --image=myapp:${{ github.sha }} \
          --output=json > security-report.json
          
    - name: Upload security report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: security-report
        path: security-report.json
```

### 3. Jenkins Integration

#### Pipeline Script
```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        ROX_CENTRAL_ENDPOINT = credentials('rox-central-endpoint')
        ROX_API_TOKEN = credentials('rox-api-token')
        IMAGE_NAME = "myapp:${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Build') {
            steps {
                script {
                    docker.build(IMAGE_NAME)
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    // Install roxctl if not available
                    sh '''
                        if ! command -v roxctl &> /dev/null; then
                            curl -O https://mirror.openshift.com/pub/rhacs/assets/latest/bin/Linux/roxctl
                            chmod +x roxctl
                            sudo mv roxctl /usr/local/bin/
                        fi
                    '''
                    
                    // Scan image
                    def scanResult = sh(
                        script: "roxctl image scan --image=${IMAGE_NAME} --output=json",
                        returnStdout: true
                    )
                    
                    // Parse and evaluate results
                    writeFile file: 'scan-result.json', text: scanResult
                    
                    def violations = sh(
                        script: "jq '.scan.components[].vulns[] | select(.severity == \"CRITICAL\") | length' scan-result.json | wc -l",
                        returnStdout: true
                    ).trim() as Integer
                    
                    if (violations > 0) {
                        error("Critical vulnerabilities found: ${violations}")
                    }
                }
            }
        }
        
        stage('Policy Check') {
            steps {
                script {
                    sh """
                        roxctl deployment check \
                            --file=k8s/deployment.yaml \
                            --output=json > policy-check.json
                    """
                    
                    def policyViolations = sh(
                        script: "jq '.alerts | length' policy-check.json",
                        returnStdout: true
                    ).trim() as Integer
                    
                    if (policyViolations > 0) {
                        error("Policy violations found: ${policyViolations}")
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Deploy to production
                    sh 'kubectl apply -f k8s/'
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '*.json', allowEmptyArchive: true
            
            script {
                // Generate and publish security report
                def report = readJSON file: 'scan-result.json'
                
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'security-report.html',
                    reportName: 'ACS Security Report'
                ])
            }
        }
        
        failure {
            emailext(
                subject: "Security scan failed for ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "Security vulnerabilities or policy violations found. Check the build logs for details.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
```

### 4. GitLab CI Integration

#### .gitlab-ci.yml Example
```yaml
stages:
  - build
  - security-scan
  - deploy

variables:
  IMAGE_NAME: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  ROX_CENTRAL_ENDPOINT: "central.example.com:443"

build:
  stage: build
  script:
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
  only:
    - branches

security-scan:
  stage: security-scan
  image: 
    name: quay.io/stackrox-io/roxctl:latest
    entrypoint: [""]
  script:
    - export ROX_API_TOKEN=$ROX_API_TOKEN_SECRET
    - |
      echo "Scanning image: $IMAGE_NAME"
      roxctl image scan --image=$IMAGE_NAME --output=json > scan-results.json
      
      # Check for critical vulnerabilities
      CRITICAL_VULNS=$(jq '.scan.components[].vulns[] | select(.severity == "CRITICAL") | length' scan-results.json | wc -l)
      
      if [ "$CRITICAL_VULNS" -gt 0 ]; then
        echo "FAILED: $CRITICAL_VULNS critical vulnerabilities found"
        exit 1
      fi
      
      echo "PASSED: No critical vulnerabilities found"
    - |
      echo "Checking deployment policies"
      roxctl deployment check --file=k8s/deployment.yaml --output=json > policy-results.json
      
      VIOLATIONS=$(jq '.alerts | length' policy-results.json)
      
      if [ "$VIOLATIONS" -gt 0 ]; then
        echo "FAILED: $VIOLATIONS policy violations found"
        jq '.alerts[] | .policy.name' policy-results.json
        exit 1
      fi
      
      echo "PASSED: No policy violations found"
  artifacts:
    reports:
      junit: scan-results.json
    paths:
      - scan-results.json
      - policy-results.json
    expire_in: 1 week
  only:
    - branches

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/
  environment:
    name: production
  only:
    - main
```

## Advanced Integration Patterns

### 1. Quality Gates
```bash
#!/bin/bash
# quality-gate.sh

# Define thresholds
MAX_CRITICAL=0
MAX_HIGH=5
MAX_MEDIUM=20

# Scan image and get results
SCAN_RESULT=$(roxctl image scan --image="$1" --output=json)

# Count vulnerabilities by severity
CRITICAL=$(echo "$SCAN_RESULT" | jq '[.scan.components[].vulns[] | select(.severity == "CRITICAL")] | length')
HIGH=$(echo "$SCAN_RESULT" | jq '[.scan.components[].vulns[] | select(.severity == "IMPORTANT")] | length')
MEDIUM=$(echo "$SCAN_RESULT" | jq '[.scan.components[].vulns[] | select(.severity == "MODERATE")] | length')

echo "Vulnerability Summary:"
echo "  Critical: $CRITICAL (max: $MAX_CRITICAL)"
echo "  High: $HIGH (max: $MAX_HIGH)"
echo "  Medium: $MEDIUM (max: $MAX_MEDIUM)"

# Apply quality gates
FAILED=0

if [ "$CRITICAL" -gt "$MAX_CRITICAL" ]; then
    echo "❌ FAILED: Critical vulnerabilities exceed threshold"
    FAILED=1
fi

if [ "$HIGH" -gt "$MAX_HIGH" ]; then
    echo "❌ FAILED: High severity vulnerabilities exceed threshold"
    FAILED=1
fi

if [ "$MEDIUM" -gt "$MAX_MEDIUM" ]; then
    echo "⚠️  WARNING: Medium severity vulnerabilities exceed threshold"
fi

if [ "$FAILED" -eq 0 ]; then
    echo "✅ PASSED: Image meets security quality gates"
fi

exit $FAILED
```

### 2. Security Report Generation
```python
#!/usr/bin/env python3
# generate-security-report.py

import json
import sys
from datetime import datetime

def generate_html_report(scan_data, output_file):
    """Generate an HTML security report"""
    
    template = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>ACS Security Report</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            .header { background: #f4f4f4; padding: 20px; }
            .critical { color: #d32f2f; font-weight: bold; }
            .high { color: #f57c00; font-weight: bold; }
            .medium { color: #fbc02d; }
            .low { color: #388e3c; }
            table { border-collapse: collapse; width: 100%; margin: 20px 0; }
            th, td { border: 1px solid #ddd; padding: 12px; text-align: left; }
            th { background-color: #f2f2f2; }
        </style>
    </head>
    <body>
        <div class="header">
            <h1>ACS Security Report</h1>
            <p>Generated: {timestamp}</p>
            <p>Image: {image}</p>
        </div>
        
        <h2>Summary</h2>
        <table>
            <tr><th>Severity</th><th>Count</th></tr>
            <tr><td class="critical">Critical</td><td>{critical_count}</td></tr>
            <tr><td class="high">High</td><td>{high_count}</td></tr>
            <tr><td class="medium">Medium</td><td>{medium_count}</td></tr>
            <tr><td class="low">Low</td><td>{low_count}</td></tr>
        </table>
        
        <h2>Detailed Findings</h2>
        {vulnerability_table}
        
    </body>
    </html>
    """
    
    # Process scan data
    vulnerabilities = []
    for component in scan_data.get('scan', {}).get('components', []):
        for vuln in component.get('vulns', []):
            vulnerabilities.append({
                'component': component.get('name', 'Unknown'),
                'cve': vuln.get('cve', 'N/A'),
                'severity': vuln.get('severity', 'Unknown'),
                'summary': vuln.get('summary', 'No description available')
            })
    
    # Count by severity
    severity_counts = {'CRITICAL': 0, 'IMPORTANT': 0, 'MODERATE': 0, 'LOW': 0}
    for vuln in vulnerabilities:
        severity = vuln['severity']
        if severity in severity_counts:
            severity_counts[severity] += 1
    
    # Generate vulnerability table
    vuln_table = "<table><tr><th>Component</th><th>CVE</th><th>Severity</th><th>Description</th></tr>"
    for vuln in vulnerabilities:
        severity_class = vuln['severity'].lower()
        vuln_table += f"""
        <tr>
            <td>{vuln['component']}</td>
            <td>{vuln['cve']}</td>
            <td class="{severity_class}">{vuln['severity']}</td>
            <td>{vuln['summary']}</td>
        </tr>
        """
    vuln_table += "</table>"
    
    # Fill template
    report_html = template.format(
        timestamp=datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        image=scan_data.get('image', {}).get('name', 'Unknown'),
        critical_count=severity_counts['CRITICAL'],
        high_count=severity_counts['IMPORTANT'],
        medium_count=severity_counts['MODERATE'],
        low_count=severity_counts['LOW'],
        vulnerability_table=vuln_table
    )
    
    with open(output_file, 'w') as f:
        f.write(report_html)

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: generate-security-report.py <scan-result.json> <output.html>")
        sys.exit(1)
    
    with open(sys.argv[1], 'r') as f:
        scan_data = json.load(f)
    
    generate_html_report(scan_data, sys.argv[2])
    print(f"Security report generated: {sys.argv[2]}")
```

## Best Practices

### 1. Pipeline Security
- Store API tokens securely using CI/CD secrets management
- Use least-privilege service accounts for ACS integration
- Implement proper error handling and logging
- Set appropriate timeout values for scan operations

### 2. Performance Optimization
- Cache roxctl binary across pipeline runs
- Use parallel scanning for multiple images
- Implement scan result caching for unchanged images
- Configure appropriate resource limits for scan jobs

### 3. Policy Management
- Start with warning-only policies and gradually enforce
- Use policy exceptions sparingly and with proper justification
- Implement policy-as-code with version control
- Regular review and update of security policies

### 4. Reporting and Monitoring
- Generate consistent security reports across all pipelines
- Integrate with existing monitoring and alerting systems
- Track security metrics and trends over time
- Implement automated notifications for critical findings

For more detailed configuration examples, see the [Security Configuration](../security/) guides.