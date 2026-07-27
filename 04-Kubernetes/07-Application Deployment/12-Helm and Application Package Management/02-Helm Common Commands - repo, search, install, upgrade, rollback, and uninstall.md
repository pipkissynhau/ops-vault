# 02-Helm Common Commands: repo, search, install, upgrade, rollback, and uninstall

## Documentation Overview
- Document Purpose: Introduction to common Helm operation commands
- Target Audience: Those who have understood the basic concepts of Helm and are ready to learn its routine usage
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/11-Helm and Application Package Management/02-Helm Common Commands: repo, search, install, upgrade, rollback, and uninstall`

## Tags
#Kubernetes #Helm #repo #search #install #upgrade #rollback #uninstall #Chart #Release #application package management #cloud-native #operations and maintenance

---

## I. Why Learn Helm Common Commands Separately

Understanding what Helm is only addresses the “conceptual” level.  
When it comes to practical use, it’s more important to establish a clear operational workflow:

- Where do Charts come from?
- How to find available Charts?
- How to install applications?
- How to view installed Releases?
- How to upgrade configurations or versions?
- How to roll back to previous versions?
- How to uninstall applications?

Therefore, Helm common commands should not be memorized individually but rather understood within the context of a complete application lifecycle.

The focus of this section is not to list all commands but to help you master the most frequently used ones that cover everyday scenarios:

- `repo`
- `search`
- `install`
- `list`
- `status`
- `get values`
- `upgrade`
- `rollback`
- `uninstall`

---

## II. Establishing a Basic Helm Usage Process

You can start by understanding the basic Helm usage process as follows:

### Step 1: Add and Update Repositories
- `helm repo add`
- `helm repo update`

### Step 2: Search for Available Charts
- `helm search repo`

### Step 3: Install Applications
- `helm install`

### Step 4: Check Installation Results
- `helm list`
- `helm status`
- `helm get values`

### Step 5: Modify and Upgrade
- `helm upgrade`

### Step 6: Roll Back in Case of Issues
- `helm rollback`

### Step 7: Uninstall When No Longer Needed
- `helm uninstall`

This process covers the most essential daily tasks with Helm.

---

## III. `helm repo add`: Adding a Chart Repository

### Command Function
This command adds a Helm repository address to your local configuration.

### Common Usage

    helm repo add bitnami https://charts.bitnami.com/bitnami

### What This Command Does
It indicates that:

- The repository alias is `bitnami`
- The repository URL is `https://charts.bitnami.com/bitnami`

### Why This Step Is Necessary
When installing many common components with Helm, you usually don’t start directly from a raw URL. Instead, you first:

- Add the repository.
- Search for the Chart within it.
- Install the Chart from the repository.

### Key Points for Operations and Maintenance Professionals
`helm repo add` essentially means:

> **Registering a specific Chart source in your local Helm environment.**

---

## IV. `helm repo update`: Updating Repository Indexes

### Command Function
This command updates the index information of locally added repositories.

### Common Usage

    helm repo update

### Why This Step Is Needed
When you first add a repository or when its content changes, the local index may not be updated automatically.

Without updating it, you might encounter issues such as:
- Inability to find new Charts.
- Seeing outdated versions.
- Inaccurate version information during installation.

### A Common Sequence
After adding a repository, you typically perform:

    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update

### Key Points for Operations and Maintenance Professionals
`helm repo update` essentially means:

> **Refreshing the local knowledge of remote Helm repository contents.**

---

## V. `helm search repo`: Searching for Charts in Repositories

### Command Function
This command searches available Charts within the added repositories.

### Common Usage

    helm search repo mysql

Or more specifically:

    helm search repo bitnami/mysql

### What This Command Typically Returns
It usually returns:
- Chart name.
- Chart version.
- App version.
- Description information.

### Why This Step Is Important
In Helm usage, it’s common not to remember the full Chart name but instead to first check:
- Whether such a component exists.
- What its name is in the repository.
- What its approximate version is.

### A Typical Scenario
For example, if you want to find nginx:

    helm search repo nginx

Or MySQL:

    helm search repo mysql

### Key Points for Operations and Maintenance Professionals
`helm search repo` essentially means:

> **First confirming whether there are any available application packages in the repository.**

---

## VI. `helm installThe essence of `helm upgrade` can be understood as:

> **Performing a controlled change to an existing Release using new parameters or a newer version.**

---

## Eleven. Why `helm upgrade` is often used in conjunction with the values file

In practical work, writing a long list of parameters directly in the command line is not conducive to maintenance.

Therefore, a more common approach is:

- To write the parameters in the `values.yaml` file
- Then perform the upgrade using `-f values.yaml`

### Common usage

    helm upgrade my-mysql bitnami/mysql -n database -f values.yaml

### Benefits of this approach
- Parameters are more centralized
- Version management becomes easier
- Reusability is improved
- Switching between different environments is simpler

### A typical scenario
For example, if you want to adjust:
- Storage size
- Root password
- Service type
- Resource limits

It is usually more appropriate to modify the `values.yaml` file rather than writing many `--set` options directly in the command line.

### Key points for operations engineers
In many cases, Helm's upgrade process essentially involves:

> **Replacing the parameters and then re-rendering the application.**

---

## Twelve. `helm rollback`: Rolling back to a previous version

### Command function
This command is used to roll back a certain Release to a previous revision.

### Common usage

    helm rollback my-mysql 1 -n database

This means:
- Rolling back `my-mysql` to revision `1`

### Why rollback is important
Because Helm upgrades are not always successful or meet expectations.  
If:
- The values file contains incorrect settings
- The upgraded Chart is incompatible
- New configurations cause application issues

There needs to be a unified way to revert the application to a working state.

### The core value of rollback
It ensures that application management includes not only the ability to upgrade but also:

> **The capability to quickly return to a known working state in case of an upgrade failure.**

### Key points for operations engineers
`helm rollback` is not just a remedial command; it is a crucial part of Helm's application lifecycle management.

---

## Thirteen. What to check before rolling back

Although this is not listed separately in the key commands section, it is usually necessary to review previous versions before performing a rollback.

### Common command

    helm history my-mysql -n database

### What this command provides
- Revision number
- Update time
- Status
- Description

### Why it is useful
Rollback should not be done arbitrarily; instead, you need to confirm:
- Which previous versions are available
- Which version is suitable for your needs
- What changes were made in the most recent upgrade

### Key points for operations engineers
Although `history` is not among the main commands discussed in this chapter, it is a very useful auxiliary command before performing a rollback.

---

## Fourteen. `helm uninstall`: Uninstalling a Release

### Command function
This command is used to uninstall a Release managed by Helm.

### Common usage

    helm uninstall my-mysql -n database

### What this command does
It removes the `my-mysql` Release managed by Helm from the specified namespace.

### Important notes
While the command uninstalls the Release, the cleanup of resources associated with different Charts may vary.  
For example:
- Some objects might be deleted
- Certain PVCs or data volumes may not be automatically cleared
- Some CRDs might not be removed along with the Release

Therefore, after uninstalling, it is still necessary to check:

    kubectl get all -n database
    kubectl get pvc -n database

to ensure that all resources are properly handled.

### Key points for operations engineers
`helm uninstall` essentially means:

> **Removing an application instance managed by Helm from the cluster.**

---

## Fifteen. Creating a complete example workflow with these commands

Below is a basic example of how to use Helm in its entire lifecycle:

### 1. Add a repository

    helm repo add bitnami https://charts.bitnami.com/bitnami

### 2. Update the repository index

    helm repo update

### 3. Search for a Chart

    helm search repo mysql

### 4. Install a Release

    helm install my-mysql bitnami/mysql -n database --create-namespace

### 5. List installed Releases

    helm list -n database

### 6. Check the status of a Release

    helm status my-mysql -n database

### 7. View the current values file

    helm get values my-mysql -n database

### 8. Upgrade the Release

    helm upgrade my-mysql bitnami/mysql -n database -f values.yaml

### 9. Review previous versions if needed

    helm history my-mysql -n database

### 10. Roll back to a previous version if issues