# One: Remember a Summary Sentence

These three are essentially all **application deployment strategies**, differing in:

- **Rolling Update**: Replace old version Pods with new version Pods in batches
- **Blue-Green Deployment**: Prepare two complete environments, switch traffic all at once from old version to new version
- **Canary Release**: New and old versions run simultaneously, first letting part of the traffic access the new version, then gradually expanding

---

# Two: What is Rolling Update

## 1. Definition
Rolling Update (Rolling Update) is the most common deployment method in Kubernetes by default.

Its core idea is:

**Not deleting all old Pods at once and starting new Pods, but instead starting new Pods and deleting old Pods simultaneously, gradually completing the version replacement.**

## 2. Example
For example, a Deployment currently has 4 v1 Pods.

When upgrading to v2, K8s might:
- Start 1 v2 Pod first
- Delete 1 v1 Pod
- Start 1 v2 Pod again
- Delete 1 v1 Pod

Until all replacements are completed.

## 3. Advantages
- Native support, simplest
- No need for two complete environments
- Business usually doesn't interrupt during deployment
- Relatively low resource cost

## 4. Disadvantages
- Old and new versions run together for a period
- If new version has issues, the impact may gradually expand
- Rollback is less direct than Blue-Green
- Not suitable for particularly complex compatibility scenarios

## 5. How K8s Implements
Mainly through:

- `Deployment`
- Update strategy `strategy.type: RollingUpdate`
- Parameters:
  - `maxUnavailable`
  - `maxSurge`

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1  
```

### Parameter Explanation

- `maxUnavailable`: Maximum number of old Pods that can be unavailable during upgrade
- `maxSurge`: Maximum number of new Pods that can exceed the desired replica count during upgrade

---

# Three: What is Blue-Green Deployment

## 1. Definition

Blue-Green Deployment (Blue-Green Deployment) refers to:

**Preparing two nearly identical runtime environments simultaneously, one running the old version and one running the new version, then switching traffic all at once to the new version after verifying it works.**

Common terminology:

- Blue: Old version
- Green: New version

## 2. Example

Current production runs on v1 (Blue).  
You deploy a new v2 (Green) environment, not yet receiving real traffic.  
After testing confirms it's fine, switch the Service or Ingress to point to v2.  
If issues occur, switch back to v1.

## 3. Advantages

- Fast switching
- Fast rollback
- Controllable risk
- Can fully validate new environment before deployment

## 4. Disadvantages

- High resource cost
- Need to maintain two environments
- Extra care needed for database change scenarios

## 5. How K8s Implements

K8s typically implements this through combinations of:

- Two sets of `Deployment`
- One `Service` for switching
- Or through `Ingress` switching backend
- More advanced scenarios can combine:
    - Argo Rollouts
    - Istio
    - Helm

## 6. Common Implementation Approaches

### Approach One: Service Switching

Example:

- `myapp-v1` Deployment
- `myapp-v2` Deployment

Original Service selects v1:

selector:  
  app: myapp  
  version: v1

Switching changes to:

selector:  
  app: myapp  
  version: v2

Traffic then switches from old version to new version.

### Approach Two: Ingress Switching

If traffic first goes through Ingress, you can also switch traffic by modifying the Ingress backend service to direct traffic from old service to new service.

---

# Four: What is Canary Release

## 1. Definition

Canary Release (Canary / Progressive Delivery) refers to:

**New and old versions coexist, first letting a small amount of real traffic enter the new version, observing it works, then gradually increasing the proportion until fully replacing the old version.**

## 2. Example

For example:

- 90% traffic goes to v1
- 10% traffic goes to v2

After observing 10 minutes with no issues:

- 70% traffic goes to v1
- 30% traffic goes to v2

After further observation:

- 50% / 50%
- Finally switch 100% to v2

## 3. Advantages

- Lowest risk
- Can let a small number of users validate the new version first
- Suitable for core business
- Can combine with monitoring for automatic scaling or automatic rollback

## 4. Disadvantages

- Complex implementation
- High requirements for traffic management capabilities
- Requires more mature monitoring, alerting, and rollback mechanisms

## 5. How K8s Implements

Gray release typically cannot be fully implemented with native Service alone, usually requiring:

- `Ingress Controller`
- `Nginx Ingress Canary`
- `Istio`
- `Argo Rollouts`
- `Flagger`

---

# Five: Which of the Three Can K8s Native Deployment Implement

## 1. Native is best at Rolling Update

Kubernetes native Deployment defaults to Rolling Update.

This is the most common and basic upgrade method.

## 2. Can Native Do Blue-Green?

Yes, but not "out-of-the-box automatic Blue-Green".

Generally needs to prepare:

- Two sets of Deployment
- Manually modify Service or Ingress to switch traffic

## 3. Can Native Do Canary?

Can do "partial effect like canary" deployment, but not precise.

Because Deployment RollingUpdate has new and old Pods coexisting, Service also directs traffic to both new and old Pods.  
But this traffic distribution is more like proportional by Pod count, not strict business traffic ratio control.

So strictly speaking:

- **Deployment RollingUpdate is not standard canary release**
- It's just "rolling upgrade with staged replacement effect"

---

# Six: Core Differences Between the Three

| Comparison Item | Rolling Update | Blue-Green Deployment | Canary Release |
|---|---|---|---|
| Do new and old versions coexist | Yes | Yes | Yes |
| Traffic switching method | Naturally changes with Pod replacement | One-time full switch | Gradually switch by proportion |
| Do you need two complete environments | Not necessarily | Usually needed | Usually needs some extra resources |
| Rollback speed | Moderate | Fast | Moderate to fast |
| Implementation complexity | Lowest | Moderate | Highest |
| Resource consumption | Low | High | Medium to high |
| Risk control | General | Better | Best |
| K8s Native Support | Strongest | Needs combination implementation | Usually needs additional components |

---

# Seven: How to Answer Steadily in an Interview

## Standard Answer

In Kubernetes, rolling update, blue-green deployment, and canary release all belong to application deployment strategies.

Rolling update is the default supported method by Kubernetes Deployment. Its characteristics are gradually replacing versions by creating new version Pods and deleting old version Pods simultaneously. The advantages are simplicity and relatively low resource cost, but if the new version has issues, the impact may gradually expand during the replacement process.

Blue-green deployment prepares two complete environments in advance, one running the old version and one running the new version. After verifying the new version works, traffic is switched all at once through Service or Ingress. Its advantages are fast switching and rollback, but the resource cost is relatively high.

Gray release is to have new and old versions online simultaneously, allowing a small amount of real traffic to enter the new version first, observing monitoring, logs, error rates, and latency, confirming there are no issues before gradually increasing the traffic proportion. It has the lowest risk but is the most complex to implement, typically requiring components like Ingress, Istio, or Argo Rollouts for more precise traffic control.

If it's just a regular business upgrade, Kubernetes native Deployment's RollingUpdate is usually sufficient; but for core business or high-risk changes, blue-green release or gray release is generally considered.

---

# Eight. Keywords You Need to Remember

## Rolling Update Keywords

- Deployment Default Strategy
- Gradual Pod Replacement
- `maxUnavailable`
- `maxSurge`

## Blue-Green Release Keywords

- Two Environments
- One-Time Traffic Switch
- Fast Rollback
- Service / Ingress Switch

## Gray Release Keywords

- Small Traffic Trial Run
- Gradual Traffic Scaling
- Traffic Ratio Control
- Ingress / Istio / Argo Rollouts

---

# Nine. Interview Bonus Answer

You can add a sentence:

> I understand that Kubernetes native most common is RollingUpdate, which is suitable for most routine releases; but if it's core business, large impact scope, or high rollback speed requirements, blue-green release is usually considered; if you want to first test the new version with small traffic and control risk lower, gray release will be adopted, combining Ingress or Service Mesh for more precise traffic management.

---

# Ten. Memorization Version

## Rolling Update

K8s Deployment default release method, gradually replacing pods by starting new ones and deleting old ones.

## Blue-Green Release

Prepare two environments, old and new versions coexist, switch traffic all at once after verification, fast rollback but high cost.

## Gray Release

New and old versions online simultaneously, give small traffic to the new version first, observe and confirm no issues before gradually scaling up, lowest risk but most complex to implement.

## K8s Implementation Methods

- Rolling Update: Deployment's RollingUpdate
- Blue-Green Release: Two Deployments + Service/Ingress Switch
- Gray Release: Ingress, Istio, Argo Rollouts, Nginx Canary, etc. traffic control capabilities