# Harness-ServiceNow Integration: 30-Minute Quick Start

This guide will get you from zero to a working Harness-ServiceNow CI/CD pipeline in 30 minutes.

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] ServiceNow instance (with admin access)
- [ ] Harness account (free trial available: https://app.harness.io/auth/#/signup)
- [ ] Git repository with sample application
- [ ] Kubernetes cluster (or use Harness Cloud for testing)
- [ ] Docker Hub account

## Step-by-Step Setup (30 Minutes)

### Phase 1: ServiceNow Setup (10 minutes)

#### 1. Create ServiceNow Service Account (3 min)

```javascript
// In ServiceNow Navigator, search for "Users" and create new user
Username: harness_automation
First Name: Harness
Last Name: Automation
Email: harness@yourcompany.com
Active: true

// Assign roles
Roles: itil, cmdb_admin, rest_api_explorer
```

#### 2. Get ServiceNow Instance Details (2 min)

```bash
# Note your instance URL
Instance URL: https://YOUR_INSTANCE.service-now.com

# Test API access
curl -u harness_automation:PASSWORD \
  GET "https://<InstanceName>.service-now.com/api/now/table/change_request?sysparm_limit=10"

# You should get a JSON response with change requests
```

#### 3. Create Change Request Template (5 min)

```
1. Navigate to: Change > Administration > Templates
2. Click "New"
3. Fill in:
   - Name: Harness_Standard_Deployment
   - Type: Standard
   - Category: Software
   - Assignment Group: Change Management
   - Risk: Low
   - Priority: 3
4. Click "Submit"
```

### Phase 2: Harness Setup (10 minutes)

#### 1. Create ServiceNow Connector (3 min)

```
1. In Harness, go to: Project Settings > Connectors > New Connector
2. Select: ServiceNow
3. Configure:
   - Name: servicenow_prod
   - ServiceNow URL: https://YOUR_INSTANCE.service-now.com
   - Authentication: Username and Password
   - Username: harness_automation
   - Password: (create secret)
4. Click "Test Connection"
5. Click "Finish"
```

#### 2. Create Git Connector (2 min)

```
1. Connectors > New Connector > GitHub
2. Configure:
   - Name: github_repo
   - URL: https://github.com/YOUR_ORG/YOUR_REPO
   - Authentication: Personal Access Token
   - Token: (create secret with your GitHub token)
3. Click "Test Connection"
4. Click "Finish"
```

#### 3. Create Docker Connector (2 min)

```
1. Connectors > New Connector > Docker Registry
2. Configure:
   - Name: dockerhub
   - Provider: Docker Hub
   - Username: your_dockerhub_username
   - Password: (create secret with your Docker Hub token)
3. Click "Test Connection"
4. Click "Finish"
```

#### 4. Import Sample Pipeline (3 min)

```
1. Go to: Pipelines > Create Pipeline
2. Name: servicenow-demo
3. Click "YAML" view
4. Paste the simplified starter pipeline (see below)
5. Click "Save"
```

**Starter Pipeline (Simplified Version):**

```yaml
pipeline:
  name: ServiceNow Demo Pipeline
  identifier: servicenow_demo
  projectIdentifier: default
  orgIdentifier: default
  
  stages:
    - stage:
        name: Build
        identifier: build
        type: CI
        spec:
          cloneCodebase: true
          execution:
            steps:
              - step:
                  type: Run
                  name: Build App
                  identifier: build_app
                  spec:
                    shell: Sh
                    command: echo "Building application..."
    
    - stage:
        name: Create ServiceNow Change
        identifier: servicenow_change
        type: Custom
        spec:
          execution:
            steps:
              - step:
                  type: ServiceNowCreate
                  name: Create Change Request
                  identifier: create_chg
                  spec:
                    connectorRef: servicenow_prod
                    ticketType: change_request
                    fields:
                      - name: short_description
                        value: "Demo Deployment - Build <+pipeline.sequenceId>"
                      - name: type
                        value: Standard
                      - name: priority
                        value: "3"
                    output:
                      - name: ticketNumber
                        value: <+step.ticket.number>
              
              - step:
                  type: ServiceNowApproval
                  name: Wait for Approval
                  identifier: wait_approval
                  spec:
                    connectorRef: servicenow_prod
                    ticketNumber: <+pipeline.stages.servicenow_change.spec.execution.steps.create_chg.output.ticketNumber>
                    approvalCriteria:
                      type: KeyValues
                      spec:
                        conditions:
                          - key: state
                            operator: equals
                            value: Scheduled
                    timeout: 1h
    
    - stage:
        name: Deploy
        identifier: deploy
        type: Custom
        spec:
          execution:
            steps:
              - step:
                  type: ShellScript
                  name: Simulate Deployment
                  identifier: simulate_deploy
                  spec:
                    shell: Bash
                    source:
                      type: Inline
                      spec:
                        script: |
                          echo "Deploying application..."
                          sleep 5
                          echo "Deployment complete!"
              
              - step:
                  type: ServiceNowUpdate
                  name: Close Change
                  identifier: close_chg
                  spec:
                    connectorRef: servicenow_prod
                    ticketNumber: <+pipeline.stages.servicenow_change.spec.execution.steps.create_chg.output.ticketNumber>
                    ticketType: change_request
                    fields:
                      - name: state
                        value: Closed
                      - name: close_code
                        value: successful
                      - name: close_notes
                        value: "Deployment completed successfully via Harness"
```

### Phase 3: Test Execution (10 minutes)

#### 1. Run the Pipeline (2 min)

```
1. In Harness, open your pipeline
2. Click "Run"
3. Select branch: main (or your default branch)
4. Click "Run Pipeline"
```

#### 2. Monitor in Harness (3 min)

```
Watch the pipeline execute:
1. Build stage completes
2. ServiceNow CHG created
3. Pipeline pauses at approval step
4. Note the CHG ticket number displayed
```

#### 3. Approve in ServiceNow (3 min)

```
1. Log into ServiceNow
2. Navigate to: Change > All
3. Find your change request (CHG00XXXXX)
4. Review the details
5. Click "Schedule" button
6. Set state to "Scheduled"
7. Click "Update"
```

#### 4. Watch Pipeline Complete (2 min)

```
1. Return to Harness
2. Pipeline should proceed automatically
3. Deploy stage executes
4. Change request closed in ServiceNow
5. Pipeline shows "Success" ✓
```

## Validation Checklist

After completing the quick start, verify:

- [ ] Pipeline executed successfully in Harness
- [ ] Change request created in ServiceNow with correct details
- [ ] Pipeline waited for ServiceNow approval
- [ ] Pipeline proceeded after approval in ServiceNow
- [ ] Change request closed with "successful" status
- [ ] All steps show green checkmarks in Harness

## Common Issues & Quick Fixes

### Issue: "ServiceNow connector test failed"

**Solution:**
```bash
# Verify credentials
curl -u username:password \
  "https://instance.service-now.com/api/now/table/sys_user?sysparm_limit=1"

# Check user has required roles in ServiceNow
# Navigate to: User Administration > Users > [your user]
# Verify roles: itil, rest_api_explorer
```

### Issue: "Cannot create change request"

**Solution:**
```
1. Check field names match your ServiceNow instance
2. Verify template exists: Change > Administration > Templates
3. Test manually creating a change in ServiceNow UI
4. Check ServiceNow system logs for errors
```

### Issue: "Approval step times out"

**Solution:**
```
1. Manually check CHG ticket in ServiceNow
2. Verify state field is exactly "Scheduled" (case-sensitive)
3. Check approval criteria in pipeline matches ServiceNow fields
4. Increase timeout: timeout: 2h
```

## Next Steps

Now that you have a working integration:

1. **Add More Stages:**
   - Add actual build steps (Maven, Docker)
   - Add test stages
   - Add real deployment (Kubernetes, ECS)

2. **Enhance ServiceNow Integration:**
   - Add more CHG fields (implementation plan, backout plan)
   - Add work notes updates during deployment
   - Add CMDB updates

3. **Add Policy Evaluation:**
   - Create OPA policies for auto-approval
   - Add risk score calculation
   - Implement change categorization

4. **Add Continuous Verification:**
   - Integrate Datadog, Prometheus, or New Relic
   - Monitor deployment health
   - Auto-rollback on anomalies

5. **Production Readiness:**
   - Review full documentation (see harness-servicenow-integration-guide.md)
   - Customize for your organization's processes
   - Train team on workflow
   - Set up monitoring and alerting

## Learning Resources

### Tutorials
- Harness University: https://university.harness.io
- ServiceNow Learning: https://nowlearning.servicenow.com

### Documentation
- Harness ServiceNow Docs: https://developer.harness.io/docs/continuous-delivery/x-platform-cd-features/cd-steps/ticketing-systems/servicenow-integration/
- ServiceNow REST API: https://docs.servicenow.com/bundle/tokyo-application-development/page/integrate/inbound-rest/concept/c_RESTAPI.html

### Community
- Harness Community Slack
- ServiceNow Community
- Stack Overflow: [harness] [servicenow] tags

## Support

If you get stuck:

1. **Check Documentation:**
   - Full guide: harness-servicenow-integration-guide.md
   - Harness docs: developer.harness.io
   - ServiceNow docs: docs.servicenow.com

2. **Community Help:**
   - Harness Community: community.harness.io
   - ServiceNow Community: community.servicenow.com

3. **Professional Support:**
   - Harness Support: support@harness.io
   - ServiceNow Support: HI Portal

---

**Congratulations!** 🎉 

You now have a working CI/CD pipeline with automated ServiceNow change management. This integration provides governance and visibility while maintaining deployment velocity.

**Time to Production:** 30 minutes  
**Complexity:** Beginner-friendly  
**Production-Ready:** Yes (with customization)
