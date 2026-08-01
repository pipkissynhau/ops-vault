# 03-Jenkins Backup and Recovery: JENKINS_HOME, Credentials, Plugins, and Pipeline Configuration Protection

Recommended Path: 08-CI-CD/02-CI-CD Platform Backup and Disaster Recovery/03-Jenkins Backup and Recovery: JENKINS_HOME, Credentials, Plugins, and Pipeline Configuration Protection.md

Tags: #CI-CD #Jenkins #BackupRestore #JENKINS_HOME #Waterline #CertificateManagement #DisasterPreparedness #SRE

---

## I. Document Explanation

Jenkins is the pipeline orchestration center in enterprise CI/CD systems, typically responsible for:

- Pulling code
- Building applications
- Executing tests
- Building images
- Pushing images to Harbor
- Invoking kubectl/helm to deploy to Kubernetes
- Triggering notifications
- Scheduling Agent task execution
- Managing build records, credentials, plugins, and Job configurations

In a typical CI/CD pipeline, Jenkins sits between GitLab and Harbor/Kubernetes:

    GitLab Code Repository
       |
       v
    Jenkins Pipeline
       |
       v
    Building Images
       |
       v
    Harbor Image Repository
       |
       v
    Kubernetes Deployment

If Jenkins fails, it may cause:

- Pipeline execution failure
- Business inability to auto-deploy
- Loss of build records
- Job configuration loss
- Credential loss
- Plugin version inconsistency
- Agent connection failure
- Kubernetes deployment permission loss
- Harbor login credential loss
- GitLab Webhook trigger failure

Therefore, Jenkins backup is not just about backing up several Jobs, but protecting the entire Jenkins Controller's core configuration, credentials, plugins, tasks, pipeline configurations, and recovery capabilities.

This document, based on Linux Package, Docker, and Kubernetes deployment methods, organizes Jenkins backup objects, backup scripts, recovery procedures, verification checklists, and production considerations.

---

## II. Core Ideas of Jenkins Backup

Jenkins' core data is typically concentrated in:

    $JENKINS_HOME

The default path may vary across different deployment methods.

Common paths:

| Deployment Method | Common JENKINS_HOME |
|---|---|
| Linux Package | /var/lib/jenkins |
| Official Docker Image | /var/jenkins_home |
| Custom Installation | /data/jenkins_home |
| Kubernetes Deployment | PVC-mounted directory, generally corresponding to /var/jenkins_home inside the container |

According to Jenkins' official backup approach, the most direct method is:

    Backup the entire $JENKINS_HOME

This preserves the entire Jenkins Controller's configuration and state.

However, in production environments, note:

- Workspace may be large
- Builds may be numerous
- Artifacts may be large
- Caches can be rebuilt
- Temporary files may not need backup
- Full backup recovery is simple, but backup files may be large
- Fine-grained backup saves space but increases recovery complexity

Therefore, common production strategies are:

    1. Small-scale Jenkins: Backup entire JENKINS_HOME.
    2. Medium-to-large Jenkins: Frequent backup of core configurations, retain build history and artifacts according to policies.
    3. Pipeline should be code-ized as much as possible, placing Jenkinsfile in GitLab.
    4. Agents should be stateless as much as possible, can be rebuilt, not as core backup objects.
    5. Regularly perform recovery drills, not just save backup files.

---

## III. What to Backup in Jenkins

### 3.1 Core Directories and Files

Important Jenkins directories and files:

| Path | Description | Recommended for Backup |
|---|---|---|
| config.xml | Jenkins global configuration | Must |
| credentials.xml | Jenkins credential index | Must |
| secrets/ | Credential encryption keys, master.key, etc. | Must |
| jobs/ | Freestyle Job, Pipeline Job configurations and build records | Must |
| users/ | User configuration | Recommended |
| nodes/ | Static Agent node configuration | Case-by-case |
| plugins/ | Installed plugins | Recommended |
| plugins/*.jpi / *.hpi | Plugin packages | Recommended |
| updates/ | Plugin update metadata | Optional |
| fingerprints/ | Build fingerprints | Case-by-case |
| build queue | Temporary state | Usually not backed up |
| workspace/ | Workspace | Typically not backed up |
| logs/ | Logs | Case-by-case based on audit requirements |
| tools/ | Jenkins automatically installed tools | Case-by-case |
| userContent/ | User-custom static content | Case-by-case |
| secret.key / secret.key.not-so-secret | Old version related keys | Backing up entire secrets is more reliable |

The most critical is:

    credentials.xml + secrets/

Only backing up credentials.xml without secrets/ may result in inability to decrypt credentials after recovery.

---

### 3.2 Job Configuration

Job configurations are typically located in:

    $JENKINS_HOME/jobs/

For example:

    /var/lib/jenkins/jobs/demo-pipeline/config.xml

If it's a multi-level Folder, the path may be:

    /var/lib/jenkins/jobs/folder-name/jobs/job-name/config.xml

Job configurations usually include:

- Job name
- Parameter configuration
- Build triggers
- Git repository address
- Pipeline definition method
- Credential ID
- Build retention policy
- Build steps
- Deployment steps
- Webhook trigger configuration

If Pipeline scripts are directly written in Jenkins Web UI, losing Job configuration will directly result in loss of pipeline logic.

Therefore, production recommendation:

    Pipeline should use Jenkinsfile as much as possible, and place it in GitLab repository.

---

### 3.3 Credentials and Keys

Jenkins credentials typically include: §§code_0§§

- GitLab Access Token
- Harbor Account Password
- SSH Private Key
- Kubernetes kubeconfig
- Docker Registry Credentials
- Webhook Token
- Email Account
- Enterprise WeChat / DingTalk / Slack Webhook
- Cloud Provider AK/SK
- Database Account Password
- Deployment Environment Key

Related Files:

    $JENKINS_HOME/credentials.xml
    $JENKINS_HOME/secrets/

Backup Requirements:

    1. credentials.xml must be backed up.
    2. secrets/ directory must be backed up.
    3. Backup files must have restricted permissions.
    4. Backup files cannot be placed in public directories.
    5. Backup files cannot be committed to Git repositories.
    6. Backup files cannot be sent to chat tools arbitrarily.
    7. After recovery, verify if credentials can be used normally.

---

### 3.4 Plugins

Jenkins's capabilities rely heavily on plugins, such as:

- Git Plugin
- Pipeline Plugin
- Credentials Plugin
- Kubernetes Plugin
- Docker Pipeline Plugin
- GitLab Plugin
- SSH Agent Plugin
- Email Extension Plugin
- Role-based Authorization Strategy Plugin

Plugin-related directories:

    $JENKINS_HOME/plugins/

Purpose of backing up plugins:

- Maintain consistent plugin versions during recovery
- Avoid missing plugins in new environments
- Prevent plugin upgrades from causing Job incompatibility
- Avoid Pipeline syntax becoming unavailable after recovery

Recommended additional export of plugin inventory:

    cd /var/lib/jenkins

    find plugins -maxdepth 1 \( -name "*.jpi" -o -name "*.hpi" \) -printf "%f\n" | sort > /backup/jenkins/plugin-list-$(date +%F).txt

If Jenkins is deployed in a container, you can also execute similar commands inside the container.

---

### 3.5 Build History and Artifacts

Job build history is typically located at:

    $JENKINS_HOME/jobs/<job-name>/builds/

Build artifacts may be found at:

    $JENKINS_HOME/jobs/<job-name>/builds/<build-number>/archive/

Whether to back up build history depends on enterprise requirements.

| Scenario | Recommendation |
|---|---|
| Only concerned with restoring pipeline execution capability | Can retain incomplete build history |
| Need to audit build records | Should retain for a certain period |
| Build artifacts have been pushed to Harbor/Nexus | Jenkins can retain less |
| Jenkins storage pressure is high | Set build retention policies |
| Need to trace issues | Retain longer for critical Jobs |

Production recommendations:

    Jenkins should not long-term act as an artifact repository.
    Images should enter Harbor.
    Package artifacts should enter Nexus/artifact repository.
    Jenkins build artifacts can be retained according to policies.

---

### 3.6 Should workspace be backed up?

Workspace is typically located at:

    $JENKINS_HOME/workspace/

Generally, it is not recommended to back up workspace.

Reasons:

- Workspace is a temporary build working directory
- Code can be re-pulled from GitLab
- The volume may be very large
- It may contain temporary files, caches, and sensitive information
- Backing up workspace will significantly increase backup volume

Recommendations:

    Workspace should not be a core backup object.
    Re-run builds after restoring Jenkins to rebuild workspace.

---

## FourI don't know.Backup Boundary Between Jenkins Controller and Agent

### 4.1 Controller is the core

Jenkins Controller stores:

- Global configuration
- Job configuration
- Credentials
- Plugins
- Users
- Permissions
- Build history
- Agent configuration
- Pipeline execution status

Therefore:

    Jenkins backup core is the $JENKINS_HOME of the Controller.

---

### 4.2 Agent should be stateless

Jenkins Agent typically handles tasks such as:

- Compiling code
- Building images
- Executing tests
- Calling kubectl
- Calling helm
- Pushing artifacts

Production recommendations:

    Agent should be made as rebuildable, replaceable, and stateless.

Agent should not long-term store critical data.

For example:

- Build caches can be rebuilt
- Docker image caches can be re-pulled
- kubectl/helm tools can be included in images
- Maven/npm caches can be optimized via cache repositories
- Critical configurations should be managed by Controller, GitLab, artifact repositories, or credential systems

---

### 4.3 Kubernetes Agent Scenario

If Jenkins uses Kubernetes Plugin to dynamically create Agent Pods:

    Jenkins Controller
          |
          v
    Kubernetes API
          |
          v
    Dynamically create Jenkins Agent Pod

In this scenario, the Agent Pod itself does not need to be backed up.

Focus on backing up:

- Jenkins Controller's JENKINS_HOME
- Kubernetes Plugin configuration
- Credentials for connecting to Kubernetes
- Pod Template configuration
- Jenkinsfile
- Build image
- Harbor credentials
- kubeconfig or ServiceAccount permissions

---

## FiveI don't know.View Jenkins Environment Information

### 5.1 View Jenkins Service Status

Linux Package Deployment:

    systemctl status jenkins

View process:

    ps -ef | grep jenkins | grep -v grep

View listening ports:

    ss -lntp | grep java

---

### 5.2 View JENKINS_HOME

Common default path:

    /var/lib/jenkins

View directory:

    ls -lh /var/lib/jenkins

View systemd configuration:

    systemctl cat jenkins

Find JENKINS_HOME:

    systemctl cat jenkins | grep -i JENKINS_HOME

You can also check the environment variable configuration file:

    grep -R "JENKINS_HOME" /etc/default/jenkins /etc/sysconfig/jenkins 2>/dev/null

If using Docker:

    docker inspect jenkins | grep -i JENKINS_HOME

If using Kubernetes:

    kubectl -n cicd get pod
    kubectl -n cicd describe pod <jenkins-pod-name>

---

### 5.3 Checking Jenkins Version

Web UI method:

    Dashboard -> Manage Jenkins -> System Information

Command-line method to check WAR package version:

    java -jar /usr/share/java/jenkins.war --version

Or check package version:

    dpkg -l | grep jenkins

RPM system:

    rpm -qa | grep jenkins

It is recommended to record the following during recovery:

- Jenkins version
- Java version
- Plugin list
- Operating system version
- Deployment method
- JENKINS_HOME path
- Access domain and port
- Whether reverse proxy is used
- Whether HTTPS is used
- Whether Kubernetes Agent is used

---

## Six. Complete Backup of Jenkins Deployed via Linux Package

### 6.1 Cold Backup Solution

Cold backup is the most reliable method:

    Stop Jenkins
    Backup JENKINS_HOME
    Start Jenkins

Advantages:

- Good data consistency
- Simple recovery
- Less likely to have inconsistent build states

Disadvantages:

- Short downtime
- Backup time depends on JENKINS_HOME size

---

### 6.2 Manual Cold Backup

Create a backup directory:

    sudo mkdir -p /backup/jenkins

Stop Jenkins:

    sudo systemctl stop jenkins

Confirm stop:

    sudo systemctl status jenkins

Backup entire JENKINS_HOME:

    sudo tar -czpf /backup/jenkins/jenkins-home-$(date +%F-%H%M%S).tar.gz -C /var/lib jenkins

Start Jenkins:

    sudo systemctl start jenkins

View backup files:

    sudo ls -lh /backup/jenkins

---

### 6.3 Exclude workspace and cache during backup

If Jenkins is very large, you can exclude workspace and some cache directories.

Example:

    sudo systemctl stop jenkins

    sudo tar -czpf /backup/jenkins/jenkins-home-core-$(date +%F-%H%M%S).tar.gz \
      --exclude='jenkins/workspace' \
      --exclude='jenkins/caches' \
      --exclude='jenkins/.cache' \
      -C /var/lib jenkins

    sudo systemctl start jenkins

Note:

    Excluding workspace can significantly reduce backup size.
    However, if build history and artifacts are also large, you need to design retention policies further.

---

### 6.4 Export plugin list

    sudo mkdir -p /backup/jenkins/meta

    cd /var/lib/jenkins

    sudo find plugins -maxdepth 1 \( -name "*.jpi" -o -name "*.hpi" \) -printf "%f\n" | sort \
      | sudo tee /backup/jenkins/meta/plugin-list-$(date +%F).txt > /dev/null

View:

    sudo cat /backup/jenkins/meta/plugin-list-$(date +%F).txt

---

### 6.5 Record Jenkins version and system information

    sudo mkdir -p /backup/jenkins/meta

    java -version > /tmp/java-version.txt 2>&1
    sudo cp /tmp/java-version.txt /backup/jenkins/meta/java-version-$(date +%F).txt

    dpkg -l | grep jenkins | sudo tee /backup/jenkins/meta/jenkins-package-$(date +%F).txt > /dev/null

RPM system:

    rpm -qa | grep jenkins | sudo tee /backup/jenkins/meta/jenkins-package-$(date +%F).txt > /dev/null

---

## Seven. Jenkins Backup Script Example

### 7.1 Cold Backup Script

Create script:

    sudo mkdir -p /opt/scripts

    sudo tee /opt/scripts/jenkins-backup.sh > /dev/null <<'EOF'
    #!/bin/bash

    set -euo pipefail

    JENKINS_HOME="/var/lib/jenkins"
    BACKUP_ROOT="/backup/jenkins"
    DATE_TIME="$(date +%F-%H%M%S)"

    BACKUP_DIR="${BACKUP_ROOT}/${DATE_TIME}"
    META_DIR="${BACKUP_DIR}/meta"

    mkdir -p "${BACKUP_DIR}" "${META_DIR}"

    echo "[INFO] Jenkins backup started at ${DATE_TIME}"
    echo "[INFO] Jenkins home: ${JENKINS_HOME}"
    echo "[INFO] Backup dir: ${BACKUP_DIR}"

```bash
echo "[INFO] Record Jenkins service status..."
systemctl status jenkins --no-pager > "${META_DIR}/jenkins-status.txt" 2>&1 || true

echo "[INFO] Record Java version..."
java -version > "${META_DIR}/java-version.txt" 2>&1 || true

echo "[INFO] Record Jenkins package version..."
if command -v dpkg >/dev/null 2>&1; then
  dpkg -l | grep jenkins > "${META_DIR}/jenkins-package.txt" 2>/dev/null || true
fi

if command -
```

### 8.3 Docker Deployment Recovery Notes

Pay attention to the following during recovery:

- Jenkins image version
- Java version
- Mounted directory permissions
- Container startup user
- Jenkins URL
- Plugin compatibility
- Whether Docker Socket is mounted
- Existence of build environment tools
- Existence of kubectl / helm
- Harbor login credentials status

---

## Nine. Kubernetes Deployment of Jenkins Backup

### 9.1 Backup Objects

If Jenkins is deployed in Kubernetes, core data is usually stored in PVCs.

Need to back up:

- Jenkins PVC
- Jenkins Helm values
- Jenkins Deployment / StatefulSet
- Service
- Ingress
- Secret
- ConfigMap
- ServiceAccount
- RBAC
- StorageClass information
- GitLab repository containing Jenkinsfile

Check resources:

    kubectl get all -n cicd
    kubectl get pvc -n cicd
    kubectl get secret -n cicd
    kubectl get configmap -n cicd

---

### 9.2 Backup PVC via Scaling Down

Stop Jenkins Controller:

    kubectl -n cicd scale deploy jenkins --replicas=0

If it's a StatefulSet:

    kubectl -n cicd scale statefulset jenkins --replicas=0

Confirm Pods have stopped:

    kubectl -n cicd get pod

Then back up via backup tools, CSI Snapshot, or temporary Pod mounting PVC.

Restart after recovery:

    kubectl -n cicd scale deploy jenkins --replicas=1

Or:

    kubectl -n cicd scale statefulset jenkins --replicas=1

---

### 9.3 Helm values Backup

If Jenkins is installed via Helm, retain the values file:

    helm -n cicd list

Export current values:

    helm -n cicd get values jenkins > jenkins-values-$(date +%F).yaml

Export all values:

    helm -n cicd get values jenkins --all > jenkins-values-all-$(date +%F).yaml

Export manifest:

    helm -n cicd get manifest jenkins > jenkins-manifest-$(date +%F).yaml

Recommend storing these files in an internal Git repository, but avoid committing sensitive Secret plaintext.

---

## Ten. Jenkins Recovery Process

### 10.1 Pre-Recovery Preparation

Confirm before recovery:

- Jenkins version
- Java version
- Operating system version
- JENKINS_HOME path
- Plugin list
- Completeness of credentials and secrets
-Integrity of backup files
- Correctness of backup file permissions
- Sufficient disk space on new server
- Consistency of access domain and port
- Whether to restore reverse proxy configuration
- Whether to restore HTTPS certificate
- Whether to restore Agent connection method

---

### 10.2 Linux Package Recovery Steps

Install Jenkins:

    sudo apt update
    sudo apt install -y jenkins

Stop Jenkins:

    sudo systemctl stop jenkins

Backup new environment default directory:

    sudo mv /var/lib/jenkins /var/lib/jenkins.bak.$(date +%F-%H%M%S)

Extract backup:

    sudo tar -xzpf /backup/jenkins/jenkins-home-2026-04-28-030000.tar.gz -C /var/lib

Confirm directory:

    sudo ls -lh /var/lib/jenkins

Fix permissions:

    sudo chown -R jenkins:jenkins /var/lib/jenkins

Start Jenkins:

    sudo systemctl start jenkins

Check status:

    sudo systemctl status jenkins

Check logs:

    sudo journalctl -u jenkins -f

---

### 10.3 Post-Recovery Checks

Access Jenkins:

    http://jenkins.example.com:8080

Check:

- Whether the page can open
- Whether administrator can log in
- Whether Jobs exist
- Whether Pipelines exist
- Whether credentials are usable
- Whether plugins load normally
- Whether Agents are online
- Whether build records exist
- Whether Jenkinsfile type Jobs can read GitLab
- Whether Freestyle Jobs can execute
- Whether Harbor login is normal
- Whether kubectl / helm deployments are normal
- Whether Webhooks can trigger
- Whether email or enterprise WeChat notifications are normal

Check Jenkins system logs:

    Dashboard -> Manage Jenkins -> System Log

Check system configuration:

    Dashboard -> Manage Jenkins -> System

Check credentials:

    Dashboard -> Manage Jenkins -> Credentials

Check plugins:

    Dashboard -> Manage Jenkins -> Plugins

---

## Eleven. Jenkinsfile and Pipeline as Code

### 11.1 Why Code the Pipeline

If Pipeline scripts are only written in Jenkins Web UI, the Jenkins backup being damaged may also result in loss of pipeline logic.

Recommended approach:

    Place Jenkinsfile in a GitLab code repository.
    Let Jenkins Job only reference the Jenkinsfile in the repository.

This way, even if Jenkins fails, just recover:

- Jenkins base configuration
- Credentials
- Plugins
- Job reference relationships

The core pipeline logic remains in GitLab.

---

### 11.2 Value of Jenkinsfile /think

# The Value of Jenkinsfile

- Pipeline Versioning
- Change Auditable
- Support Code Review
- Support Branch Differences
- Reduce Jenkins Native Configuration Dependency
- Facilitate Jenkins Migration
- Facilitate Job Duplication
- Facilitate Recovery

Recommended Structure:

    app-repo/
    ├── Dockerfile
    ├── Jenkinsfile
    ├── deploy/
    │   ├── values-dev.yaml
    │   ├── values-test.yaml
    │   └── values-prod.yaml
    └── src/

---

### 11.3 Relationship Between Jenkins Job and Jenkinsfile

Recommended Practices:

    Jenkins Job should only store minimal configuration.
    Pipeline logic should be placed in Jenkinsfile.
    Jenkinsfile should be stored in GitLab.
    Deployment parameters and credentials should be managed by Jenkins.

This will significantly reduce Jenkins recovery pressure.

---

## Twelve, Credential Recovery Verification

After Jenkins recovery, credentials are one of the most problematic parts.

Key Verification:

| Credential Type | Verification Method |
|---|---|
| GitLab Token | Run a code pull |
| SSH Private Key | Test Git SSH Clone |
| Harbor Account | docker login / docker push |
| Kubernetes kubeconfig | kubectl get ns |
| Helm Deployment Permissions | helm upgrade --dry-run |
| Webhook Token | Trigger a notification |
| Email Account | Send a test email |
| Cloud Provider AK/SK | Execute minimal permission test |

If credentials cannot be used, focus on checking:

    credentials.xml
    secrets/
    File permissions
    Jenkins user permissions
    Plugin loading status

---

## Thirteen, Plugin Recovery Notes

### 13.1 Risk of Inconsistent Plugin Versions

Inconsistent plugin versions may cause:

- Job configuration cannot be loaded
- Pipeline steps do not exist
- Credential types cannot be recognized
- Kubernetes Agent configuration anomalies
- GitLab Webhook anomalies
- Old Job page errors
- Jenkins startup failure

Recovery Recommendations:

    1. Use the original backup's plugins directory as much as possible.
    2. Record the plugin inventory.
    3. Confirm Jenkins main version compatibility before recovery.
    4. Avoid large-scale plugin upgrades on the day of recovery.
    5. Recover first, verify next, and plan upgrades afterward.

---

### 13.2 Plugin Directory Permissions

If plugins fail to load after recovery, check permissions:

    ls -lh /var/lib/jenkins/plugins

Fix:

    sudo chown -R jenkins:jenkins /var/lib/jenkins/plugins

Restart:

    sudo systemctl restart jenkins

Check logs:

    sudo journalctl -u jenkins -f

---

## Fourteen, Common Faults and Troubleshooting

### 14.1 Jenkins Cannot Start After Recovery

Possible Causes:

- Java version incompatibility
- Jenkins version incompatibility
- Plugin damage
- Permission errors
- JENKINS_HOME path error
- Insufficient disk space

Troubleshooting:

    sudo systemctl status jenkins
    sudo journalctl -u jenkins -xe
    sudo journalctl -u jenkins -f
    df -h
    java -version
    ls -ld /var/lib/jenkins

---

### 14.2 Jobs Disappear After Recovery

Possible Causes:

- jobs/ directory not recovered
- Decompression path error
- Permission errors
- Folder plugin not loaded properly
- Jenkins skipped some configuration during startup

Troubleshooting:

    ls -lh /var/lib/jenkins/jobs
    find /var/lib/jenkins/jobs -name config.xml | head
    sudo chown -R jenkins:jenkins /var/lib/jenkins
    sudo systemctl restart jenkins

---

### 14.3 Credentials Cannot Be Used After Recovery

Possible Causes:

- credentials.xml not recovered
- secrets/ not recovered
- secrets permission errors
- Missing plugins
- Jenkins user lacks file read permissions

Troubleshooting:

    ls -lh /var/lib/jenkins/credentials.xml
    ls -lh /var/lib/jenkins/secrets
    sudo chown -R jenkins:jenkins /var/lib/jenkins
    sudo systemctl restart jenkins

Key Points:

    credentials.xml and secrets/ must be recovered together.

---

### 14.4 Pipeline Reports a Step Does Not Exist After Recovery

Example:

    No such DSL method 'docker'
    No such DSL method 'withCredentials'
    No such DSL method 'sshagent'

Possible Causes:

- Missing Docker Pipeline Plugin
- Missing Credentials Binding Plugin
- Missing SSH Agent Plugin
- Related pipeline plugins not loaded properly
- Plugin version incompatibility

Resolution:

    1. Check if plugins are installed.
    2. Check if plugins are enabled.
    3. Check Jenkins logs.
    4. Compare with the plugin inventory before backup.
    5. Recover the original plugins directory or install corresponding plugins.

---

### 14.5 Agent Offline

Possible Causes:

- Agent node address changed
- SSH credential anomalies
- Inbound agent secret changed
- Firewall port unreachable
- Kubernetes Plugin configuration anomalies
- ServiceAccount permission anomalies

Troubleshooting:

    Dashboard -> Manage Jenkins -> Nodes

Check node logs.

For static SSH Agent:

    ssh jenkins-agent-node

For Kubernetes Agent: /think

kubectl get pod -n cicd
kubectl logs -n cicd <agent-pod-name>

---

### 14.6 Build Can Start But Cannot Pull Code

Possible Causes:

- GitLab Credentials Abnormal
- SSH known_hosts Issue
- GitLab Address Changed
- Network Unreachable
- Git Plugin Abnormal

Troubleshooting:

    1. Check Jenkins Credentials.
    2. Check GitLab Address.
    3. Check Network Between Jenkins and GitLab.
    4. Manually git clone on Jenkins Node.
    5. Check known_hosts.

---

### 14.7 Build Succeeds But Cannot Push Harbor

Possible Causes:

- Harbor Credentials Abnormal
- Docker / containerd Configuration Abnormal
- Harbor Certificate Not Trusted
- Harbor Project Permission Insufficient
- Image Address Written Incorrectly

Troubleshooting:

    docker login harbor.example.com
    docker push harbor.example.com/devops/demo-app:test

If using containerd:

    crictl pull harbor.example.com/devops/demo-app:test

---

### 14.8 Build Succeeds But Cannot Deploy Kubernetes

Possible Causes:

- kubeconfig Credentials Abnormal
- kubectl Not Exist
- helm Not Exist
- kube-apiserver Network Unreachable
- RBAC Permissions Insufficient
- Namespace Does Not Exist
- Context Error

Troubleshooting:

    kubectl version
    kubectl get ns
    kubectl auth can-i get pods -n default
    helm version
    helm list -A

---

## FifteenI don't know.Jenkins Backup Verification Checklist

| Check Item | Verification Method | Pass/Fail |
|---|---|---|
| Jenkins Service Status | systemctl status jenkins |  |
| Web Page Access | Browser Access Jenkins |  |
| Administrator Login | Login Jenkins |  |
| Job Existence | Dashboard Check |  |
| Pipeline Existence | Open Pipeline Job |  |
| Credentials Existence | Manage Credentials |  |
| Credentials Usability | Code Pull/Push/Deployment Test |  |
| Plugin Normality | Manage Plugins |  |
| Agent Online Status | Manage Nodes |  |
| GitLab Code Pull | Execute Test Build |  |
| Harbor Login/Push | docker login/push |  |
| Kubernetes Deployment | kubectl/helm Test |  |
| Webhook Trigger | GitLab Push Test |  |
| Notification Capability | Email/Enterprise WeChat Test |  |
| Build History | View History Build |  |
| Permission Policy | Ordinary User Access Test |  |

---

## SixteenI don't know.Production Backup Strategy Recommendations

### 16.1 Backup Frequency

| Environment | Backup Frequency | Recommendation |
|---|---|---|
| Test Jenkins | Weekly | Acceptable for Lower Recovery Requirements |
| Ordinary Production Jenkins | Daily | At Least Daily Backup |
| Core Deployment Jenkins | Daily or Higher Frequency | Backup Immediately After Configuration Change |
| Large-Scale Jenkins | Tiered Backup | Core Configuration High Frequency, Historical Data Low Frequency |

Recommendations:

    Backup Immediately After Configuration Change.
    Backup Immediately Before Plugin Upgrade.
    Backup Immediately Before Jenkins Upgrade.
    Backup Immediately Before Large Job Adjustments.

---

### 16.2 Backup Retention Policy

Example:

    Last 7 Days: Daily Retention
    Last 4 Weeks: Weekly Retention
    Last 6 Months: Monthly Retention

Simple Cleanup:

    find /backup/jenkins -type f -mtime +30 -delete

If Backup Directory is Organized by Date:

    find /backup/jenkins -mindepth 1 -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;

---

### 16.3 Backup Security

Jenkins Backup May ContainMass Sensitive Information:

- GitLab Token
- Harbor Password
- SSH Private Key
- kubeconfig
- Cloud Vendor AK/SK
- Webhook Token
- Internal System Account Password
- Deployment Script
- Internal Service Address

Security Requirements:

    1. Limit Backup File Permissions.
    2. Do Not Place in Web Accessible Directory.
    3. Do Not Commit to Git Repository.
    4. Do Not Transmit via Ordinary Chat Tools.
    5. Do Not Allow Unrelated Personnel to Access Backup Server.
    6. Remote Synchronization Use SSH/VPN/Dedicated Line.
    7. Important Backup Encrypt and Save.
    8. Revoke Backup Server Permissions for Departed Personnel Immediately.

Example Permissions:

    sudo chown -R root:root /backup/jenkins
    sudo chmod -R 700 /backup/jenkins

---

## SeventeenI don't know.Jenkins Backup Recovery Process Flowchart

```
┌──────────────────────┐
│   Jenkins Controller  │
└──────────┬───────────┘
           │
           │ $JENKINS_HOME
           v
┌──────────────────────┐
│      Backup Server    │
└──────────┬───────────┘
           │
           │ Disaster Recovery
           v
┌──────────────────────┐
│   New Jenkins Instance │
└──────────┬───────────┘
           │
           │ Restore JENKINS_HOME
           v
┌──────────────────────┐
│   Validate Job / Credentials / │
│   Plugins / Agent / Release    │
└──────────────────────┘

---

## 18. RPO and RTO

### 18.1 RPO

RPO:

    Recovery Point Objective, Recovery Point Objective.
    Represents the maximum acceptable data loss duration.

If Jenkins backs up once daily at 3 AM:

    If a failure occurs at 5 PM the same day, the system may lose Jenkins configuration changes and build records from 3 PM onwards.

Ways to reduce RPO:

- Increase backup frequency
- Backup immediately after configuration changes
- Place Jenkinsfile in GitLab
- Code Job configurations
- Require approval for critical configuration changes
- Use external artifact repositories to save build outputs

---

### 18.2 RTO

RTO:

    Recovery Time Objective, Recovery Time Objective.
    Represents the maximum acceptable downtime after Jenkins failure.

Factors affecting RTO:

- Availability of backups
- Availability of recovery documentation
- Availability of standby servers
- Availability of Jenkins installation packages
- Plugin compatibility
- Ability to decrypt credentials
- Ability to quickly rebuild Agents
- Network status of GitLab/Harbor/Kubernetes

Ways to reduce RTO:

- Regular recovery drills
- Preserve Jenkins installation version information
- Backup plugin inventory
- Backup recovery scripts
- Code Jenkinsfile
- Make Agents stateless
- Prepare standby Jenkins hosts or images

---

## 19. Interview Answer Structure

If asked in an interview:

    How to backup Jenkins?

You can answer:

    Jenkins' core backup target is $JENKINS_HOME. Linux package deployments are typically /var/lib/jenkins, Docker official images are usually /var/jenkins_home.
    The most stable production method is to regularly backup the entire JENKINS_HOME, as it contains Jenkins global configurations, Job configurations, credentials.xml, secrets, plugins, users, nodes, and build history.
    Among these, credentials.xml and secrets directory must be backed up together, otherwise credentials may not be decryptable after recovery, such as GitLab Tokens, Harbor passwords, SSH private keys, and kubeconfig files may become unusable.
    If Jenkins is large, you can exclude workspace and cache according to policy, as workspace can typically be regenerated by pulling code again, and Jenkins should not long-term act as an artifact repository.
    During recovery, prepare a same version or compatible Jenkins, stop the service, decompress the backup JENKINS_HOME back to the original path, fix jenkins user permissions, and then start Jenkins.
    After startup, verify Jobs, Pipelines, credentials, plugins, Agents, GitLab pulls, Harbor pushes, Kubernetes releases, and notification capabilities.
    Additionally, pipelines should preferably use Jenkinsfile stored in GitLab, i.e., Pipeline as Code. This way, even if Jenkins fails, the core pipeline logic remains in the code repository, significantly reducing recovery pressure.
    Agents should preferably be stateless, and not be core backup targets if they can be rebuilt. In production, backups should also be synchronized to another machine, and regular recovery drills should be conducted.

---

## 20. Production Implementation Recommendations

Recommended backup implementation method for SMEs:

    1. Clearly define JENKINS_HOME path.
    2. Schedule daily backups of JENKINS_HOME.
    3. Exclude workspace and rebuildable caches.
    4. Backup credentials.xml and secrets/ directory.
    5. Save plugin directory and plugin inventory.
    6. Manually backup before configuration changes, plugin upgrades, and Jenkins upgrades.
    7. Synchronize backup files to another machine.
    8. Limit permissions on backup files.
    9. Store Jenkinsfile in GitLab.
    10. Move build outputs to Harbor/Nexus, not long-term rely on Jenkins.
    11. Make Agents stateless as much as possible.
    12. Conduct regular recovery drills.

---

## 21. Reference Documents

- Jenkins Docs: Backing-up/Restoring Jenkins
  https://www.jenkins.io/doc/book/system-administration/backing-up/

- Jenkins Docs: Pipeline as Code
  https://www.jenkins.io/doc/book/pipeline/pipeline-as-code/

- Jenkins Docs: Using a Jenkinsfile
  https://www.jenkins.io/doc/book/pipeline/jenkinsfile/

- Jenkins Docs: Pipeline Syntax
  https://www.jenkins.io/doc/book/pipeline/syntax/
```