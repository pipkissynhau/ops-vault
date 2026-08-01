# 02-Helm Common Commands: repo, search, install, upgrade, rollback, and uninstall

## Document Overview
- Document positioning: Helm basic operation command introduction
- Applicable stage: After understanding Helm basic concepts, entering Helm common usage flow
- Recommended path: `04-Kubernetes/07-Apply deployment/11-Helm and application package management/02-Helm Common command:repoI don't know.searchI don't know.installI don't know.upgradeI don't know.rollback and uninstall`

## Tags
#Kubernetes #Helm #repo #search #install #upgrade #rollback #uninstall #Chart #Release #ApplyPackageManagement #Clouds. #Transport

---

## I. Why Learn Helm Common Commands Separately

Understanding what Helm is can only solve "conceptual" issues.  
After entering the usage phase, it's more important to establish a clear operational mainline:

- Where do Charts come from
- How to find available Charts
- How to install applications
- How to view already installed Releases
- How to upgrade configurations or versions
- How to rollback to historical versions
- How to uninstall applications

Therefore, Helm's common commands shouldn't be memorized as scattered fragments, but are better understood as a complete application lifecycle.

This article's focus isn't to exhaustively list all commands, but to first master the most commonly used commands that can cover daily scenarios:

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

## II. Establish the Most Basic Helm Usage Flow

You can first understand Helm's common usage flow as the following chain:

### Step 1: Add and Update Repository
- `helm repo add`
- `helm repo update`

### Step 2: Find Available Charts
- `helm search repo`

### Step 3: Install Application
- `helm install`

### Step 4: View Installation Results
- `helm list`
- `helm status`
- `helm get values`

### Step 5: Modify and Upgrade
- `helm upgrade`

### Step 6: Rollback When Issues Occur
- `helm rollback`

### Step 7: Uninstall When No Longer Needed
- `helm uninstall`

This chain can basically cover Helm's most core daily usage scenarios.

---

## III. `helm repo add`: Add Chart Repository

### Command Purpose
Used to add a Helm repository address to the local Helm configuration.

### Common Usage

    helm repo add bitnami https://charts.bitnami.com/bitnami

### What This Command Expresses
It indicates:

- Repository alias is `bitnami`
- Repository address is `https://charts.bitnami.com/bitnami`

### Why This Step is Needed
Because Helm installs many common components not directly from a raw URL, but typically:

- Add repository
- Then search for Chart in the repository
- Then install Chart from the repository

### Operations Understanding Focus
The significance of `helm repo add` can be understood as:

> **Registering a Chart source in the local Helm environment.**

---

## IV. `helm repo update`: Update Repository Index

### Command Purpose
Used to update the local index information of added repositories.

### Common Usage

    helm repo update

### Why This Step is Needed
Because when a repository is first added or its content is updated, the local environment won't automatically synchronize the latest index.

Without updating, you may encounter:
- Unable to find new Charts
- Seeing outdated versions
- Inaccurate version information during installation

### A Common Combination
Typically, after adding a repository, you'll immediately execute:

    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update

### Operations Understanding Focus
The significance of `helm repo update` can be understood as:

> **Refresh the local understanding of remote Helm repository content.**

---

## V. `helm search repo`: Search for Charts in Repository

### Command Purpose
Used to search for available Charts in added repositories.

### Common Usage

    helm search repo mysql

Or more specifically:

    helm search repo bitnami/mysql

### What This Command Typically Returns
It usually returns:
- Chart name
- Chart version
- App version
- Description information

### Why This Step is Important
Because in Helm usage, it's common not to remember the full Chart name, but to first check:
- Whether the component exists
- What it's called in the repository
- What version it is approximately

### A Typical Scenario
For example, looking for nginx:

    helm search repo nginx

Looking for MySQL:

    helm search repo mysql

### Operations Understanding Focus
The significance of `helm search repo` can be understood as:

> **Confirm whether the repository contains available application packages.**

---

## VI. `helm install`: Install a Release

### Command Purpose
Used to install an actual Release from a Chart into the cluster.

### Common Usage

    helm install my-mysql bitnami/mysql

This command indicates:

- Release name is `my-mysql`
- Used Chart is `bitnami/mysql`

### Why Install is Critical
Because it marks Helm's transition from the "repository and Chart stage" to the "actual installation instance stage."

In other words:
- `bitnami/mysql` is a Chart
- `my-mysql` is a Release

### Common Usage with Namespace

    helm install my-mysql bitnami/mysql -n database --create-namespace

This indicates:
- Install in the `database` namespace
- Automatically create the namespace if it doesn't exist

### Operations Understanding Focus
The essence of `helm install` can be understood as:

> **Install a Chart as specified with a name and parameters into a Kubernetes cluster.**

---

## VII. Why Look at `helm list` After Installation

### Command Purpose
Used to view currently installed Releases.

### Common Usage

    helm list

If you need to view a specific namespace:

    helm list -n database

If you need to view all namespaces:

    helm list -A

### What You Usually See with This Command
Typically you can see:
- Release name
- Namespace
- Revision (REVISION)
- Update time
- Status
- Chart name
- App version

### Why This Step Is Important
Because after installation, you need to confirm first:
- Whether it was installed successfully
- Which namespace it was installed in
- Whether the current status is `deployed`

This is more reliable than guessing the object status directly.

### Operations Understanding Focus
The meaning of `helm list` can be understood as:

> **View the list of Helm-managed application instances in the current cluster.**

---

## VIII. `helm status`: Check the Status of a Specific Release

### Command Purpose
Used to check the current status and summary information of a specific Release.

### Common Usage

    helm status my-mysql -n database

### Common Information Seen
Usually includes:
- Release name
- Namespace
- Current status
- Revision number
- Last update time
- Rendered resource summary
- Sometimes also includes NOTESHint information

### Why It's More Detailed Than `helm list`
`helm list` is more like an overview.  
`helm status` is more like viewing details of a specific Release.

### Operations Understanding Focus
If `helm list` is "viewing the directory",  
then `helm status` is more like "opening a specific application instance to check its current status".

---

## IX. `helm get values`: View Parameters Used by Current Release

### Command Purpose
Used to view the values currently used by a Release.

### Common Usage

    helm get values my-mysql -n database

If you want to see more complete information, you can use:

    helm get values my-mysql -n database -a

### Why This Command Is Important
One of the core uses of Helm is values.

Often, when troubleshooting or reviewing a Release, you need to first confirm:
- What parameters were actually passed initially
- What changes were made compared to default values
- What changes occurred between upgrades and current values

### Operations Understanding Focus
The meaning of `helm get values` can be understood as:

> **View what parameters were actually used to install this Release.**

---

## X. `helm upgrade`: Upgrade a Release

### Command Purpose
Used to upgrade an already installed Release.

Upgrades may include:
- Upgrading Chart version
- Modifying values
- Modifying image tag
- Modifying replica count
- Modifying Service type
- Modifying resource limits

### Common Usage

    helm upgrade my-mysql bitnami/mysql -n database

If using a values file:

    helm upgrade my-mysql bitnami/mysql -n database -f values.yaml

### What This Command Represents
It indicates:
- Updating an existing `my-mysql`
- Continuing to use `bitnami/mysql` as the Chart
- The new rendered result is determined by new parameters

### Why Upgrade Is Critical
Helm's "application package management" value isn't just in installation, but also in subsequent changes.

### Operations Understanding Focus
The essence of `helm upgrade` can be understood as:

> **Perform a controlled change on an existing Release using new parameters or a new version.**

---

## XI. Why `helm upgrade` Is Often Used with Values Files

In actual work, writing long parameter strings directly in the command line is not conducive to maintenance.

Therefore, a more common approach is:
- Write parameters in `values.yaml`
- Then use `-f values.yaml` for upgrades

### Common Usage

    helm upgrade my-mysql bitnami/mysql -n database -f values.yaml

### Benefits of This Approach
- Parameters are more centralized
- Easier version management
- More reusable
- Easier to switch between environments

### A Typical Scenario
For example, wanting to adjust:
- Storage size
- Root password
- Service type
- Resource limits

It's usually more suitable to modify the values file rather than directly concatenating many `--set` in the command line.

### Operations Understanding Focus
Helm's upgrade operation often essentially means:

> **Change a set of parameters and re-render the application.**

---

## XII. `helm rollback`: Rollback to a Historical Version

### Command Purpose
Used to rollback a specific Release to a historical revision version.

### Common Usage

    helm rollback my-mysql 1 -n database

This indicates:
- Rollback `my-mysql` to revision `1`

### Why Rollback Is Important
Because Helm upgrades aren't always successful or as expected.  
If:
- Values were modified incorrectly
- Chart upgrade is incompatible
- New configuration causes application anomalies

There needs to be a unified way to revert to a previously known working state.

### Core Value of Rollback
It makes application management not just "able to upgrade", but also:

> **Quickly return to a known working state after an upgrade failure.**

### Operations Understanding Focus
`helm rollback` isn't just a remedial command, but a critical part of Helm's application lifecycle management.

---

## XIII. What to Check Before Rolling Back

Although this section's main command list doesn't include it separately, you usually check historical versions before rolling back.

### Common Commands

    helm history my-mysql -n database

### What This Command Usually Shows
- Revision number
- Update time
- Status
- Description information

### Why It's Useful
Because rollback isn't just writing a random number, but you need to first confirm:
- What historical versions exist
- Which version is a working version
- What happened in the last upgrade

### Operations Understanding Focus
Although `history` isn't in the main commands of this section's title, it's a very practical auxiliary command before rollback.

---

## XIV. `helm uninstall`: Uninstall a Release

### Command Purpose
Used to uninstall a Release.

### Common Usage

    helm uninstall my-mysql -n database

### What This Command Represents
Indicates removing the Helm-managed `my-mysql` from the specified namespace.

### What to Be Aware Of
Uninstalling is for Helm-managed Releases, but different Chart resource cleanup behaviors may vary.  
For example:
- Some objects will be deleted
- Some PVCs or data volumes may not be automatically cleared
- Some CRDs may not be deleted with the Release

Therefore, after uninstallation, it's still recommended to combine with: /think

kubectl get all -n database  
kubectl get pvc -n database  

Confirm that the resources match expectations.  

### Key Operational Understanding  
`helm uninstall` can be understood as:  

> **Remove a Helm-managed application instance from the cluster.**  

---  

## Fifteen. Turn These Commands into a Complete Workflow Example  

The following provides the most basic Helm usage cycle:  

### 1. Add a Repository  

    helm repo add bitnami https://charts.bitnami.com/bitnami  

### 2. Update Repository Index  

    helm repo update  

### 3. Search for Charts  

    helm search repo mysql  

### 4. Install a Release  

    helm install my-mysql bitnami/mysql -n database --create-namespace  

### 5. View Installed Releases  

    helm list -n database  

### 6. Check Release Status  

    helm status my-mysql -n database  

### 7. View Current Values  

    helm get values my-mysql -n database  

### 8. Upgrade a Release  

    helm upgrade my-mysql bitnami/mysql -n database -f values.yaml  

### 9. Optionally Check History  

    helm history my-mysql -n database  

### 10. Rollback When Issues Occur  

    helm rollback my-mysql 1 -n database  

### 11. Uninstall When No Longer Needed  

    helm uninstall my-mysql -n database  

This workflow covers Helm's core daily operations.  

---  

## Sixteen. Common Points of Confusion with These Commands  

### 1. Confusion Between Chart and Release  
For example:  

    helm install my-mysql bitnami/mysql  

Here:  
- `my-mysql` is the Release  
- `bitnami/mysql` is the Chart  

### 2. Confusion Between `repo update` and `upgrade`  
- `repo update` updates the repository index  
- `upgrade` upgrades an installed Release  

### 3. Confusion Between `list` and `status`  
- `list` views the overview  
- `status` views details of a single Release  

### 4. Confusion Between `get values` and values Files  
- values files are the source of input parameters  
- `helm get values` views the parameters actually used by the Release  

### 5. Confusion Between `rollback` and `uninstall`  
- `rollback` rolls back to an old version  
- `uninstall` deletes the entire Release  

---  

## Seventeen. The Most Important Understandings in This Section  

### 1. Helm Commands Should Be Remembered by Application Lifecycle  
This is the first key understanding.  

### 2. `repo` and `search` Address "Where Does the Chart Come From"  
This is the second key understanding.  

### 3. `install`, `list`, `status`, and `get values` Address "What Is the Current State of the Application"  
This is the third key understanding.  

### 4. `upgrade` and `rollback` Address "How to Change and Rollback the Application"  
This is the fourth key understanding.  

### 5. `uninstall` Addresses "How to Remove the Application as a Whole"  
This is the fifth key understanding.  

---  

## Eighteen. Summary by Stage  

Although Helm's common commands aren't numerous, they are not isolated from each other.  
A more reasonable understanding is to place them within a complete application lifecycle:  

- Repository management  
- Chart search  
- Release installation  
- Status checks  
- Parameter confirmation  
- Upgrade changes  
- History rollback  
- Uninstallation cleanup  

Once this chain is clarified, the next steps naturally lead to:  

- The actual role of values.yaml  
- Customizing Helm deployment results through parameters  
- Practical Helm installation, upgrade, and rollback scenarios  

---  

## Nineteen. Keyword Mnemonics  

- `helm repo add`: Add a repository  
- `helm repo update`: Update repository index  
- `helm search repo`: Search for Charts in the repository  
- `helm install`: Install a Release  
- `helm list`: View installed Releases  
- `helm status`: View status of a single Release  
- `helm get values`: View parameters used by the current Release  
- `helm upgrade`: Upgrade a Release  
- `helm rollback`: Rollback a Release  
- `helm uninstall`: Uninstall a Release  

---  

## Twenty. Operational Extension Understanding  

The value of Helm's common commands lies not just in memorizing operations, but in completing a transition of "application lifecycle management" within the overall context.  

In previous content, the focus was more on:  

- How to write objects  
- How to combine objects  

At the Helm command level, the focus shifts to:  

- How to install an application  
- How to view an application  
- How to modify an application  
- How to rollback an application  
- How to remove an application  

In other words, the perspective has shifted from:  

> **Managing objects**  

To:  

> **Managing application instances**  

This is precisely where Helm adds the most value in Kubernetes operations and delivery scenarios.  

---  

## References  
- Helm Official Documentation  
- Kubernetes Official Documentation  

---  

## Next Day Recommendation  
The next article suggests organizing:  

**"03-values.yaml Basics: Customizing Helm Deployment Parameters Through Configuration"**