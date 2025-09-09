# ADS Lab Validation Guide

This document provides specific guidance for validating Red Hat Advanced Developer Suite (ADS) lab environments using the OpenShift Application Validation Framework.

## ADS Components Overview

The ADS workshop typically includes these components:
- **OpenShift GitOps** (ArgoCD) - GitOps deployment automation
- **GitLab** - Source code management and CI/CD
- **Jenkins** - Alternative CI/CD pipeline
- **Red Hat Developer Hub** - Software catalog and templates
- **OpenShift Dev Spaces** - Browser-based development environment
- **Vault** - Secrets management
- **External Secrets Operator** - Kubernetes secrets automation
- **RHACM** - Multi-cluster management
- **ACS/RHACS** - Security and compliance scanning
- **Keycloak/SSO** - Identity and access management
- **Quay** - Container registry
- **TPA** - Trusted Profile Analyzer
- **Showroom** - Workshop instruction delivery
- **Orchestrator** - Workflow automation with Sonataflow

## Quick Start

### Basic ADS Validation

```bash
cd playbooks
ansible-playbook validate_lab.yml
```

This runs the pre-configured ADS validation from `vars/ads_validation_config.yml`.

### Custom Validation

```bash
ansible-playbook validate_lab.yml -e strict=true -e validate_timeout=900
```

## ADS Validation Configuration

The ADS validation is defined in `vars/ads_validation_config.yml` with these validation groups:

### Baseline Cluster Health
```yaml
- cluster:
    name: baseline
    onfail: fail
  validations:
    - kind: Node                    # All nodes Ready
    - kind: StorageClass           # Default storage exists
```

### Core Infrastructure
```yaml
- cluster:
    name: gitops
  validations:
    - kind: Deployment
      name: openshift-gitops-server
      namespace: openshift-gitops
    - kind: Route
      name: openshift-gitops-server
      namespace: openshift-gitops
```

### Development Tools
```yaml
- cluster:
    name: rhdh
  validations:
    - kind: Deployment
      name: developer-hub           # RHDH deployment
      namespace: tssc-dh
    - kind: Route
      name: developer-hub
      namespace: tssc-dh
```

## Component-Specific Validation

### Red Hat Developer Hub

RHDH can be deployed in multiple namespace patterns:
```yaml
# Primary namespace (tssc-dh)
- kind: Deployment
  name: developer-hub
  namespace: tssc-dh

# Alternative namespace (backstage)  
- kind: Deployment
  name: backstage
  namespace: backstage
  onfail: ignore                   # Won't fail if not found
```

### GitLab Validation

GitLab requires multiple components:
```yaml
- cluster:
    name: gitlab
  validations:
    # Web service
    - kind: Deployment
      name: gitlab-webservice-default
      namespace: gitlab
      retries: 120                 # GitLab takes time to start
      delay: 15

    # Database
    - kind: StatefulSet
      name: gitlab-postgresql
      namespace: gitlab

    # External access
    - kind: Route
      name: gitlab
      namespace: gitlab
      status_code: [200, 302]
```

### Jenkins Validation

Jenkins with persistent storage:
```yaml
- cluster:
    name: jenkins
  validations:
    # Application
    - kind: Deployment
      name: jenkins
      namespace: jenkins

    # Storage
    - kind: PersistentVolumeClaim
      name: jenkins
      namespace: jenkins
      conditions:
        - _r_validate_universal.resources[0].status.phase == "Bound"

    # Access
    - kind: Route
      name: jenkins
      namespace: jenkins
```

### Dev Spaces Validation

OpenShift Dev Spaces uses CheCluster CRD:
```yaml
- cluster:
    name: devspaces
  validations:
    # Operator
    - kind: Deployment
      name: devspaces-operator
      namespace: openshift-devspaces

    # Custom Resource
    - kind: CheCluster
      api_version: org.eclipse.che/v2
      name: devspaces
      namespace: openshift-devspaces
      conditions:
        - _r_validate_universal.resources[0].status.chePhase == 'Active'
      retries: 120                 # Dev Spaces setup takes time
```

### ACS/RHACS Validation

Advanced Cluster Security validation:
```yaml
- cluster:
    name: acs
  validations:
    # Central deployment
    - kind: Central
      api_version: platform.stackrox.io/v1alpha1
      name: stackrox-central-services
      namespace: tssc-acs
      conditions:
        - _r_validate_universal.resources[0].status.conditions | selectattr('type', 'equalto', 'Deployed') | selectattr('status', 'equalto', 'True') | list | length > 0

    # Web interface
    - kind: Route
      name: central
      namespace: tssc-acs
```

### Orchestrator/Sonataflow Validation

Workflow orchestration platform:
```yaml
- cluster:
    name: orchestrator
  validations:
    # Operator
    - kind: Deployment
      name: sonataflow-operator-controller-manager
      namespace: sonataflow-operator-system

    # Platform
    - kind: SonataFlowPlatform
      api_version: sonataflow.org/v1alpha08
      name: sonataflow-platform
      namespace: sonataflow-infra
```

## Namespace Patterns

ADS components may use different namespace naming patterns:

| Component | Common Namespaces |
|-----------|------------------|
| RHDH | `tssc-dh`, `backstage`, `rhdh`, `developer-hub` |
| GitLab | `gitlab`, `gitlab-system` |
| Keycloak | `tssc-keycloak`, `rhsso`, `keycloak` |
| ACS | `tssc-acs`, `stackrox`, `rhacs-operator` |
| Quay | `tssc-quay`, `quay-registry` |
| TPA | `tssc-tpa` |

The validation configuration includes multiple namespace checks with `onfail: ignore` for flexibility.

## Timeout Recommendations

ADS components have different startup times:

| Component | Retries | Delay | Total Time |
|-----------|---------|-------|------------|
| GitOps | 60 | 10s | 10 minutes |
| GitLab | 120 | 15s | 30 minutes |
| RHDH | 90 | 15s | 22.5 minutes |
| Dev Spaces | 120 | 15s | 30 minutes |
| ACS Central | 120 | 20s | 40 minutes |

## Error Handling Strategy

The ADS validation uses a mixed error handling approach:

### Critical Components (onfail: fail)
- Baseline cluster health
- Core storage and networking

### Optional Components (onfail: ignore)  
- Individual ADS tools
- Alternative namespace checks
- Secondary features

This ensures that missing optional components don't fail the entire validation.

## Customizing ADS Validation

### Add Custom Components

Edit `vars/ads_validation_config.yml`:
```yaml
- cluster:
    name: my_custom_tool
    onfail: ignore
  validations:
    - kind: Deployment
      name: custom-deployment
      namespace: custom-namespace
      retries: 60
      delay: 15
```

### Modify Timeouts

For slower environments:
```yaml
- kind: Deployment
  name: slow-starting-app
  namespace: apps
  retries: 180          # Increase retries
  delay: 20             # Increase delay
```

### Add Route Authentication

For protected routes:
```yaml
- kind: Route
  name: protected-app
  namespace: apps
  user: admin
  password: secret
  status_code: [200, 302]
```

## Troubleshooting ADS Issues

### Common Problems

1. **RHDH Not Found**
   ```bash
   # Check multiple namespaces
   oc get deployments -A | grep -i rhdh
   oc get deployments -A | grep -i backstage
   oc get deployments -A | grep -i developer
   ```

2. **GitLab Startup Issues**
   ```bash
   # Check GitLab pods
   oc get pods -n gitlab
   oc logs -n gitlab deployment/gitlab-webservice-default
   ```

3. **Dev Spaces Not Ready**
   ```bash
   # Check CheCluster status
   oc get checluster -n openshift-devspaces -o yaml
   ```

4. **Route Connectivity Issues**
   ```bash
   # Test route manually
   oc get route -A
   curl -I https://route-hostname
   ```

### Debug Commands

```bash
# Check operator installations
oc get csv -A

# Check cluster resources
oc get nodes
oc get sc

# Check specific component
oc get all -n component-namespace

# Validate route accessibility  
oc get routes -A
curl -k https://$(oc get route route-name -n namespace -o jsonpath='{.spec.host}')
```

## Integration with ADS Workshops

### Pre-Workshop Validation

Run validation before students access the lab:
```bash
ansible-playbook validate_lab.yml -e strict=true
```

This ensures all components are ready for the workshop.

### During Workshop Monitoring

Use relaxed validation during workshop:
```bash
ansible-playbook validate_lab.yml -e strict=false
```

This monitors health without failing on temporary issues.

### Post-Workshop Cleanup Verification

Verify environment state after workshop:
```bash
# Check that components are still running
ansible-playbook validate_lab.yml
```

## Automation Controller Integration

### Job Template Configuration
- **Name**: ADS Lab Validation
- **Playbook**: `playbooks/validate_lab.yml`  
- **Extra Variables**:
  ```yaml
  validate_timeout: 1200
  strict: false
  ```
- **Credentials**: OpenShift cluster credential

### Workflow Integration

```yaml
# In AAP workflow template
- name: Provision ADS Lab
  job_template: ads_provisioning_template
  
- name: Validate ADS Lab  
  job_template: ads_validation_template
  dependencies:
    - ads_provisioning_template
    
- name: Send Notification
  job_template: notification_template
  dependencies:
    - ads_validation_template
```

## Expected Results

### Successful Validation
```json
[
  {
    "scope": "baseline",
    "check": "Node validation", 
    "status": "PASS",
    "detail": "All 6 nodes are Ready"
  },
  {
    "scope": "gitops",
    "check": "openshift-gitops-server - Route validation",
    "status": "PASS",
    "detail": "Route accessible"
  }
]
```

### Typical Warnings
```json
[
  {
    "scope": "rhdh",
    "check": "backstage - Deployment validation",
    "status": "WARN", 
    "detail": "Deployment not found in backstage namespace"
  }
]
```

These warnings are expected when components use alternative namespaces.

## Performance Considerations

### Parallel Execution
The framework runs validations sequentially within each cluster group but can validate multiple clusters in parallel.

### Resource Usage
Validation queries are lightweight and don't impact cluster performance significantly.

### Network Traffic
Only cluster API calls and route HTTP checks are performed - no external internet required.

## Security Considerations

### Authentication
- Uses existing kubeconfig/service account permissions
- No additional credentials stored or transmitted
- Route validation uses anonymous HTTP requests unless credentials specified

### Permissions Required
- Read access to all namespaces being validated
- List access to cluster-level resources (Nodes, StorageClasses)
- No write permissions required

### Network Access
- Cluster API access required
- Route validation requires ingress network access
- No external internet dependencies