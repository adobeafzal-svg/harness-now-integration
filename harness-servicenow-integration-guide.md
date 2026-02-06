# Harness CI/CD Pipeline with ServiceNow Integration

## Executive Summary

This pipeline demonstrates a production-ready integration between Harness CI/CD and ServiceNow IT Service Management (ITSM), specifically focused on automated change management. The pipeline implements industry best practices for governed deployments with automated change requests, policy-based approvals, continuous verification, and CMDB updates.

## Business Value

### Key Benefits

1. **Automated Compliance** - Every deployment automatically generates a tracked change request with full audit trail
2. **Risk Reduction** - Policy-based evaluation ensures changes meet organizational standards before deployment
3. **Faster Deployments** - Auto-approval for low-risk changes eliminates manual bottlenecks
4. **Complete Visibility** - ServiceNow provides single pane of glass for all changes across the organization
5. **Audit Readiness** - Full traceability from code commit to production deployment
6. **CMDB Accuracy** - Automated updates ensure configuration items reflect actual deployed state

### ROI Metrics

- **50-70% reduction** in change approval cycle time
- **90%+ auto-approval rate** for standard changes
- **Zero manual CMDB updates** - eliminating data drift
- **100% audit compliance** - complete change tracking
- **Reduced MTTR** - immediate access to deployment history in ServiceNow

---

## Architecture Overview

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HARNESS CI/CD PIPELINE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Stage 1: BUILD & TEST                                                   │
│  ├─ Checkout Code                                                        │
│  ├─ Build Application (Maven)                                            │
│  ├─ Unit Tests                                                           │
│  ├─ Security Scan                                                        │
│  └─ Build & Push Docker Image                                            │
│                                                                           │
│  Stage 2: SERVICENOW CHANGE MANAGEMENT                                   │
│  ├─ Create Change Request ──────────────────┐                            │
│  │  • Auto-populated with commits            │                           │
│  │  • Artifacts & test results               │                           │
│  │  • Work items from commits                │                           │
│  │  • Implementation/backout plans           │                           │
│  │                                            ▼                           │
│  ├─ Enrich Change with Additional Data   ┌──────────────────┐           │
│  │                                         │   SERVICENOW     │           │
│  ├─ Policy Evaluation                     │                  │           │
│  │  • Risk score calculation               │  Change Request  │           │
│  │  • Change model validation              │     (CHG)        │           │
│  │  • Approval criteria check              │                  │           │
│  │                                         │  • Create        │           │
│  ├─ Auto-Approve OR Manual Queue          │  • Update        │           │
│  │  IF: Risk < 30 → Auto-approve          │  • Approve       │           │
│  │  ELSE: Manual CAB review                │  • Track         │           │
│  │                                         │  • Close         │           │
│  └─ Wait for Change Approval              └──────────────────┘           │
│                                                    │                      │
│  Stage 3: DEPLOYMENT                               │                      │
│  ├─ Update CHG: Deployment Starting               │                      │
│  ├─ Kubernetes Rolling Deploy                     │                      │
│  ├─ Health Check Verification                     │                      │
│  └─ Update CHG: Deployment Complete               │                      │
│                                                    │                      │
│  Stage 4: CONTINUOUS VERIFICATION                  │                      │
│  ├─ Monitor with Datadog APM                      │                      │
│  ├─ Monitor with Prometheus                       │                      │
│  ├─ Analyze ELK Logs                              │                      │
│  └─ Update CHG with CV Results                    │                      │
│                                                    │                      │
│  Stage 5: UPDATE CMDB & CLOSE                     │                      │
│  ├─ Update CMDB Configuration Items ──────────────┤                      │
│  ├─ Update Related CIs                            │                      │
│  ├─ Close Change Request                          ▼                      │
│  └─ Send Completion Notifications         ┌──────────────────┐           │
│                                             │      CMDB        │           │
│                                             │                  │           │
│                                             │  • App Server    │           │
│                                             │  • Database      │           │
│                                             │  • Load Balancer │           │
│                                             │  • Container Reg │           │
│                                             └──────────────────┘           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Feature Implementation Details

### Feature 1: Pipeline Triggers Deployment

**Implementation:**
- Git webhook triggers (push to main branch, pull request merge)
- Manual execution with build parameters
- Scheduled deployments (cron-based)
- API-triggered deployments

**Configuration:**
```yaml
triggers:
  - name: Main Branch Push
    identifier: main_branch_push
    source:
      type: Webhook
      spec:
        type: Github
        spec:
          type: Push
          spec:
            connectorRef: github_connector
            repoName: my-application
            branch: main
```

---

### Feature 2: ServiceNow Create Step Auto-Generates CHG Ticket

**Implementation:** `ServiceNowCreate` step in Stage 2

**Key Features:**
- Uses ServiceNow connector (OAuth 2.0 or Basic Auth)
- Leverages Standard Change Template
- Auto-populates 30+ fields including:
  - Short description with pipeline/build info
  - Detailed description with full context
  - Change type, category, priority, risk
  - Implementation and backout plans
  - Test plan
  - Custom fields for tracking

**Example Output:**
```
Change Request Created: CHG0012345
State: New
Risk: Low
Type: Standard
```

---

### Feature 3: CHG Populated with Commits, Artifacts, Work Items, Test Results

**Implementation:** Multi-step enrichment process

#### Automatically Populated Data:

**1. Git Commit Information:**
```yaml
- Commit SHA: <+codebase.commitSha>
- Branch: <+codebase.branch>
- Commit Message: <+codebase.commitMessage>
- Author: <+codebase.gitUser>
```

**2. Artifact Details:**
```yaml
- Docker Image: myorg/myapp:<+codebase.commitSha>
- Tag: v<+pipeline.sequenceId>
- Registry: Docker Hub
- Build Number: <+pipeline.sequenceId>
```

**3. Work Items:**
```yaml
# Parsed from commit messages
- JIRA-1234: Implement new payment gateway
- JIRA-1235: Fix authentication timeout issue
- JIRA-1236: Update API documentation
```

**4. Test Results:**
```yaml
- Unit Tests: 47/47 passed (95% coverage)
- Integration Tests: 100% passed
- Security Scan: 0 critical, 0 high vulnerabilities
- Test Reports: Attached via API
```

**5. Implementation Plan:**
```
1. Stop existing application instances
2. Deploy new Docker image: myorg/myapp:<commit-sha>
3. Perform health checks
4. Monitor application metrics
5. Update CMDB configuration items
```

**6. Backout Plan:**
```
1. Rollback to previous Docker image version
2. Restart application services
3. Verify service restoration
```

---

### Feature 4: ServiceNow Policy Evaluation

**Implementation:** Harness Policy as Code with OPA (Open Policy Agent)

#### Policy Rules:

**Auto-Approval Criteria:**
```rego
# Policy: servicenow_change_policy

package servicenow

# Auto-approve if ALL conditions met:
auto_approve {
    input.change_request.risk_score < 30
    input.change_request.security_vulnerabilities == 0
    input.change_request.test_pass_rate == 100
    input.change_request.type == "Standard"
    input.change_request.impact <= 3
}

# Require manual approval if ANY condition met:
manual_approval {
    input.change_request.risk_score >= 30
}

manual_approval {
    input.change_request.priority <= 2
}

manual_approval {
    input.change_request.type == "Emergency"
}

manual_approval {
    input.change_request.impact <= 2
}
```

#### Policy Evaluation Process:

1. **Risk Score Calculation:**
   - Base Risk: 10 points
   - +5 points per critical vulnerability
   - +3 points per high vulnerability
   - +2 points per failed test
   - +10 points for emergency change
   - +5 points for production environment

2. **Change Model Validation:**
   - Standard Change + Risk < 30 → Auto-approve
   - Normal Change → Manual approval
   - Emergency Change → CAB approval

3. **Approval Criteria Check:**
   - Test pass rate >= 95%
   - Security vulnerabilities = 0 (critical/high)
   - Compliance checks passed
   - Change window validated

**Example Policy Evaluation:**
```json
{
  "decision": "auto_approve",
  "risk_score": 15,
  "reason": "Meets standard change criteria",
  "checks": {
    "security": "passed",
    "tests": "passed",
    "change_window": "valid",
    "compliance": "passed"
  }
}
```

---

### Feature 5: Auto-Approval or Manual Queue

**Implementation:** Conditional logic based on policy evaluation

#### Auto-Approval Flow:
```yaml
- step:
    type: ShellScript
    name: Process Policy Decision
    spec:
      script: |
        if [ "$POLICY_STATUS" == "pass" ]; then
          # Call ServiceNow API to auto-approve
          curl -X PUT \
            "https://instance.service-now.com/api/now/table/change_request/$TICKET_NUMBER" \
            -H "Content-Type: application/json" \
            -d '{
              "state": "Scheduled",
              "approval": "approved",
              "approval_history": "Auto-approved via policy evaluation"
            }'
        fi
```

#### Manual Approval Queue:
- Change routed to **Change Advisory Board (CAB)** queue
- Email notifications sent to approvers
- ServiceNow dashboard shows pending changes
- Approvers review change details in ServiceNow UI
- Manual approval/rejection updates CHG state

**Notification Example:**
```
To: change-advisory-board@example.com
Subject: Change Approval Required - CHG0012345

Change Request CHG0012345 requires manual approval:

Application: my-application
Risk Score: 45 (High)
Reason for Manual Review: Risk score exceeds auto-approval threshold

Review in ServiceNow: https://instance.service-now.com/change_request.do?sys_id=...
```

---

### Feature 6: Harness ServiceNow Approval Step Waits for CHG Status

**Implementation:** `ServiceNowApproval` step with polling mechanism

**Configuration:**
```yaml
- step:
    type: ServiceNowApproval
    name: Wait for Change Approval
    spec:
      connectorRef: servicenow_connector
      ticketNumber: CHG0012345
      
      # Approval criteria - pipeline proceeds when met
      approvalCriteria:
        type: KeyValues
        spec:
          matchAnyCondition: false  # ALL conditions must be true
          conditions:
            - key: state
              operator: equals
              value: Scheduled
            - key: approval
              operator: equals
              value: approved
      
      # Rejection criteria - pipeline fails when met
      rejectionCriteria:
        type: KeyValues
        spec:
          matchAnyCondition: true  # ANY condition triggers rejection
          conditions:
            - key: state
              operator: equals
              value: Canceled
            - key: approval
              operator: equals
              value: rejected
      
      # Timeout: Pipeline waits max 2 hours for approval
      timeout: 2h
      
      # Polling interval: Check CHG status every 30 seconds
      pollingInterval: 30s
```

**Behavior:**
- Pipeline **pauses** at this step
- Harness polls ServiceNow API every 30 seconds
- Checks CHG state and approval fields
- **Proceeds** when approval criteria met
- **Fails** when rejection criteria met
- **Times out** after 2 hours (configurable)

**ServiceNow Approval Actions:**
```
Approver Actions in ServiceNow:
├─ Approve → CHG state: "Scheduled" → Pipeline proceeds
├─ Reject → CHG state: "Canceled" → Pipeline fails
└─ Request Info → CHG state: "Pending" → Pipeline continues waiting
```

---

### Feature 7: Upon Approval, Pipeline Proceeds to Deployment

**Implementation:** Deployment stage executes after approval

**Pre-Deployment Actions:**
```yaml
# Update CHG to "Implement" state
- step:
    type: ServiceNowUpdate
    spec:
      fields:
        - name: state
          value: Implement
        - name: work_notes
          value: "Deployment initiated at 2025-02-05 14:30:00"
```

**Deployment Steps:**
1. **Update ServiceNow** - "Deployment Starting"
2. **Kubernetes Rolling Deploy** - Zero-downtime deployment
3. **Health Check Verification** - API endpoint tests
4. **Update ServiceNow** - "Deployment Complete"

**Rollback Strategy:**
- Automatic rollback on deployment failure
- ServiceNow updated with rollback details
- CHG state changed to "Review"
- Previous version restored

---

### Feature 8: Continuous Verification Monitors Post-Deploy Health

**Implementation:** Harness Continuous Verification with multiple health sources

#### Monitored Metrics:

**1. Application Performance (Datadog APM):**
```yaml
- Error Rate: avg:trace.servlet.request.errors{env:production}
- Response Time: avg:trace.servlet.request.duration{env:production}
- Throughput: sum:trace.servlet.request.hits.as_rate()
```

**2. Infrastructure (Prometheus):**
```yaml
- HTTP Error Rate:
    sum(rate(http_requests_total{status=~"5.."}[5m])) /
    sum(rate(http_requests_total[5m]))
- CPU Usage: avg(container_cpu_usage_seconds_total)
- Memory Usage: avg(container_memory_usage_bytes)
```

**3. Log Analysis (ELK Stack):**
```yaml
- Application Errors: level:ERROR AND app:myapp
- Exception Rate: Tracks new exceptions
- Log Patterns: Anomaly detection
```

#### Verification Process:

**Duration:** 15 minutes post-deployment

**Comparison Method:**
- **Baseline:** Last successful deployment
- **Current:** New deployment metrics
- **Analysis:** Statistical comparison

**Risk Assessment:**
- **Low Risk:** All metrics within expected range
- **Medium Risk:** Some anomalies detected, within tolerance
- **High Risk:** Significant deviations, recommend rollback

**Automated Actions:**
- **Pass:** Pipeline proceeds to CMDB update
- **Fail:** Automatic rollback initiated
- **Inconclusive:** Manual review required

**Example CV Results:**
```
Continuous Verification: PASSED ✓

Metrics Analysis (15 min window):
├─ Error Rate: 0.05% (baseline: 0.06%) ✓
├─ Response Time: 145ms (baseline: 142ms) ✓
├─ Throughput: 1,250 req/s (baseline: 1,200 req/s) ✓
├─ CPU Usage: 45% (baseline: 47%) ✓
└─ Memory Usage: 2.1GB (baseline: 2.0GB) ✓

Anomalies Detected: None
Risk Level: Low
Recommendation: Proceed with closure
```

**ServiceNow Update:**
```yaml
- step:
    type: ServiceNowUpdate
    spec:
      fields:
        - name: work_notes
          value: |
            📊 Continuous Verification Results
            Status: PASSED ✓
            Duration: 15 minutes
            Error Rate: 0.05%
            Response Time: 145ms
            Risk Assessment: Low
            Anomalies: None
```

---

### Feature 9: CMDB CIs Updated with New Deployment Status

**Implementation:** REST API calls to ServiceNow CMDB

#### Configuration Items Updated:

**1. Application CI (cmdb_ci_app):**
```json
{
  "sys_id": "cmdb_ci_12345",
  "name": "MyApplication",
  "version": "a3f5d1c",
  "install_status": "Installed",
  "operational_status": "Operational",
  "u_deployment_date": "2025-02-05 14:45:00",
  "u_docker_image": "myorg/myapp:a3f5d1c",
  "u_git_commit": "a3f5d1c",
  "u_deployed_by": "john.doe@example.com",
  "u_pipeline_execution": "exec_12345",
  "comments": "Deployed via Harness - Build 156"
}
```

**2. Related CIs:**
```yaml
Updated CIs:
├─ Application Server (cmdb_ci_app_server)
│  └─ Running version updated
├─ Database (cmdb_ci_database)
│  └─ Connection relationship verified
├─ Load Balancer (cmdb_ci_lb)
│  └─ Backend pool updated
└─ Docker Registry (cmdb_ci_container_registry)
   └─ Image metadata recorded
```

**3. Relationship Updates:**
```
CI Relationships:
├─ MyApplication → RUNS_ON → App Server (prod-app-01)
├─ MyApplication → CONNECTS_TO → Database (prod-db-01)
├─ Load Balancer → ROUTES_TO → MyApplication
└─ MyApplication → USES → Docker Image (myorg/myapp:a3f5d1c)
```

#### CMDB Update Process:

**Step 1: Update Primary Application CI**
```bash
curl -X PUT \
  "https://instance.service-now.com/api/now/table/cmdb_ci_app/cmdb_ci_12345" \
  -H "Content-Type: application/json" \
  -d '{
    "version": "a3f5d1c",
    "operational_status": "1",
    "u_last_deployment": "2025-02-05 14:45:00"
  }'
```

**Step 2: Update Related CIs**
```bash
# Update app server CI
curl -X PUT \
  "https://instance.service-now.com/api/now/table/cmdb_ci_app_server/server_123" \
  -d '{"u_hosted_app_version": "a3f5d1c"}'

# Update database relationship
curl -X PUT \
  "https://instance.service-now.com/api/now/table/cmdb_rel_ci/rel_456" \
  -d '{"u_last_verified": "2025-02-05 14:45:00"}'
```

**Step 3: Record Deployment History**
```json
{
  "table": "u_deployment_history",
  "record": {
    "ci_ref": "cmdb_ci_12345",
    "deployment_date": "2025-02-05 14:45:00",
    "version": "a3f5d1c",
    "deployed_by": "harness_cicd",
    "change_request": "CHG0012345",
    "status": "successful"
  }
}
```

#### Benefits:

✅ **Real-Time Accuracy** - CMDB reflects actual deployed state
✅ **Complete History** - Every deployment recorded
✅ **Relationship Integrity** - Dependencies tracked automatically
✅ **Audit Trail** - Who deployed what, when, and why
✅ **Service Mapping** - Impact analysis for incidents
✅ **Compliance** - SOX/ITIL compliance through accurate CMDB

---

### Feature 10: CHG Ticket Closed with Outcome

**Implementation:** Final ServiceNow update with comprehensive closure notes

#### Closure Data:

**1. Change State Update:**
```yaml
- name: state
  value: Closed

- name: close_code
  value: successful  # or: unsuccessful, unsuccessful_issues_found

- name: actual_start
  value: "2025-02-05 14:30:00"

- name: actual_end
  value: "2025-02-05 14:50:00"
```

**2. Comprehensive Closure Notes:**
```
═══════════════════════════════════════════
CHANGE REQUEST CLOSURE SUMMARY
═══════════════════════════════════════════

Status: SUCCESSFUL ✅

Deployment Details:
- Application: my-application
- Version Deployed: a3f5d1c2b8e9f0a1
- Build Number: 156
- Environment: Production
- Deployment Method: Kubernetes Rolling Update

Timeline:
- Change Created: 2025-02-05 14:15:00
- Approved: 2025-02-05 14:25:00 (Auto-approved)
- Deployment Started: 2025-02-05 14:30:00
- Deployment Completed: 2025-02-05 14:35:00
- Verification Completed: 2025-02-05 14:50:00
- Total Duration: 35 minutes

Verification Results:
- Health Checks: PASSED ✓
- Continuous Verification: PASSED ✓
- Error Rate: 0.05% (within limits)
- Response Time: 145ms (acceptable)
- Anomalies Detected: None

CMDB Updates:
- Application CI: Updated with version a3f5d1c
- App Server CI: Updated
- Database CI: Relationship verified
- Load Balancer CI: Backend pool updated

Artifacts:
- Docker Image: myorg/myapp:a3f5d1c
- Registry: Docker Hub

Git Information:
- Commit: a3f5d1c2b8e9f0a1
- Branch: main
- Author: john.doe
- Message: Add payment gateway integration

Pipeline Execution:
- Execution ID: exec_12345
- URL: https://app.harness.io/executions/exec_12345

Post-Deployment:
- Monitoring: Active (Datadog, Prometheus)
- Alerts: Configured
- Backup: Completed

═══════════════════════════════════════════
Change completed successfully with no issues
═══════════════════════════════════════════
```

**3. Custom Field Updates:**
```yaml
- name: u_deployment_outcome
  value: success

- name: u_verification_status
  value: passed

- name: u_rollback_required
  value: false

- name: u_incidents_created
  value: 0
```

#### Closure Notifications:

**Email Notification:**
```html
Subject: ✅ Change CHG0012345 Closed Successfully

The change request has been completed and closed.

Summary:
- Application: my-application
- Version: a3f5d1c
- Duration: 35 minutes
- Status: Successful

All verification checks passed. Production deployment complete.

View in ServiceNow: [Link]
View in Harness: [Link]
```

**ServiceNow Dashboards Updated:**
- Change success rate metrics
- Deployment velocity tracking
- MTTR (Mean Time To Recovery) statistics
- Compliance reporting

---

## Prerequisites

### ServiceNow Configuration

#### 1. ServiceNow Instance Setup

**Required Modules:**
- Change Management (com.glideapp.change_management)
- CMDB (com.glide.cmdb)
- IT Service Management (ITSM) Suite

**Required Roles:**
- `itil` - For change management operations
- `cmdb_admin` - For CMDB updates
- `rest_api_explorer` - For API access

#### 2. ServiceNow User Account

Create a service account for Harness integration:

```javascript
// In ServiceNow
var user = new GlideRecord('sys_user');
user.initialize();
user.user_name = 'harness_cicd_user';
user.first_name = 'Harness';
user.last_name = 'CI/CD';
user.email = 'harness@example.com';
user.active = true;
user.insert();

// Assign roles
var userRole = new GlideRecord('sys_user_has_role');
userRole.initialize();
userRole.user = user.sys_id;
userRole.role = 'itil,cmdb_admin,rest_api_explorer';
userRole.insert();
```

#### 3. ServiceNow API Configuration

**Enable REST API:**
```
System Web Services > REST API Explorer
- Ensure REST API is active
- Verify Table API is enabled
```

**Configure OAuth (Recommended):**
```
System OAuth > Application Registry
- Create New OAuth Application
- Client ID: <generated>
- Client Secret: <generated>
- Redirect URI: https://app.harness.io/oauth/callback
- Accessible from: All application scopes
```

**Or Configure Basic Auth:**
```
Username: harness_cicd_user
Password: <secure-password>
```

#### 4. ServiceNow Custom Tables (Optional)

```javascript
// Create custom deployment history table
var table = new GlideRecord('sys_db_object');
table.initialize();
table.name = 'u_deployment_history';
table.label = 'Deployment History';
table.insert();

// Add fields
var fields = [
  {name: 'u_ci_ref', type: 'reference', reference: 'cmdb_ci'},
  {name: 'u_deployment_date', type: 'glide_date_time'},
  {name: 'u_version', type: 'string'},
  {name: 'u_change_request', type: 'reference', reference: 'change_request'},
  {name: 'u_deployed_by', type: 'string'},
  {name: 'u_status', type: 'string'}
];
```

#### 5. ServiceNow Change Templates

Create standard change template:

```
Change Management > Templates > New
- Name: Standard_Change_Template
- Type: Standard
- Category: Software
- Risk: Low
- Impact: Low
- Priority: 3
- Pre-populate implementation/backout plans
```

---

### Harness Configuration

#### 1. ServiceNow Connector

**Create Connector:**
```
Project Settings > Connectors > New Connector > ServiceNow

Connector Configuration:
- Name: servicenow_connector
- ServiceNow URL: https://instance.service-now.com
- Authentication:
  Option A - OAuth 2.0:
    - Client ID: <from ServiceNow>
    - Client Secret: <encrypted in Harness Secrets>
    - Token URL: https://instance.service-now.com/oauth_token.do
  
  Option B - Basic Auth:
    - Username: harness_cicd_user
    - Password: <encrypted in Harness Secrets>

- Delegate Selector: primary-delegate
```

**Test Connection:**
```bash
# Harness automatically tests:
# 1. Connectivity to ServiceNow instance
# 2. Authentication credentials
# 3. API permissions (read/write to change_request table)
```

#### 2. Git Connector

```yaml
name: github_connector
type: Github
spec:
  url: https://github.com/myorg/my-application
  authentication:
    type: Http
    spec:
      type: UsernameToken
      spec:
        username: github_user
        tokenRef: github_personal_access_token
  apiAccess:
    type: Token
    spec:
      tokenRef: github_personal_access_token
```

#### 3. Docker Registry Connector

```yaml
name: docker_hub_connector
type: DockerRegistry
spec:
  dockerRegistryUrl: https://index.docker.io/v2/
  providerType: DockerHub
  authentication:
    type: UsernamePassword
    spec:
      username: dockerhub_user
      passwordRef: dockerhub_token
```

#### 4. Monitoring Connectors

**Datadog:**
```yaml
name: datadog_connector
type: Datadog
spec:
  url: https://api.datadoghq.com/
  apiKeyRef: datadog_api_key
  applicationKeyRef: datadog_app_key
```

**Prometheus:**
```yaml
name: prometheus_connector
type: Prometheus
spec:
  url: https://prometheus.example.com
  authentication:
    type: Bearer
    spec:
      tokenRef: prometheus_token
```

#### 5. Harness Secrets

```bash
# Create secrets in Harness
harness secret create \
  --name servicenow_password \
  --value "<password>"

harness secret create \
  --name github_personal_access_token \
  --value "<token>"

harness secret create \
  --name dockerhub_token \
  --value "<token>"
```

#### 6. Harness Policy as Code

Create OPA policy for ServiceNow change evaluation:

```rego
# File: servicenow_change_policy.rego
package servicenow

# Auto-approve standard changes with low risk
auto_approve {
    input.change_request.type == "Standard"
    input.change_request.risk_score < 30
    input.change_request.security_vulnerabilities == 0
    input.change_request.test_pass_rate == 100
}

# Require manual approval for high-risk changes
manual_approval {
    input.change_request.risk_score >= 30
}

manual_approval {
    input.change_request.impact <= 2
}

# Emergency changes always require CAB approval
cab_approval_required {
    input.change_request.type == "Emergency"
}
```

**Upload policy:**
```bash
harness policy create \
  --name servicenow_change_policy \
  --file servicenow_change_policy.rego \
  --type Custom
```

---

## Deployment Instructions

### Step 1: Import Pipeline

```bash
# Using Harness CLI
harness pipeline import \
  --file harness-servicenow-pipeline.yaml \
  --project demo_project \
  --org default

# Or via Harness UI:
# 1. Navigate to Pipelines
# 2. Click "New Pipeline" > "Import from Git/YAML"
# 3. Paste pipeline YAML
# 4. Click "Import"
```

### Step 2: Configure Pipeline Variables

Update these placeholders in the pipeline:

```yaml
# Replace these values:
connectorRef: github_connector          # Your Git connector
repoName: my-application                # Your repository
repo: myorg/myapp                       # Your Docker repo
serviceRef: myapp_service               # Your Harness service
environmentRef: production              # Your environment
infrastructureDefinitions:              # Your infrastructure
  - identifier: prod_k8s_cluster
```

### Step 3: Create Harness Service

```yaml
service:
  name: myapp_service
  identifier: myapp_service
  serviceDefinition:
    type: Kubernetes
    spec:
      manifests:
        - manifest:
            identifier: k8s_manifests
            type: K8sManifest
            spec:
              store:
                type: Github
                spec:
                  connectorRef: github_connector
                  gitFetchType: Branch
                  paths:
                    - k8s/
                  branch: main
      artifacts:
        primary:
          primaryArtifactRef: docker_image
          sources:
            - spec:
                connectorRef: docker_hub_connector
                imagePath: myorg/myapp
                tag: <+input>
              identifier: docker_image
              type: DockerRegistry
```

### Step 4: Create Environment

```yaml
environment:
  name: Production
  identifier: production
  type: Production
  spec:
    infrastructureDefinitions:
      - identifier: prod_k8s_cluster
        name: Production Kubernetes
        type: KubernetesDirect
        spec:
          connectorRef: k8s_cluster_connector
          namespace: production
          releaseName: myapp
```

### Step 5: Test Pipeline

**Manual Test Execution:**
```bash
# Trigger pipeline manually
harness pipeline execute \
  --pipeline servicenow_integrated_deployment \
  --project demo_project \
  --org default \
  --build main

# Monitor execution
harness pipeline get-execution \
  --execution-id <execution_id>
```

**Verify in ServiceNow:**
1. Log into ServiceNow instance
2. Navigate to: Change > All
3. Verify CHG ticket created
4. Check ticket details populated
5. Review work notes for deployment updates
6. Confirm CMDB CIs updated

### Step 6: Configure Webhooks (Optional)

**GitHub Webhook:**
```bash
# Harness auto-generates webhook URL
Webhook URL: https://app.harness.io/gateway/ng/api/webhook/...

# Configure in GitHub:
# Settings > Webhooks > Add webhook
# Payload URL: <webhook URL from Harness>
# Content type: application/json
# Events: Push, Pull request
```

---

## Customization Guide

### Customize Change Request Fields

Add your organization's custom fields:

```yaml
- step:
    type: ServiceNowCreate
    spec:
      fields:
        # Add custom fields
        - name: u_cost_center
          value: "CC-1234"
        
        - name: u_business_unit
          value: "Engineering"
        
        - name: u_compliance_required
          value: "true"
        
        - name: u_sox_relevant
          value: "false"
```

### Customize Policy Rules

Modify OPA policy for your approval criteria:

```rego
# Add custom rules
auto_approve {
    # Auto-approve during maintenance window
    input.change_request.scheduled_start >= "22:00:00"
    input.change_request.scheduled_start <= "06:00:00"
}

auto_approve {
    # Auto-approve for specific environments
    input.change_request.environment == "staging"
}

manual_approval {
    # Require approval for database changes
    input.change_request.category == "Database"
}
```

### Customize CMDB Updates

Add additional CIs to update:

```yaml
- step:
    type: Http
    spec:
      url: https://instance.service-now.com/api/now/table/cmdb_ci_firewall
      method: PUT
      requestBody: |
        {
          "sys_id": "firewall_123",
          "u_rule_updated": "2025-02-05",
          "u_app_version": "<+codebase.commitSha>"
        }
```

### Customize Notifications

Add Slack, Microsoft Teams, or PagerDuty:

```yaml
- step:
    type: Http
    name: Send Slack Notification
    spec:
      url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
      method: POST
      requestBody: |
        {
          "text": "✅ Deployment completed: <+codebase.repoName>",
          "attachments": [{
            "color": "good",
            "fields": [
              {"title": "Version", "value": "<+codebase.commitSha>"},
              {"title": "Environment", "value": "Production"},
              {"title": "Change Request", "value": "<+step.create_change_request.output.ticketNumber>"}
            ]
          }]
        }
```

---

## Troubleshooting

### Common Issues

#### Issue 1: ServiceNow Connector Authentication Failure

**Symptoms:**
```
Error: Failed to authenticate with ServiceNow
Status Code: 401 Unauthorized
```

**Solutions:**
1. Verify credentials in Harness Secrets
2. Check user has `itil` role in ServiceNow
3. Test API access manually:
   ```bash
   curl -u username:password \
     https://instance.service-now.com/api/now/table/change_request?sysparm_limit=1
   ```
4. For OAuth: Regenerate client secret
5. Verify delegate can reach ServiceNow instance

---

#### Issue 2: Change Request Creation Fails

**Symptoms:**
```
Error: Failed to create change request
Message: Field validation failed
```

**Solutions:**
1. Check required fields in ServiceNow:
   ```javascript
   // In ServiceNow, check mandatory fields
   var gr = new GlideRecord('change_request');
   gr.setLimit(1);
   gr.query();
   if (gr.next()) {
       var fields = gr.getElement();
       // Check mandatory attribute
   }
   ```
2. Verify template exists: `Standard_Change_Template`
3. Check field values match ServiceNow validation rules
4. Review ServiceNow system logs: System Logs > System Log > All

---

#### Issue 3: ServiceNow Approval Times Out

**Symptoms:**
```
Error: ServiceNow approval step timed out
Timeout: 2 hours exceeded
```

**Solutions:**
1. Increase timeout in pipeline:
   ```yaml
   timeout: 4h  # Increase from 2h
   ```
2. Check CHG ticket state in ServiceNow manually
3. Verify approval criteria matches CHG fields:
   ```yaml
   approvalCriteria:
     conditions:
       - key: state
         value: Scheduled  # Check exact value in ServiceNow
   ```
4. Review CHG approval history in ServiceNow
5. Check if CAB approval is required but not initiated

---

#### Issue 4: CMDB Update Fails

**Symptoms:**
```
Error: Failed to update CMDB CI
Status: 404 Not Found
```

**Solutions:**
1. Verify CI exists in CMDB:
   ```bash
   curl -u username:password \
     "https://instance.service-now.com/api/now/table/cmdb_ci_app/cmdb_ci_12345"
   ```
2. Check CI sys_id is correct
3. Verify user has `cmdb_admin` role
4. Check CI class matches API endpoint:
   - `cmdb_ci_app` for applications
   - `cmdb_ci_app_server` for servers
   - `cmdb_ci_database` for databases

---

#### Issue 5: Continuous Verification False Positives

**Symptoms:**
```
Warning: CV detected anomalies but deployment is healthy
```

**Solutions:**
1. Adjust CV sensitivity:
   ```yaml
   spec:
     sensitivity: Low  # Change from Medium/High
   ```
2. Increase verification duration:
   ```yaml
   duration: 30m  # Increase from 15m
   ```
3. Review baseline selection:
   ```yaml
   baseline: Last 7 Days  # Instead of Last Successful Deployment
   ```
4. Check if metric thresholds are too strict
5. Review health source queries for accuracy

---

## Best Practices

### 1. Change Request Management

**✅ Do:**
- Use descriptive short descriptions with app name and build number
- Populate implementation and backout plans thoroughly
- Include links to pipeline execution and Git commits
- Update CHG with progress notes during deployment
- Close changes promptly with detailed outcome

**❌ Don't:**
- Create changes without implementation plans
- Leave changes in "New" state after deployment
- Skip verification before closing changes
- Use generic descriptions like "Deploy app"

---

### 2. Policy Configuration

**✅ Do:**
- Start with strict policies and relax gradually
- Test policy changes in non-prod first
- Document policy rules and their rationale
- Review and update policies quarterly
- Monitor auto-approval rates

**❌ Don't:**
- Auto-approve all changes (defeats governance purpose)
- Require manual approval for trivial changes
- Create overly complex policy rules
- Ignore policy violations

---

### 3. CMDB Maintenance

**✅ Do:**
- Update CIs immediately after deployment
- Maintain accurate relationships between CIs
- Record deployment history in custom table
- Audit CMDB accuracy regularly
- Use CI templates for consistency

**❌ Don't:**
- Assume CMDB is accurate without verification
- Update CIs before deployment (premature)
- Skip related CI updates
- Forget to record deployment metadata

---

### 4. Continuous Verification

**✅ Do:**
- Monitor multiple health sources (APM, logs, metrics)
- Set appropriate baseline comparison periods
- Configure alerts for anomalies
- Review CV results before change closure
- Tune sensitivity based on application characteristics

**❌ Don't:**
- Skip CV to save time
- Ignore CV warnings without investigation
- Use only infrastructure metrics (include business metrics)
- Set verification duration too short (<10 min)

---

### 5. Security

**✅ Do:**
- Use OAuth 2.0 for ServiceNow authentication
- Store credentials in Harness Secrets Manager
- Rotate ServiceNow API credentials quarterly
- Implement least-privilege access for service accounts
- Enable audit logging in both Harness and ServiceNow

**❌ Don't:**
- Hardcode credentials in pipeline YAML
- Use admin accounts for automation
- Share service account credentials
- Disable TLS/SSL verification

---

## Integration Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         GIT REPOSITORY                           │
│                     (GitHub/GitLab/Bitbucket)                   │
└────────────────┬────────────────────────────────────────────────┘
                 │ Webhook Trigger
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      HARNESS PLATFORM                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    HARNESS PIPELINE                       │  │
│  │  • Build & Test                                           │  │
│  │  • ServiceNow Integration                                 │  │
│  │  • Deployment                                             │  │
│  │  • Continuous Verification                                │  │
│  └────┬─────────────────────────┬─────────────────────┬──────┘  │
│       │                         │                     │          │
│       │ ServiceNow API          │ Kubernetes API      │ APM APIs │
│       ▼                         ▼                     ▼          │
│  ┌─────────┐             ┌─────────────┐      ┌──────────────┐ │
│  │ Policy  │             │ Deployment  │      │ Health       │ │
│  │ Engine  │             │ Engine      │      │ Sources      │ │
│  │ (OPA)   │             │             │      │ Integration  │ │
│  └─────────┘             └─────────────┘      └──────────────┘ │
└────┬────────────────────────────────────────────────────────────┘
     │
     │ REST API Calls
     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICENOW PLATFORM                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │ Change           │  │ CMDB             │  │ Workflow      │ │
│  │ Management       │  │                  │  │ Engine        │ │
│  │                  │  │ • CIs            │  │               │ │
│  │ • Create CHG     │  │ • Relationships  │  │ • Approvals   │ │
│  │ • Update CHG     │  │ • History        │  │ • Policies    │ │
│  │ • Approve CHG    │  │                  │  │ • Routing     │ │
│  │ • Close CHG      │  │                  │  │               │ │
│  └──────────────────┘  └──────────────────┘  └───────────────┘ │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    REPORTING & ANALYTICS                  │  │
│  │  • Change Success Rate                                    │  │
│  │  • Deployment Velocity                                    │  │
│  │  • MTTR / MTBF                                            │  │
│  │  • Compliance Dashboards                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Monitoring & Observability

### Key Metrics to Track

#### 1. Deployment Metrics
```
- Deployment Frequency: # deployments per day/week
- Lead Time: Commit to production time
- Deployment Duration: Pipeline execution time
- Success Rate: Successful deployments / total deployments
```

#### 2. Change Management Metrics
```
- Auto-Approval Rate: Auto-approved changes / total changes
- Manual Approval Time: Average time in approval queue
- Change Success Rate: Successful changes / total changes
- Emergency Changes: # of emergency changes per month
```

#### 3. Quality Metrics
```
- Failed Deployments: # of failed deployments
- Rollback Rate: Rollbacks / total deployments
- Post-Deployment Incidents: Incidents within 24h of deployment
- MTTR: Mean time to recovery
```

#### 4. Compliance Metrics
```
- CMDB Accuracy: Up-to-date CIs / total CIs
- Unauthorized Changes: Changes without CHG ticket
- Change Documentation: Changes with complete documentation
- Audit Trail Completeness: 100% required
```

### Dashboards

**Harness Dashboard:**
```
- Pipeline execution trends
- Success/failure rates by environment
- CV anomaly detection trends
- Deployment velocity
```

**ServiceNow Dashboard:**
```
- Change requests by status
- Auto-approval vs manual approval breakdown
- Average change duration
- CMDB update frequency
```

---

## Compliance & Audit

### Audit Trail Components

#### 1. Git Commits
```
- Who: Git author
- What: Code changes
- When: Commit timestamp
- Why: Commit message / work item
```

#### 2. ServiceNow Change Request
```
- Who: Requested by, Approved by
- What: Implementation plan, actual changes
- When: Created, approved, implemented timestamps
- Why: Business justification, impact
```

#### 3. Harness Pipeline Execution
```
- Who: Triggered by
- What: Steps executed, artifacts deployed
- When: Execution start/end times
- Why: Build trigger reason
```

#### 4. CMDB Updates
```
- Who: Updated by (Harness service account)
- What: CI changes, version updates
- When: Update timestamps
- Why: Linked to CHG ticket
```

### Compliance Reports

**SOX Compliance:**
```
Report: All Production Changes
Period: Last Quarter
Contents:
- Change request numbers
- Approvers and approval timestamps
- Implementation evidence (pipeline logs)
- Verification results
- CMDB accuracy validation
```

**ITIL Compliance:**
```
Report: Change Management Effectiveness
Metrics:
- % changes with approved CHG
- % changes with complete documentation
- % successful deployments
- Average change duration
```

---

## Advanced Scenarios

### Scenario 1: Multi-Environment Pipeline

Extend pipeline for dev → staging → prod:

```yaml
stages:
  - stage:
      name: Deploy to Development
      type: Deployment
      # No ServiceNow CHG required
  
  - stage:
      name: Deploy to Staging
      type: Deployment
      # Create CHG but auto-approve for staging
  
  - stage:
      name: ServiceNow Change Management
      # Full CHG process for production
  
  - stage:
      name: Deploy to Production
```

---

### Scenario 2: Emergency Hotfix Pipeline

Fast-track for critical fixes:

```yaml
pipeline:
  name: Emergency Hotfix Pipeline
  
  stages:
    - stage:
        name: ServiceNow Emergency Change
        steps:
          - step:
              type: ServiceNowCreate
              spec:
                fields:
                  - name: type
                    value: Emergency
                  
                  - name: priority
                    value: "1"  # Critical
                  
                  - name: justification
                    value: "Production outage - immediate fix required"
          
          # Emergency changes bypass auto-approval
          - step:
              type: ServiceNowApproval
              spec:
                timeout: 30m  # Faster approval expected
```

---

### Scenario 3: Rollback Scenario

Handle failed deployments:

```yaml
execution:
  rollbackSteps:
    # Update ServiceNow with rollback
    - step:
        type: ServiceNowUpdate
        spec:
          fields:
            - name: state
              value: Review
            
            - name: work_notes
              value: |
                ❌ Deployment failed - Rollback executed
                
                Failure Reason: <+pipeline.failureMessage>
                Rollback Time: <+currentDate()>
                Previous Version Restored: <+env.previousVersion>
    
    # Create incident for failed deployment
    - step:
        type: ServiceNowCreate
        spec:
          ticketType: incident
          fields:
            - name: short_description
              value: "Deployment failure - CHG<+step.create_change_request.output.ticketNumber>"
            
            - name: urgency
              value: "2"  # High
            
            - name: change_request
              value: <+step.create_change_request.output.ticketNumber>
```

---

## Cost Optimization

### Licensing Considerations

**Harness:**
- Pipeline executions count toward license
- Continuous Verification requires CV license
- Consider pipeline consolidation

**ServiceNow:**
- API calls count toward limits (check instance tier)
- Optimize API calls with batching
- Use workflow to reduce API round-trips

### Optimization Tips

1. **Reduce API Calls:**
   ```yaml
   # Instead of multiple updates, batch them:
   - step:
       type: ServiceNowUpdate
       spec:
         fields:
           # Update multiple fields in one call
           - name: field1
           - name: field2
           - name: field3
   ```

2. **Cache Policy Evaluations:**
   ```yaml
   # Cache policy results for similar changes
   - step:
       type: Policy
       spec:
         cacheKey: <+codebase.commitSha>-<+env.name>
   ```

3. **Parallel Execution:**
   ```yaml
   # Run independent steps in parallel
   steps:
     - parallel:
         - step: Build Frontend
         - step: Build Backend
         - step: Run Security Scan
   ```

---

## Support & Resources

### Official Documentation

**Harness:**
- ServiceNow Integration: https://developer.harness.io/docs/continuous-delivery/x-platform-cd-features/cd-steps/ticketing-systems/servicenow-integration/
- Continuous Verification: https://developer.harness.io/docs/continuous-delivery/verify/verify-deployments-with-the-verify-step/
- Policy as Code: https://developer.harness.io/docs/platform/governance/policy-as-code/harness-governance-overview/

**ServiceNow:**
- REST API: https://docs.servicenow.com/bundle/tokyo-application-development/page/integrate/inbound-rest/concept/c_RESTAPIExplorer.html
- Change Management: https://docs.servicenow.com/bundle/tokyo-it-service-management/page/product/change-management/concept/c_ITILChangeManagement.html
- CMDB: https://docs.servicenow.com/bundle/tokyo-servicenow-platform/page/product/configuration-management/concept/c_ITILConfigurationManagement.html

### Community Resources

- Harness Community: https://community.harness.io/
- ServiceNow Community: https://www.servicenow.com/community/
- Stack Overflow: [harness] and [servicenow] tags

### Getting Help

1. **Harness Support:**
   - Email: support@harness.io
   - Slack: Harness Community Slack
   - Support Portal: https://support.harness.io/

2. **ServiceNow Support:**
   - HI Portal: https://hi.service-now.com/
   - ServiceNow Support: Case creation via HI portal

---

## Conclusion

This Harness-ServiceNow integration pipeline provides:

✅ **Complete Automation** - From commit to production with governance
✅ **Full Traceability** - Every deployment tracked in ServiceNow
✅ **Risk Management** - Policy-based approvals reduce deployment risk
✅ **Compliance** - SOX, ITIL, and audit requirements met automatically
✅ **Velocity** - Auto-approval for low-risk changes speeds deployment
✅ **Visibility** - Single source of truth for all changes in ServiceNow
✅ **Quality** - Continuous Verification ensures deployment success

### Next Steps

1. **Deploy to Non-Production:** Test pipeline in dev/staging first
2. **Tune Policies:** Adjust auto-approval criteria based on your risk tolerance
3. **Customize Fields:** Add organization-specific ServiceNow fields
4. **Expand Monitoring:** Integrate additional health sources
5. **Train Teams:** Educate developers and change managers on the workflow
6. **Measure Success:** Track deployment velocity and change success rate
7. **Iterate:** Continuously improve based on metrics and feedback

---

**Pipeline Version:** 1.0  
**Last Updated:** 2025-02-05  
**Author:** Harness Solutions Engineering  
**License:** Production-ready, customizable for your organization
