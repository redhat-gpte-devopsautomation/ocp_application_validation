# ADS Multi-User Lab Validation

This document describes how to validate that dynamic user provisioning worked correctly for N users in an ADS workshop environment.

## Overview

The multi-user validation performs these key checks:

1. **User Discovery**: Discovers actual workshop users from Keycloak
2. **Infrastructure Validation**: Validates core ADS platform components
3. **User Provisioning Validation**: Verifies each discovered user exists across all systems
4. **Content Validation**: Checks for user-specific workshop content

## Quick Start

### Validate Multi-User Environment

```bash
cd playbooks
ansible-playbook validate_multiuser_lab.yml
```

### With Custom Configuration

```bash
ansible-playbook validate_multiuser_lab.yml \
  -e user_base_name=student \
  -e keycloak_realm=WORKSHOP \
  -e validate_timeout=900
```

## Validation Flow

### 1. User Discovery Phase

The validation first discovers users from Keycloak using the pattern `{user_base_name}[0-9]+`:

```
Discovering users from Keycloak realm 'ADS'...
Found 30 workshop users: user1, user2, user3, ..., user30
```

**What it checks:**
- Keycloak accessibility and authentication
- All users matching the workshop pattern
- Total count of provisioned users

### 2. Infrastructure Validation Phase

Validates core platform components needed for multi-user access:

```
Validating ADS infrastructure...
✓ Cluster nodes ready
✓ Storage classes configured  
✓ OpenShift GitOps ready
✓ Keycloak StatefulSet ready
✓ GitLab deployment ready
```

### 3. User Provisioning Validation Phase  

For each discovered user, validates:

**GitLab User Existence:**
```
Checking GitLab users...
✓ user1 found in GitLab
✓ user2 found in GitLab  
✗ user3 not found in GitLab
```

**User Access Validation:**
- Showroom accessibility for workshop instructions
- RHDH accessibility for software catalog
- User-specific components and projects

### 4. Content Validation Phase

Checks for user-generated workshop content:

```
User content validation...
✓ Showroom deployment ready
✓ RHDH accessible
✓ Found 15 user-related components in RHDH catalog
✓ Found 12 user-related projects in GitLab
```

## Configuration Variables

### Required Variables
```yaml
# Set in playbook or command line
user_base_name: "user"          # Expected user prefix pattern
keycloak_namespace: "tssc-keycloak"
keycloak_realm: "ADS"
gitlab_namespace: "gitlab"
rhdh_namespace: "tssc-dh"
```

### Optional Variables
```yaml
validate_timeout: 600           # Overall timeout in seconds
strict: false                   # Fail on warnings
```

## Expected Output

### Console Output
```
TASK [=== ADS Multi-User Lab Validation Started ===]
ok: [localhost] => {
    "msg": "Starting multi-user validation of ADS lab environment\nExpected user pattern: userN\nKeycloak namespace: tssc-keycloak\nKeycloak realm: ADS\n"
}

TASK [=== Discovering Users from Keycloak ===]
ok: [localhost] => {
    "msg": "User discovery completed\nTotal users in Keycloak realm 'ADS': 32\nWorkshop users found: 30\nWorkshop users: user1, user2, user3, ..., user30\n"
}

TASK [=== Multi-User Validation Summary ===]
ok: [localhost] => {
    "msg": "Multi-user validation completed\nWorkshop users found: 30\nTotal validation checks: 125\nPassed: 120\nFailed: 2\nWarnings: 3\n"
}
```

### JSON Results
The validation generates `multiuser-validation-results.json`:

```json
[
  {
    "scope": "keycloak_infrastructure",
    "check": "Keycloak route accessibility",
    "status": "PASS",
    "detail": "Keycloak accessible at https://keycloak-tssc-keycloak.apps..."
  },
  {
    "scope": "user_discovery",
    "check": "Workshop user discovery", 
    "status": "PASS",
    "detail": "Found 30 workshop users: user1, user2, user3, ..."
  },
  {
    "scope": "gitlab_users",
    "check": "user1 user existence in GitLab",
    "status": "PASS",
    "detail": "User user1 found in GitLab"
  },
  {
    "scope": "gitlab_users", 
    "check": "user3 user existence in GitLab",
    "status": "FAIL",
    "detail": "User user3 not found in GitLab"
  },
  {
    "scope": "gitlab_user_summary",
    "check": "GitLab user provisioning summary",
    "status": "FAIL", 
    "detail": "28/30 users found in GitLab. Missing: user3, user17"
  }
]
```

## Validation Scope Details

### Infrastructure Checks

| Component | Validation | Failure Impact |
|-----------|------------|----------------|
| Keycloak | StatefulSet ready, route accessible | Critical - stops validation |
| GitLab | Deployment ready, route accessible | Critical - affects user validation |
| RHDH | Deployment ready, route accessible | Warning - users can't access catalog |
| Showroom | Deployment ready, route accessible | Warning - users can't access instructions |

### User-Specific Checks

| Check Type | What It Validates | Expected Result |
|------------|------------------|-----------------|
| Keycloak Users | User exists in realm | All discovered users = 100% |
| GitLab Users | User has GitLab account | All discovered users = 100% |
| GitLab Projects | User has created projects | Variable - depends on workshop progress |
| RHDH Components | User has catalog entries | Variable - depends on workshop progress |

### Content Validation

| Content Type | What It Checks | Status Meaning |
|--------------|----------------|----------------|
| RHDH Components | User-created software catalog entries | WARN if none found (users may not have started) |
| GitLab Projects | User-created repositories | WARN if none found (users may not have started) |
| Showroom Data | User-specific configuration | WARN if missing (content-only mode possible) |

## Troubleshooting

### Common Issues

#### No Users Found
```
Workshop user discovery: WARN
No workshop users found matching pattern user[0-9]+
```

**Causes:**
- Wrong `user_base_name` pattern
- Wrong `keycloak_realm` name
- Users not provisioned yet
- Keycloak authentication issues

**Solutions:**
```bash
# Check actual users in Keycloak
oc exec -n tssc-keycloak statefulset/keycloak -- \
  bash -c "curl -s http://localhost:8080/realms/ADS | jq"

# Try different user pattern
ansible-playbook validate_multiuser_lab.yml -e user_base_name=student

# Check different realm
ansible-playbook validate_multiuser_lab.yml -e keycloak_realm=WORKSHOP
```

#### GitLab User Mismatch
```
GitLab user provisioning summary: FAIL
28/30 users found in GitLab. Missing: user3, user17
```

**Causes:**
- Dynamic user provisioning incomplete
- GitLab provisioning failed for specific users
- Username format mismatch

**Solutions:**
```bash
# Check GitLab users directly
oc exec -n gitlab deployment/gitlab-webservice-default -- \
  gitlab-rails console -e production <<< "User.pluck(:username)"

# Re-run user provisioning for missing users
ansible-playbook provision_users.yml -e target_users=user3,user17
```

#### RHDH/Showroom Not Accessible
```
RHDH route accessibility: FAIL
RHDH not accessible
```

**Solutions:**
```bash
# Check deployment status
oc get deployments -n tssc-dh
oc get routes -n tssc-dh

# Check for alternative namespaces
oc get deployments -A | grep -i rhdh
oc get deployments -A | grep -i backstage
```

### Debug Commands

```bash
# Check Keycloak users
oc exec -n tssc-keycloak statefulset/keycloak -- \
  /opt/keycloak/bin/kcadm.sh get users -r ADS

# Check GitLab users  
oc exec -n gitlab deployment/gitlab-webservice-default -- \
  gitlab-rails runner "puts User.all.map(&:username)"

# Check user-related RHDH components
oc get components -n tssc-dh -o json | \
  jq '.items[].metadata.name' | grep user

# Check user-related GitLab projects
oc exec -n gitlab deployment/gitlab-webservice-default -- \
  gitlab-rails runner "puts Project.all.map(&:name)" | grep user
```

## Integration Examples

### Automation Controller Workflow

```yaml
# Provision ADS Lab workflow
- name: Provision ADS Infrastructure
  job_template: ads_infrastructure_template
  
- name: Provision Workshop Users
  job_template: ads_user_provisioning_template
  dependencies: [ads_infrastructure_template]
  
- name: Validate Multi-User Environment
  job_template: ads_multiuser_validation_template
  dependencies: [ads_user_provisioning_template]
  
- name: Send Workshop Ready Notification
  job_template: notification_template
  dependencies: [ads_multiuser_validation_template]
```

### Custom User Count Validation

```bash
# Validate specific number of users
ansible-playbook validate_multiuser_lab.yml \
  -e expected_user_count=50 \
  -e strict=true

# This will fail if not exactly 50 users found
```

### Workshop Progress Monitoring

```bash
# Check workshop progress during execution
ansible-playbook validate_multiuser_lab.yml \
  -e strict=false \
  --tags content_validation

# Only validates user-generated content, not infrastructure
```

## Performance Considerations

### Scalability
- **10 users**: ~2 minutes validation time
- **30 users**: ~5 minutes validation time  
- **100 users**: ~15 minutes validation time

### API Rate Limits
The validation makes multiple API calls to:
- Keycloak Admin API
- GitLab API
- Kubernetes API

For large user counts (>50), consider:
- Increasing timeouts: `-e validate_timeout=1200`
- Running during low-activity periods
- Monitoring cluster API rate limits

### Network Impact
- Minimal cluster impact (read-only operations)
- Route validation generates HTTP requests to each service
- No external internet required

## Best Practices

1. **Run After User Provisioning**: Always validate after completing user provisioning
2. **Use Appropriate Timeouts**: Increase timeouts for larger user counts
3. **Monitor During Workshops**: Use non-strict mode during active workshops
4. **Automate in Workflows**: Include in provisioning automation workflows
5. **Check Patterns**: Verify user naming patterns match your configuration
6. **Document Failures**: Investigate and document any systematic failures

This multi-user validation ensures that your ADS workshop environment is properly configured for all expected users and can handle the workshop workload effectively.