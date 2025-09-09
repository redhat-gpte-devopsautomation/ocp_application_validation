# OpenShift Application Validation Framework

A comprehensive validation framework for OpenShift applications and workloads using pure Ansible and the kubernetes.core collection.

## Overview

This framework validates OpenShift environments by checking:
- Baseline cluster health (nodes, storage, operators)
- Application deployments and their dependencies
- Network connectivity via route validation
- Custom resource states and conditions
- Storage claims and persistence

## Prerequisites

- `ansible >= 2.14`
- `kubernetes.core >= 2.4.0`
- Kubeconfig configured or running from a machine with cluster access
- No external internet required (only cluster API and in-cluster routes)

## Architecture

The framework uses a template-driven approach with the existing `ocp_application_validation` role:

```
ocp_application_validation/
├── defaults/main.yml              # Default configuration
├── tasks/
│   ├── main.yml                   # Main orchestration
│   ├── loop-validations.yml       # Validation loop logic
│   ├── multiuser_discovery.yml    # User discovery from Keycloak
│   ├── multiuser_validation.yml   # Core multi-user validation
│   ├── rhdh_validation.yml        # RHDH-specific validation
│   ├── find-template.yml          # Template discovery
│   └── log-result.yml             # Result logging
├── templates/
│   └── universal.yml.j2           # Universal validation template
├── playbooks/
│   └── validate_lab.yml           # General validation playbook
├── vars/
│   ├── ads_validation_config.yml  # ADS lab configuration
│   └── ocp4_getting_started_validation.yml  # OCP4 Getting Started config
└── README.md                      # This file
```

## Basic Usage

### Simple Validation

```bash
ansible-playbook -i localhost, validate_example.yml
```

### With Custom Configuration

```yaml
---
- name: Validate Environment
  hosts: localhost
  tasks:
    - name: Run validation
      include_role:
        name: ocp_application_validation
      vars:
        ocp_application_validation_list: "{{ my_validation_config }}"
```

## Configuration Format

The validation framework uses a structured YAML configuration:

```yaml
validation_config:
  - cluster:
      name: component_group_name      # Logical grouping
      host: cluster_api_url           # Optional: remote cluster
      username: cluster_user          # Optional: cluster auth
      password: cluster_password      # Optional: cluster auth  
      onfail: ignore|fail            # Error handling strategy
    validations:
      - kind: Deployment             # Kubernetes resource type
        name: resource_name          # Specific resource name (optional)
        namespace: target_namespace  # Target namespace
        api_version: custom.io/v1    # Custom API version (optional)
        retries: 60                  # Retry attempts
        delay: 15                    # Delay between retries
        conditions: []               # Custom validation conditions
        status_code: [200, 302]      # For Route validation
        user: basic_auth_user        # For authenticated routes
        password: basic_auth_pass    # For authenticated routes
        onfail: ignore               # Per-check error handling
```

## Supported Resource Types

### Core Kubernetes Resources
- `Node`, `Namespace`, `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`
- `Service`, `Route`, `Ingress`, `ConfigMap`, `Secret`
- `PersistentVolume`, `PersistentVolumeClaim`, `StorageClass`

### OpenShift-Specific Resources
- `Route` (with HTTP validation)
- `DeploymentConfig`, `BuildConfig`, `ImageStream`

### Operator Resources
- `InstallPlan`, `Subscription`, `ClusterServiceVersion`
- `OperatorGroup`, `CatalogSource`

### Third-Party CRDs
The framework auto-detects API versions for common operators:
- **Tekton**: `Pipeline`, `PipelineRun`, `Task`, `TaskRun`
- **ArgoCD**: `ArgoCD`, `Application`, `AppProject`
- **Kafka**: `Kafka`, `KafkaTopic`, `KafkaUser`
- **NooBaa**: Storage system resources
- **RHACM**: `ManagedCluster`, `MultiClusterHub`

## Validation Types

### Resource Existence
```yaml
- kind: Deployment
  name: my-app
  namespace: production
```

### Resource Health Conditions
```yaml
- kind: Deployment
  name: my-app
  namespace: production
  conditions:
    - _r_validate_universal.resources[0].status.readyReplicas >= 1
    - _r_validate_universal.resources[0].status.replicas == _r_validate_universal.resources[0].status.readyReplicas
```

### Route Connectivity
```yaml
- kind: Route
  name: my-app-route
  namespace: production
  status_code: [200, 302]
  user: admin                    # Optional basic auth
  password: secret              # Optional basic auth
```

### Custom Resource States
```yaml
- kind: ArgoCD
  name: openshift-gitops
  namespace: openshift-gitops
  api_version: argoproj.io/v1alpha1
  conditions:
    - _r_validate_universal.resources[0].status.phase == "Available"
```

### Pipeline Execution
```yaml
- kind: PipelineRun
  namespace: ci-cd
  pipeline: build-pipeline        # Filter by pipeline name
  state: Succeeded               # Expected state
```

### Operator Installation
```yaml
- kind: InstallPlan
  namespace: openshift-operators
  operator: my-operator          # Operator name prefix
  conditions:
    - Install plan completed successfully
```

## Error Handling

### Global Error Strategy
```yaml
- cluster:
    name: critical_components
    onfail: fail                 # Fail entire validation
```

### Per-Check Error Strategy  
```yaml
- kind: Deployment
  name: optional-component
  namespace: apps
  onfail: ignore                 # Log warning, continue
```

### Timeout Configuration
```yaml
- kind: StatefulSet
  name: database
  namespace: data
  retries: 120                   # 120 attempts
  delay: 30                      # 30 seconds between attempts
  # Total timeout: 120 * 30 = 3600 seconds (1 hour)
```

## Advanced Features

### Multi-Cluster Validation
```yaml
validation_config:
  - cluster:
      name: hub_cluster
      host: https://api.hub.example.com:6443
      username: admin
      password: secret
    validations:
      - kind: ManagedCluster
        # Validates managed clusters
        
  - cluster:
      name: local_cluster
      # Uses current kubeconfig
    validations:
      - kind: Deployment
        name: local-app
```

### Dynamic Resource Discovery
```yaml
- kind: Pod
  namespace: monitoring
  # Validates all pods in namespace
  conditions:
    - _r_validate_universal.resources | length > 0
    - _r_validate_universal.resources | selectattr('status.phase', 'equalto', 'Running') | list | length == (_r_validate_universal.resources | length)
```

### Conditional Validation
```yaml
- kind: Deployment
  name: gpu-workload
  namespace: compute
  conditions:
    - _r_validate_universal.resources | length > 0
    - _r_validate_universal.resources[0].spec.template.spec.nodeSelector['accelerator'] is defined
  retries: 30
  delay: 10
```

## Output Formats

### Console Logs
```
TASK [[gitops][Route]=> checking openshift-gitops-server]
ok: [localhost] => {
    "msg": "[ openshift-gitops-server - Route validation on gitops cluster : PASS ]"
}

TASK [[database][StatefulSet]=> checking postgresql]
failed: [localhost] => {
    "msg": "[ postgresql - StatefulSet validation on database cluster : FAIL ]"
}
```

### Results File
Results are written to `ocp_application_validation_result_path`:
```yaml
- scope: gitops
  check: "openshift-gitops-server - Route validation"
  status: PASS
  detail: "Route accessible with status 200"
  
- scope: database  
  check: "postgresql - StatefulSet validation"
  status: FAIL
  detail: "StatefulSet not ready: 0/1 replicas"
```

## Variables Reference

### Required Variables
```yaml
ocp_application_validation_list: []  # Validation configuration
```

### Optional Variables
```yaml
ocp_application_validation_file_path: "/tmp/validations/{{ guid[:5] | d('common') }}/testaments"
ocp_application_validation_remove_testaments: true
ocp_application_validation_result_path: "/tmp/validations/{{ guid[:5] | d('common') }}/"
```

## Examples

### Basic Application Stack
```yaml
app_stack_validation:
  - cluster:
      name: application
    validations:
      # Database tier
      - kind: StatefulSet
        name: postgresql
        namespace: database
        
      # Application tier  
      - kind: Deployment
        name: api-server
        namespace: application
        
      # Web tier
      - kind: Deployment
        name: frontend
        namespace: application
        
      # Ingress
      - kind: Route
        name: app-route
        namespace: application
        status_code: 200
```

### CI/CD Pipeline Validation
```yaml
cicd_validation:
  - cluster:
      name: cicd
    validations:
      # GitOps
      - kind: ArgoCD
        name: openshift-gitops
        namespace: openshift-gitops
        
      # Build pipelines
      - kind: Pipeline
        name: build-pipeline
        namespace: cicd
        
      # Recent pipeline runs
      - kind: PipelineRun
        namespace: cicd
        pipeline: build-pipeline
        state: Succeeded
```

## Extending the Framework

### Adding Custom Resource Types
Edit `templates/universal.yml.j2` to add API version mappings:
```jinja2
{% elif _validate.kind == 'MyCustomResource' %}
    api_version: custom.example.com/v1alpha1
```

### Custom Validation Logic
Create specialized task files for complex validations:
```yaml
# tasks/custom_validation.yml
- name: Complex validation logic
  kubernetes.core.k8s_info:
    kind: "{{ resource_kind }}"
    namespace: "{{ resource_namespace }}"
  register: resources
  
- name: Custom condition check
  assert:
    that:
      - resources.resources | length > 0
      - custom_validation_function(resources.resources)
```

## Integration Examples

### Automation Controller Job Template
- **Playbook**: `validate_environment.yml`
- **Variables**: Environment-specific validation config
- **Credentials**: OpenShift cluster credential
- **Execution**: As part of deployment pipeline

### GitLab CI/CD Pipeline
```yaml
validate_deployment:
  stage: validate
  script:
    - ansible-playbook validate_app.yml -e environment=production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

### ArgoCD Post-Sync Hook
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: validation-
  annotations:
    argocd.argoproj.io/hook: PostSync
spec:
  template:
    spec:
      containers:
      - name: validator
        image: ansible-runner:latest
        command: [ansible-playbook, validate_app.yml]
```

## Lab Environment Examples

### OCP4 Getting Started Lab

Use the general validation framework with OCP4 Getting Started configuration:

```bash
# Basic OCP4 Getting Started validation
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ocp4_getting_started_validation.yml

# With multi-user validation
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ocp4_getting_started_validation.yml \
  -e ocp_application_validation_multiuser_enabled=true \
  -e user_count=10 \
  -e user_base_name=student

# Custom namespaces for specific environment
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ocp4_getting_started_validation.yml \
  -e rhsso_namespace=keycloak \
  -e gitea_namespace=git \
  -e showroom_namespace=lab-instructions
```

**OCP4 Getting Started Components:**
- **Baseline**: Cluster nodes, storage classes
- **OpenShift Pipelines**: Tekton controller and webhook
- **OpenShift GitOps**: ArgoCD server and controllers
- **Gitea**: Git repository server and database
- **RHSSO/Keycloak**: Authentication server (optional)
- **Showroom**: Lab instruction delivery (optional)
- **Operators**: Catalog sources and subscriptions

### ADS Lab Environment

**Multi-User ADS Workshop:**
```bash
# ADS workshop validation
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ads_validation_config.yml

# ADS with multi-user validation  
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ads_validation_config.yml \
  -e ocp_application_validation_multiuser_enabled=true \
  -e keycloak_namespace=tssc-keycloak \
  -e keycloak_realm=ADS
```

**Single-User RHADS Enterprise Demo:**
```bash
# RHADS enterprise demo validation (single-user demo)
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/rhads_demo_validation.yml \
  -e ocp_application_validation_multiuser_enabled=true
```

### Custom Lab Environment

Create your own validation configuration:

```bash
# Using custom validation config
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/my_custom_validation.yml \
  -e ocp_application_validation_multiuser_enabled=true
```

## Modular Validation Architecture

The framework uses a modular approach where product-specific validations are separated into their own task files:

### Core Multi-User Validation
- **File**: `tasks/multiuser_validation.yml`
- **Purpose**: Generic multi-user validation for any lab type
- **Validates**: Keycloak users, Git repositories (GitLab/Gitea), Showroom, GitOps applications
- **Used by**: All lab types with multi-user environments

### Technology-Specific Validations
- **RHADS Technology**: `tasks/rhdh_validation.yml`
  - Validates Red Hat Developer Hub components
  - Triggered when `ocp_application_validation_technology: rhads`
  - Used in ADS labs

- **KubeVirt Technology**: `tasks/kubevirt_validation.yml`
  - Validates OpenShift Virtualization components
  - Triggered when `ocp_application_validation_technology: kubevirt`
  - Used in virtualization roadshow labs

### Technology-Based Configuration
Each lab configuration file specifies its technology type to determine which validations to run:

```yaml
# In vars/ads_validation_config.yml
ocp_application_validation_technology: rhads  # Enables RHDH validation

# In vars/ocp4_getting_started_validation.yml  
ocp_application_validation_technology: ocp   # Basic OCP validation only

# In vars/ocp_virt_roadshow_validation.yml
ocp_application_validation_technology: kubevirt  # Enables KubeVirt validation
```

### Adding New Technology Validations
1. Create new task file: `tasks/technology_validation.yml`
2. Add conditional include in `multiuser_validation.yml`:
   ```yaml
   - name: Run AI validation for AI technology
     ansible.builtin.include_tasks: ./ai_validation.yml
     when: ocp_application_validation_technology | default('') == 'ai'
   ```
3. Set technology type in lab configuration files:
   ```yaml
   ocp_application_validation_technology: ai
   ```

### Supported Technologies
- **`rhads`**: Red Hat Application Development Suite (RHDH + GitLab)
- **`kubevirt`**: OpenShift Virtualization (KubeVirt + MTV)
- **`ocp`**: Basic OpenShift (GitOps + Pipelines + Gitea)
- **`ai`**: AI/ML workloads (future)
- **`generic`**: Core validation only

### Supported Authentication Types
- **`rhsso`**: Red Hat Single Sign-On (legacy Keycloak)
- **`rhbk`**: Red Hat build of Keycloak (modern Keycloak)
- **`htpasswd`**: OpenShift htpasswd identity provider

### Authentication Configuration
Each lab specifies its authentication method for user discovery:

```bash
# RHADS lab with RHBK
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ads_validation_config.yml \
  -e ocp_application_validation_multiuser_enabled=true

# OCP4 Getting Started with RHSSO  
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ocp4_getting_started_validation.yml \
  -e ocp_application_validation_multiuser_enabled=true

# OCP Virt Roadshow with htpasswd
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/ocp_virt_roadshow_validation.yml \
  -e ocp_application_validation_multiuser_enabled=true

# Override auth type at runtime
ansible-playbook playbooks/validate_lab.yml \
  -e @vars/my_lab_validation.yml \
  -e ocp_application_validation_multiuser_enabled=true \
  -e ocp_application_validation_auth_type=htpasswd
```

## Best Practices

1. **Start Simple**: Begin with basic resource existence checks
2. **Add Conditions Gradually**: Build complex validation logic incrementally  
3. **Use Appropriate Timeouts**: Set realistic retry/delay values for your environment
4. **Group Related Checks**: Use logical cluster names for better organization
5. **Handle Failures Gracefully**: Use `onfail: ignore` for non-critical components
6. **Test Validation Logic**: Verify conditions work in your specific environment
7. **Document Custom Logic**: Comment complex validation conditions

## Troubleshooting

### Common Issues
- **Authentication**: Ensure proper kubeconfig or cluster credentials
- **Timeouts**: Increase retries/delay for slow-starting applications
- **API Versions**: Verify CRD API versions match your cluster
- **Conditions**: Test condition logic with actual resource data

### Debug Tips
```bash
# Run with verbose output
ansible-playbook validate_app.yml -v

# Check specific resource
oc get deployment my-app -n my-namespace -o yaml

# Test condition logic
ansible localhost -m debug -a "var=my_complex_condition"
```