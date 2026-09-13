## CD Using Azure Pipelines — Short Live Delivery Notes

### 1. Continuous Delivery

**Definition:** CD automatically deploys a tested build artifact to environments such as Dev, Test, Staging, and Production.

```text
Code
 ↓
CI Build
 ↓
Artifact
 ↓
Release Pipeline
 ↓
Dev → Test → Staging → Production
```

**Remember:** CI builds the application. CD deploys the application.

---

## 2. Create Classic Release Pipeline

Steps:

```text
Azure DevOps
→ Pipelines
→ Releases
→ New Pipeline
→ Empty Job
→ Add Artifact
→ Select Build Pipeline
→ Add Stage
```

Basic flow:

```text
Build Artifact
      ↓
Development
      ↓
Testing
      ↓
Approval
      ↓
Production
```

---

## 3. Automatically Trigger Release

Enable Continuous Deployment Trigger:

```text
Release Pipeline
→ Edit
→ Artifact
→ ⚡ Continuous Deployment Trigger
→ Enable
```

Flow:

```text
Git Push
   ↓
Build Pipeline
   ↓
New Artifact
   ↓
Release Automatically Starts
```

---

## 4. Deploy to Azure App Service

Add task:

```text
Azure App Service Deploy
```

Configure:

```text
Azure Subscription : azure-service-connection
App Type           : Web App on Linux
App Name           : mywebapp
Package            : **/*.zip
```

YAML equivalent:

```yaml
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'azure-service-connection'
    appType: 'webAppLinux'
    appName: 'mywebapp'
    package: '$(Pipeline.Workspace)/**/*.zip'
```

---

## 5. Configure Approval

**Definition:** Approval requires a person to approve deployment.

Steps:

```text
Release Pipeline
→ Production Stage
→ Pre-deployment Conditions
→ Pre-deployment Approval
→ Enable
→ Add Approver
```

Architecture:

```text
Staging
   ↓
Approval
   ↓
Production
```

**Use case:** Production deployment requires manager/team-lead approval.

---

## 6. Add Multiple Stages

Recommended structure:

```text
Artifact
   ↓
DEV
   ↓
TEST
   ↓
STAGING
   ↓
Approval
   ↓
PRODUCTION
```

Typical configuration:

| Stage      | Deployment        |
| ---------- | ----------------- |
| Dev        | Automatic         |
| Test       | Automatic         |
| Staging    | Automatic         |
| Production | Approval required |

---

## 7. Deployment Groups

**Definition:** A Deployment Group is a collection of target VMs/servers used by Classic Release Pipelines.

Architecture:

```text
Release Pipeline
       ↓
Deployment Group
   ┌───┼───┐
   ↓   ↓   ↓
 VM1  VM2  VM3
```

Steps:

```text
Pipelines
→ Deployment Groups
→ New
→ Select Linux/Windows
→ Copy Registration Script
→ Run Script on VM
```

Example tags:

```text
VM1 → frontend
VM2 → frontend
VM3 → backend
```

---

## 8. Deploy Application to Azure VM

Architecture:

```text
Artifact
   ↓
Release Pipeline
   ↓
Deployment Group
   ↓
Azure VM
   ↓
Nginx / Apache / IIS
```

Linux example:
```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```
Allow Firewall: 
```
sudo ufw allow 'Nginx Full'
sudo ufw enable
```
Deployment command:
```bash
echo "<h1>Webserver: $(hostname)</h1>" | sudo tee /var/www/html/index.html
sudo systemctl restart nginx
```

Verify:
```bash
curl localhost
```

---

## 9. Create SQL Table Using Pipeline

Example `create-table.sql`:

```sql
CREATE TABLE Students
(
    Id INT IDENTITY(1,1) PRIMARY KEY,
    Name NVARCHAR(100),
    Course NVARCHAR(100)
);
```

Pipeline:

```yaml
- task: SqlAzureDacpacDeployment@1
  inputs:
    azureSubscription: 'azure-service-connection'
    AuthenticationType: 'server'
    ServerName: 'myserver.database.windows.net'
    DatabaseName: 'appdb'
    SqlUsername: '$(SQL_USERNAME)'
    SqlPassword: '$(SQL_PASSWORD)'
    deployType: 'SqlTask'
    SqlFile: 'database/create-table.sql'
```

**Remember:** Store SQL username/password as secret variables.

---

## 10. Add Connection String to App Service

Example:

```yaml
- task: AzureAppServiceSettings@1
  inputs:
    azureSubscription: 'azure-service-connection'
    appName: 'mywebapp'
    resourceGroupName: 'myrg'

    connectionStrings: |
      [
        {
          "name": "DefaultConnection",
          "value": "$(SQL_CONNECTION_STRING)",
          "type": "Custom",
          "slotSetting": false
        }
      ]
```

Architecture:

```text
App Service
     │
Connection String
     │
     ▼
Azure SQL Database
```

---

## 11. Associate Work Item with Deployment

**Definition:** Work Items track tasks, bugs, stories, and features.

Commit example:

```bash
git commit -m "Fix login issue AB#123"
```

Flow:

```text
Work Item
   ↓
Commit
   ↓
Build
   ↓
Release
   ↓
Production
```

This gives deployment traceability.

---

## 12. Query Work Items as Deployment Gate

**Definition:** A Gate is an automated condition checked before deployment.

Example condition:

```text
Critical Bugs = 0
```

Architecture:

```text
Staging
   ↓
Query Work Items Gate
   ↓
0 Critical Bugs?
   ↓
Yes
   ↓
Production
```

Steps:

```text
Stage
→ Pre-deployment Conditions
→ Gates
→ Enable
→ Add Query Work Items
→ Select Query
```

Example Azure Boards query:

```text
Work Item Type = Bug
State <> Closed
Priority = 1
```

---

## 13. Approval vs Gate

| Feature  | Meaning             |
| -------- | ------------------- |
| Approval | Human decision      |
| Gate     | Automated condition |

Example:

```text
Staging
   ↓
Gate: Critical Bugs = 0
   ↓
Manager Approval
   ↓
Production
```

---

## 14. Deploy to App Service Deployment Slot

Create staging slot:

```bash
az webapp deployment slot create \
  --resource-group myrg \
  --name mywebapp \
  --slot staging
```

Deploy to slot:

```yaml
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'azure-service-connection'
    appType: 'webAppLinux'
    appName: 'mywebapp'
    deployToSlotOrASE: true
    resourceGroupName: 'myrg'
    slotName: 'staging'
    package: '$(Pipeline.Workspace)/drop/app.zip'
```

Architecture:

```text
Release
   ↓
Staging Slot
   ↓
Test
   ↓
Approval
   ↓
Swap
   ↓
Production
```

---

## 15. Swap Staging to Production

```bash
az webapp deployment slot swap \
  --resource-group myrg \
  --name mywebapp \
  --slot staging \
  --target-slot production
```

Before:

```text
Production = v1
Staging    = v2
```

After:

```text
Production = v2
Staging    = v1
```

---

## 16. Container Deployment

Architecture:

```text
Source Code
    ↓
Dockerfile
    ↓
Docker Build
    ↓
Azure Container Registry
    ↓
Azure App Service / Container
```

Dockerfile:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

Pipeline:

```yaml
- task: Docker@2
  inputs:
    containerRegistry: 'acr-service-connection'
    repository: 'myapp'
    command: 'buildAndPush'
    Dockerfile: '**/Dockerfile'
    tags: |
      $(Build.BuildId)
```

---

# 17. Simple Build Pipeline for Practice

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

steps:

- task: ArchiveFiles@2
  inputs:
    rootFolderOrFile: '$(Build.SourcesDirectory)'
    includeRootFolder: false
    archiveType: 'zip'
    archiveFile: '$(Build.ArtifactStagingDirectory)/app.zip'

- task: PublishPipelineArtifact@1
  inputs:
    targetPath: '$(Build.ArtifactStagingDirectory)'
    artifact: 'drop'
```

---

# 18. Complete Architecture to Draw Live

```text
Developer
   │
   │ git push
   ▼
GitHub / Azure Repos
   │
   ▼
CI Pipeline
   │
   ├─ Build
   ├─ Test
   └─ Publish Artifact
           │
           ▼
     Release Pipeline
           │
           ▼
          DEV
           │
           ▼
          TEST
           │
           ▼
        STAGING
           │
           ▼
     Deployment Gate
           │
           ▼
        Approval
           │
           ▼
      PRODUCTION
       /      \
      ▼        ▼
 App Service  VM
      │
      ▼
 Azure SQL
```

# Points to Remember

* **Build once, deploy the same artifact everywhere.**
* CI creates the artifact; CD deploys it.
* Use **Approvals** for human control.
* Use **Gates** for automated validation.
* Use **Deployment Groups** mainly for Classic Release VM deployments.
* Use **Deployment Slots** for safer App Service deployments.
* Never put passwords directly in YAML.
* Use secret variables, Variable Groups, or Azure Key Vault.
* Production should normally have stronger controls than Dev/Test.
* Always verify the deployment and keep a rollback option.

### One-line flow for students

```text
CODE → BUILD → ARTIFACT → RELEASE → STAGING → GATE → APPROVAL → PRODUCTION
```
