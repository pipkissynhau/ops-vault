# 03-Jenkins Backup and Recovery: Protection of JENKINS_HOME, Credentials, Plugins, and Pipeline Configurations

Recommended Path: 08-CI-CD/02-CI-CD Platform Backup and Disaster Recovery/03-Jenkins Backup and Recovery: Protection of JENKINS_HOME, Credentials, Plugins, and Pipeline Configurations.md

Tags: #CI-CD #Jenkins #Backup and Recovery #JENKINS_HOME #Pipelines #Credential Management #Disaster Recovery #SRE

---

## I. Document Overview

Jenkins serves as the central hub for pipeline scheduling in an enterprise's CI/CD system, typically handling tasks such as:

- Pulling code
- Building applications
- Executing tests
- Creating images
- Pushing images to Harbor
- Using kubectl/helm for deployment to Kubernetes
- Sending notifications
- Scheduling Agent tasks
- Managing build records, credentials, plugins, and Job configurations

In a typical CI/CD pipeline, Jenkins sits between GitLab and Harbor/Kubernetes:

    GitLab Code Repository
       |
       v
    Jenkins Pipeline
       |
       v
    Image Building
       |
       v
    Harbor Image Repository
       |
       v
    Kubernetes Deployment

A Jenkins failure can lead to various issues, including:

- Inability to execute pipelines
- Failure in automatic business deployments
- Loss of build records
- Loss of Job configurations
- Loss of credentials
- Inconsistent plugin versions
- Agent connection issues
- Loss of Kubernetes deployment permissions
- Loss of Harbor login credentials
- Failure of GitLab Webhook triggering

Therefore, backing up Jenkins is not just about backing up individual Jobs but rather protecting the entire core configuration, credentials, plugins, tasks, pipeline settings, and recovery capabilities of the Jenkins Controller.

This document focuses on Linux Package, Docker, and Kubernetes deployment scenarios, detailing Jenkins backup objects, scripts, recovery processes, verification checks, and production considerations.

---

## II. Key Concepts in Jenkins Backup

Jenkins' critical data is typically stored in:

    $JENKINS_HOME

The default path may vary depending on the deployment method.

Common paths include:

| Deployment Method | Common JENKINS_HOME Path |
|---|---|
| Linux Package | /var/lib/jenkins |
| Docker Official Image | /var/jenkins_home |
| Custom Installation | /data/jenkins_home |
| Kubernetes Deployment | PVC-mounted directory, usually /var/jenkins_home in the container |

The most straightforward approach to backing up Jenkins is to back up the entire $JENKINS_HOME folder. This ensures that all configurations and states of the Jenkins Controller are preserved.

However, in a production environment, it's important to consider:

- The workspace can be quite large.
- There may be numerous builds and artifacts.
- Caches can be recreated.
- Temporary files may not need to be backed up.
- While overall backup and recovery are straightforward, the backup files can be large.
- More refined backups save space but increase complexity during restoration.

Therefore, common production strategies include:

    1. For small-scale Jenkins instances: Back up the entire $JENKINS_HOME folder.
    2. For medium to large-scale Jenkins instances: Frequently back up core configurations and retain build history and artifacts based on specific policies.
    3. Code pipeline scripts should be stored in Jenkinsfile format within a GitLab repository.
    4. Agents should be stateless where possible, allowing for easy reconstruction without being considered critical backup targets.
    5. Regular recovery drills should be conducted instead of merely storing backup files.

---

## III. What Needs to Be Backed Up in Jenkins

### 3.1 Core Directories and Files

Important directories and files in Jenkins include:

| Path | Description | Recommended for Backup |
|---|---|---|
| config.xml | Global Jenkins configuration | Required |
| credentials.xml | Credential index | Required |
| secrets/ | Credential encryption keys, master.key, etc. | Required |
| jobs/ | Freestyle Job and Pipeline Job configurations and build records | Required |
| users/ | User configurations | Recommended |
| nodes/ | Static Agent node configurations | Depending on usage |
| plugins/ | Installed plugins | Recommended |
| plugins/*.jpi / *.hpi | Plugin packages | Recommended |
| updates/ | Plugin update metadata | Optional |
| fingerprints/ | Build fingerprints | Depending on requirements |
| build queue | Temporary status information | Generally not backed up |
| workspace/ | Workspace directory | Usually not backed up |
| logs/ | Logs | Depending on audit needs |
| tools/ | Automatically installed Jenkins tools | Depending on usage |
| userContent/ | User-defined static content | Depending on usage |
| secret.key / secret.key.not-so-secret | Old version-related keys | It's safer to back up the entire secrets folder |

The most critical items to back up are:

    credentials.xml + secrets/

If only credentials.xml is backed up and secrets/ isThe workspace is not considered a core backup object. After restoring Jenkins, you can rebuild the workspace by executing builds again.

---

## IV. Backup Boundaries between Jenkins Controller and Agent

### 4.1 The Controller is Core

The Jenkins Controller stores:

- Global configurations
- Job configurations
- Secrets
- Plugins
- Users
- Permissions
- Build history
- Agent configurations
- Pipeline execution status

Therefore, the core of Jenkins backup lies in the $JENKINS_HOME directory of the Controller.

---

### 4.2 Agents Should Be Stateless Whenever Possible

Jenkins Agents are typically responsible for executing tasks such as:

- Compiling code
- Building images
- Running tests
- Calling kubectl or helm commands
- Pushing artifacts

In production environments, it is recommended that Agents be designed to be rebuildable, replaceable, and stateless. It is not advisable to store critical data on Agents for an extended period.

For example:

- Build caches can be recreated.
- Docker image caches can be pulled again.
- kubectl/helm tools can be included within the images.
- Maven/npm caches can be managed through cache repositories.
- Truly critical configurations should be managed by the Controller, GitLab, artifact repositories, or secret management systems.

---

### 4.3 In the Case of Using Kubernetes Agents

If Jenkins uses Kubernetes plugins to dynamically create Agent Pods:

    Jenkins Controller
          |
          v
    Kubernetes API
          |
          v
    Dynamically created Jenkins Agent Pod

In this scenario, it is even less necessary to back up the Agent Pod itself. The key items to back up include:

- The JENKINS_HOME directory of the Jenkins Controller.
- Kubernetes plugin configurations.
- Secrets used to connect to Kubernetes.
- Pod Template configurations.
- Jenkinsfile.
- Built images.
- Harbor secrets.
- kubeconfig or ServiceAccount permissions.

---

## V. Viewing Jenkins Environment Information

### 5.1 Checking Jenkins Service Status

For Linux Package Deployments:

    Use `systemctl status jenkins`.

To view processes:

    Run `ps -ef | grep jenkins | grep -v grep`.

To check listening ports:

    Use `ss -lntp | grep java`.

---

### 5.2 Viewing JENKINS_HOME

Common default paths include:

    `/var/lib/jenkins`

To list the contents of the directory:

    Run `ls -lh /var/lib/jenkins`.

To view systemd configuration:

    Execute `systemctl cat jenkins`.

To locate the JENKINS_HOME path:

    Use `systemctl cat jenkins | grep -i JENKINS_HOME`.

You can also check environment variable configuration files:

    Run `grep -R "JENKINS_HOME" /etc/default/jenkins /etc/sysconfig/jenkins 2>/dev/null`.

For Docker deployments:

    Use `docker inspect jenkins | grep -i JENKINS_HOME`.

For Kubernetes deployments:

    Execute `kubectl -n cicd get pod` and then `kubectl -n cicd describe pod <jenkins-pod-name>`.

---

### 5.3 Checking Jenkins Version

To view the version through the Web UI:

    Go to Dashboard -> Manage Jenkins -> System Information.

To check the war package version via command line:

    Run `java -jar /usr/share/java/jenkins.war --version`.

Or to view the package version:

    For Debian/RHEL systems, use `dpkg -l | grep jenkins`.
    For RPM systems, use `rpm -qa | grep jenkins`.

When restoring Jenkins, it is recommended to record the following information:

- Jenkins version.
- Java version.
- List of installed plugins.
- Operating system version.
- Deployment method.
- JENKINS_HOME path.
- Access domain name and port.
- Whether a reverse proxy is used.
- Whether HTTPS is enabled.
- Whether Kubernetes Agents are being utilized.

---

## VI. Comprehensive Backup for Jenkins Deployed via Linux Packages

### 6.1 Cold Backup Approach

The cold backup method is the most reliable:

1. Stop Jenkins.
2. Back up the JENKINS_HOME directory.
3. Restart Jenkins.

Advantages:

- Good data consistency.
- Simple recovery process.
- Reduces the risk of inconsistent build states during recovery.

Disadvantages:

- Requires a temporary shutdown of Jenkins.
- The backup time depends on the size of the JENKINS_HOME directory.

---

### 6.2 Manual Cold Backup

Create a backup directory:

    `sudo mkdir -p /backup/jenkins`

Stop Jenkins:

    `sudo systemctl stop jenkins`

Verify shutdown status:

    `sudo systemctl status jenkins`

Back up the entire JENKINS_HOME directory:

    `sudo tar -czpf /backup/jenkins/jenkins-home-$(date +%F-%H%M%S).tar.gz -C /var/lib jenkins`

Restart Jenkins:

    `sudo systemctl start jenkins`

To check the    echo "[INFO] Recording Java version..."
    java -version > "${META_DIR}/java-version.txt" 2>&1 || true

    echo "[INFO] Recording Jenkins package version..."
    if command -v dpkg >/dev/null 2>&1; then
      dpkg -l | grep jenkins > "${META_DIR}/jenkins-package.txt" 2>/dev/null || true
    fi

    if command -v rpm >/dev/null 2>&1; then
      rpm -qa | grep jenkins > "${META_DIR}/jenkins-package.txt" 2>/dev/null || true
    fi

    echo "[INFO] Exporting plugin list..."
    if [ -d "${JENKINS_HOME}/plugins" ]; then
      find "${JENKINS_HOME}/plugins" -maxdepth 1 \( -name "*.jpi" -o -name "*.hpi" \) -printf "%f\n" | sort > "${META_DIR}/plugin-list.txt" || true
    fi

    echo "[INFO] Stopping Jenkins..."
    systemctl stop jenkins

    echo "[INFO] Creating Jenkins home archive..."
    tar -czpf "${BACKUP_DIR}/jenkins-home-${DATE_TIME}.tar.gz" \
      --exclude='jenkins/workspace' \
      --exclude='jenkins/caches' \
      --exclude='jenkins/.cache' \
      -C "$(dirname "${JENKINS_HOME}")" "$(basename "${JENKINS_HOME}")"

    echo "[INFO] Starting Jenkins..."
    systemctl start jenkins

    echo "[INFO] Clearing backups older than 14 days..."
    find "${BACKUP_ROOT}" -mindepth 1 -maxdepth 1 -type d -mtime +14 -exec rm -rf {} \;

    echo "[INFO] Jenkins backup completed."
    EOF

Add execution permissions:

    sudo chmod +x /opt/scripts/jenkins-backup.sh

Run manually:

    sudo /opt/scripts/jenkins-backup.sh

View backups:

    sudo find /backup/jenkins -maxdepth 3 -type f | sort

---

### 7.2 Setting Up Scheduled Backups

Execute daily at 3 AM:

    sudo crontab -e

Add the following line:

    0 3 * * * /opt/scripts/jenkins-backup.sh >> /var/log/jenkins-backup.log 2>&1

Check scheduled tasks:

    sudo crontab -l

View logs:

    sudo tail -f /var/log/jenkins-backup.log

---

### 7.3 Synchronizing to Remote Backup Servers

Local backups cannot replace disaster recovery measures.

Sync to a remote backup server:

    rsync -avz /backup/jenkins/ backup-user@10.0.0.100:/data/backup/jenkins/

Alternatively, use scp:

    scp -r /backup/jenkins backup-user@10.0.0.100:/data/backup/

Recommendations:

    1. Keep short-term backups locally.
    2. Store long-term backups on a remote server.
    3. Encrypt important backups.
    4. Regularly test the recoverability of remote backups.

---

## Section 8: Backups for Docker-deployed Jenkins

### 8.1 Using Docker Volumes

If Jenkins uses Docker Volumes:

    docker volume ls | grep jenkins

Example:

    jenkins_home

For backup:

    docker stop jenkins

    docker run --rm \
      -v jenkins_home:/var/jenkins_home \
      -v /backup/jenkins:/backup \
      busybox \
      tar -czf /backup/jenkins_home_$(date +%F-%H%M%S).tar.gz -C /var/jenkins_home .

    docker start jenkins

To view:

    ls -lh /backup/jenkins

---

### 8.2 Using Docker Bind Mounts

If the startup command is similar to:

    docker run -d \
      --name jenkins \
      -p 8080:8080 \
      -p 50000:50000 \
      -v /data/jenkins_home:/var/jenkins_home \
      jenkins/jenkins:lts

Then, perform the following for backup:

    docker stop jenkins

    tar -czpf /backup/jenkins/jenkins-home-$(date +%F-%H%M%S).tar.gz -C /data jenkins_home

    docker start jenkins

During restoration, ensure to:

- Restore /data/jenkins_home and verify that the container is still mounted to this directory.

---

### 8.3 Precautions for Restoring Docker-deployed Jenkins

During restoration, pay attention to the following:

- Jenkins image version
- Java version
- Permissions of the mounted directories
- Container startup user
- Jenkins URL
- Plugin compatibility
-- Is it necessary to restore the reverse proxy configuration?
- Is it necessary to restore the HTTPS certificate?
- Is it necessary to restore the Agent connection method?

---

### 10.2 Steps for Restoring Linux Packages

Install Jenkins:

    sudo apt update
    sudo apt install -y jenkins

Stop Jenkins:

    sudo systemctl stop jenkins

Back up the default directory of the new environment:

    sudo mv /var/lib/jenkins /var/lib/jenkins.bak.$(date +%F-%H%M%S)

Unzip the backup:

    sudo tar -xzpf /backup/jenkins/jenkins-home-2026-04-28-030000.tar.gz -C /var/lib

Verify the directory:

    sudo ls -lh /var/lib/jenkins

Repair permissions:

    sudo chown -R jenkins:jenkins /var/lib/jenkins

Start Jenkins:

    sudo systemctl start jenkins

Check the status:

    sudo systemctl status jenkins

View logs:

    sudo journalctl -u jenkins -f

---

### 10.3 Post-Recovery Checks

Access Jenkins:

    http://jenkins.example.com:8080

Verify:

- Whether the page can be opened.
- Whether the administrator can log in.
- Whether Jobs exist.
- Whether Pipelines exist.
- Whether credentials are available.
- Whether plugins are loading correctly.
- Whether Agents are online.
- Whether build records exist.
- Whether Jenkinsfile-type Jobs can read from GitLab.
- Whether Freestyle Jobs can be executed.
- Whether Harbor login is functioning properly.
- Whether kubectl /helm releases are working correctly.
- Whether Webhooks can be triggered.
- Whether email or WeCom notifications are functioning normally.

View Jenkins system logs:

    Dashboard -> Manage Jenkins -> System Log

Check system configuration:

    Dashboard -> Manage Jenkins -> System

Review credentials:

    Dashboard -> Manage Jenkins -> Credentials

Examine plugins:

    Dashboard -> Manage Jenkins -> Plugins

---

## Chapter Eleven: Jenkinsfile and Pipeline as Code

### 11.1 Why Codify Pipelines

If the Pipeline script is only written in Jenkins' Web UI, then any damage to the Jenkins backup could result in the loss of pipeline logic.

It is more recommended to:

- Store the Jenkinsfile in a GitLab code repository.
- Have the Jenkins Job simply reference this Jenkinsfile from the repository.

This way, even if Jenkins fails, as long as you restore:

- The basic Jenkins configuration.
- Credentials.
- Plugins.
- The Job reference relationships,

the core pipeline logic will still be intact in GitLab.

---

### 11.2 The Value of Jenkinsfile

The benefits of using a Jenkinsfile include:

- Version control for pipelines.
- Auditable changes.
- Support for code reviews.
- Comparison of branch differences.
- Reduction of reliance on local Jenkins configurations.
- Facilitation of Jenkins migrations.
- Ease in replicating Jobs.
- Simplified recovery processes.

Recommended structure:

    app-repo/
    ├── Dockerfile
    ├── Jenkinsfile
    ├── deploy/
    │   ├── values-dev.yaml
    │   ├── values-test.yaml
    │   └── values-prod.yaml
    └── src/

---

### 11.3 The Relationship Between Jenkins Job and Jenkinsfile

The recommended approach is:

- To keep only a minimal amount of configuration in the Jenkins Job.
- To place the pipeline logic in the Jenkinsfile.
- To store the Jenkinsfile in GitLab.
- To have Jenkins manage release parameters and credentials.

This will significantly reduce the stress on Jenkins during recovery processes.

---

## Chapter Twelve: Verification of Credential Restoration

After restoring Jenkins, credentials are one of the most likely issues to arise.

Key verification points include:

| Credential Type | Verification Method |
|---|---|
| GitLab Token | Execute a code pull once. |
| SSH Private Key | Test Git SSH Clone. |
| Harbor Account | Use docker login / docker push. |
| Kubernetes kubeconfig | Execute kubectl get ns. |
| Helm Release Permissions | Try helm upgrade --dry-run. |
| Webhook Token | Trigger a notification once. |
| Email Account | Send a test email. |
- Cloud provider AK/SK | Perform minimal permission tests.

If any credentials fail to function correctly, focus on checking:

    credentials.xml
    secrets/
    File permissions.
    Jenkins user permissions.
    Whether plugins are loading properly.

---

## Chapter Thirteen: Precautions for Restoring Plugins

### 13.1 Risks of Inconsistent Plugin Versions

Inconsistent plugin versions can lead to:

- Jobs failing to load configurations correctly.
- Missing Pipeline steps.
- Unrecognized credential types.
- Abnormal Kubernetes Agent settings.
- Issues with GitLab Webhooks.
- Old Job pages displaying errors.
- Jenkins failing to start.

Restoration recommendations include:

    1. Try to use the plugins directory from the original backup.
    2.- Issues with SSH known_hosts
- Changes in GitLab address
- Network connectivity issues
- Abnormalities with Git plugins

Troubleshooting:

    1. Check Jenkins credentials.
    2. Verify the GitLab address.
    3. Inspect the network connection between Jenkins and GitLab.
    4. Manually perform a git clone on the Jenkins node.
    5. Check the known_hosts file.

---

### 14.7 Successful build but unable to push to Harbor

Possible causes:

- Abnormal Harbor credentials
- Issues with Docker / containerd configuration
- Harbor certificates not trusted
- Insufficient permissions for the Harbor project
- Incorrect mirror address

Troubleshooting:

    docker login harbor.example.com
    docker push harbor.example.com/devops/demo-app:test

For containerd:

    crictl pull harbor.example.com/devops/demo-app:test

---

### 14.8 Successful build but unable to deploy to Kubernetes

Possible causes:

- Abnormal kubeconfig credentials
- Kubectl not installed
- Helm not installed
- Network issues with the kube-apiserver
- Insufficient RBAC permissions
- Non-existent Namespace
- Context errors

Troubleshooting:

    kubectl version
    kubectl get ns
    kubectl auth can-i get pods -n default
    helm version
    helm list -A

---

## Section Fifteen: Jenkins Backup Verification Checklist

| Inspection Item | Verification Method | Pass or Fail |
|-----------------|-------------------|---------------|
| Jenkins service status | systemctl status jenkins |  |
| Web page accessibility | Access Jenkins via browser |  |
| Administrator login | Log in to Jenkins |  |
| Existence of jobs | Check on the dashboard |  |
| Existence of pipelines | Open pipeline jobs |  |
| Presence of credentials | Manage Credentials |  |
| Credential usability | Test code pulling/image pushing/deployment |  |
| Plugin functionality | Manage Plugins |  |
| Agent availability | Manage Nodes |  |
| GitLab code retrieval | Execute test builds |  |
| Harbor login/push | docker login/push |  |
| Kubernetes deployment | kubectl/helm tests |  |
| Webhook triggering | Test with GitLab pushes |  |
| Notification capability | Email/WeChat Work tests |  |
| Build history access | View past builds |  |
| Permission policy testing | Access for regular users |  |

---

## Section Sixteen: Production Backup Strategy Recommendations

### 16.1 Backup Frequency

| Environment | Backup Frequency | Recommendation |
|-----------------|-------------------|---------------|
| Test Jenkins | Weekly | Lower recovery requirements are acceptable |
| Regular production Jenkins | Daily | At least daily backups are necessary |
| Core deployment Jenkins | Daily or more frequent | Back up immediately after configuration changes |
| Large-scale Jenkins | Tiered backup | High-frequency backups for core configurations, less frequent for historical data |

Recommendations:

    Back up immediately after any configuration change.
    Back up before upgrading plugins.
    Back up before upgrading Jenkins itself.
    Back up promptly before making significant adjustments to a large number of jobs.

---

### 16.2 Backup Retention Policy

Examples:

    Last 7 days: Retain daily
    Last 4 weeks: Retain weekly
    Last 6 months: Retain monthly

Simple cleanup command:

    find /backup/jenkins -type f -mtime +30 -delete

If the backup directory is organized by date:

    find /backup/jenkins -mindepth 1 -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;

---

### 16.3 Backup Security

Jenkins backups may contain sensitive information such as:

- GitLab Tokens
- Harbor passwords
- SSH private keys
- kubeconfig files
- Cloud provider access keys
- Webhook Tokens
- Internal system account credentials
- Deployment scripts
- Internal service addresses

Security measures:

    1. Restrict permissions on backup files.
    2. Do not store backups in web-accessible directories.
    3. Avoid storing backups in Git repositories.
    4. Do not transmit backups via ordinary communication tools.
    5. Prevent unauthorized access to the backup server.
    6. Use SSH/VPN/dedicated lines for remote synchronization.
    7. Encrypt important backups.
    8. Immediately revoke backup server permissions for departing employees.

Example permission settings:

    sudo chown -R root:root /backup/jenkins
    sudo chmod -R 700 /backup/jenkins

---

## Section Seventeen: Jenkins Backup and Recovery Flowchart

    ┌──────────────────────┐
    │   Jenkins Controller  │
    └──────────┬───────────┘
               │
               │ $JENKINS_HOME
               v
    ┌──────────────────────┐
    │   Jobs/Plugins       │Additionally, the pipeline itself should ideally use a Jenkinsfile stored in GitLab, which is known as "Pipeline as Code." This way, even if there are issues with Jenkins, the core logic of the pipeline remains within the code repository, reducing the effort required for recovery.

On the Agent side, it's best to strive for statelessness; components that can be rebuilt should not be considered critical backup targets. In a production environment, backups should also be synchronized to separate machines, and regular recovery drills should be conducted.

---

## Twenty, Production Implementation Suggestions

For small and medium-sized enterprises, the following recommendations are suggested for Jenkins backup implementation:

    1. Define the path for JENKINS_HOME.
    2. Back up JENKINS_HOME regularly every day.
    3. Exclude workspaces and cache that can be rebuilt.
    4. Make sure credentials.xml and secrets/ are backed up.
    5. Keep the plugin directory and a list of installed plugins.
    6. Manually back up configurations before making changes, upgrading plugins, or updating Jenkins itself.
    7. Synchronize backup files to separate machines.
    8. Set appropriate permissions for backup files.
    9. Store Jenkinsfile in GitLab.
    10. Place build artifacts in systems like Harbor/Nexus instead of relying on Jenkins for long-term storage.
    11. Try to make Agents stateless wherever possible.
    12. Conduct regular recovery drills.

---

## Twenty-One, Reference Documents

- Jenkins Docs: Backing-up/Restoring Jenkins
  https://www.jenkins.io/doc/book/system-administration/backing-up/

- Jenkins Docs: Pipeline as Code
  https://www.jenkins.io/doc/book/pipeline/pipeline-as-code/

- Jenkins Docs: Using a Jenkinsfile
  https://www.jenkins.io/doc/book/pipeline/jenkinsfile/

- Jenkins Docs: Pipeline Syntax
  https://www.jenkins.io/doc/book/pipeline/syntax/