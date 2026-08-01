# DaoCloud Interview Question 02: How to Troubleshoot When Deployment Creates Pod Without Event

## Question Description
Interview Question: When creating a Pod via Deployment, if no Event occurs, how should you troubleshoot?

---

## Core Conclusion
This question examines Kubernetes controller chain troubleshooting, not just Pod itself.

Deployment does not directly create Pod, but works through the following chain:

Deployment → ReplicaSet → Pod

When "no Event" occurs, you should first determine:

- Pod was never created
- Or the queried object is incorrect, scope is wrong
- Or Event has expired but object status remains

Kubernetes official documentation clearly states that Deployment manages Pod replicas through ReplicaSet; ReplicaSet ensures the desired number of Pods exists. ([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com)) ([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/?utm_source=chatgpt.com))

---

## Correct Understanding
Do not focus solely on Pod at first.

The more reasonable troubleshooting order is:

1. First check Deployment
2. Then check ReplicaSet
3. Then check Pod
4. Then check namespace-level Events
5. Then check Pod's status / conditions / containerStatuses
6. If needed, check kubelet / runtime on node side

Focusing only on Pod can easily miss issues in Deployment and ReplicaSet layers.  
Kubernetes official also recommends doing object layering judgment during troubleshooting, distinguishing between workload controller, Pod, or node-side issues. ([kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/?utm_source=chatgpt.com))

---

## How to Answer in Interview

### One-sentence Version
If Deployment creates Pod without Event, I will first troubleshoot along the controller chain rather than directly looking at container: first check if Deployment successfully generated ReplicaSet, then check if ReplicaSet created Pod; if Pod doesn't exist, continue to check Deployment, ReplicaSet and namespace-level Events, and if Pod exists, check Pod's status, conditions and node-side information. ([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com))

### More Detailed Version
Deployment does not directly create Pod, but first creates ReplicaSet, then ReplicaSet maintains Pod replicas.  
So if "no Event", I will first determine whether Pod was actually created.  
Troubleshooting order: first check Deployment status and describe, then check if corresponding ReplicaSet was generated, and whether ReplicaSet's desired / current / ready are normal.  
If Pod doesn't exist, the issue is more likely controller chain or admission layer; if Pod exists, then check Pod's describe, status, conditions, containerStatuses and namespace-level Events.  
If Pod is Pending, investigate scheduling; if Pod is scheduled to node but container doesn't start, check kubelet, container runtime, image pull and volume mount. ([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/?utm_source=chatgpt.com))

---

## Specific Troubleshooting Steps

### 1. First Check Deployment
Confirm Deployment itself is normal:

Common commands:
    kubectl get deploy -n <ns>
    kubectl describe deploy <deploy-name> -n <ns>

Focus on:
- replicas
- updatedReplicas
- availableReplicas
- conditions
- rollout status

Kubernetes documentation clearly states that Deployment's status and conditions can be used to judge rollout anomalies or stalls. ([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com))

#### Issues to Pay Attention To
- Are replicas 0?
- Do selector and template labels match?
- Is rollout timeout occurred?
- Is it blocked by admission / quota / policy?

---

### 2. Then Check ReplicaSet
Deployment's next layer is ReplicaSet.  
Without checking ReplicaSet, the troubleshooting chain is broken.

Common commands:
    kubectl get rs -n <ns>
    kubectl describe rs <rs-name> -n <ns>

ReplicaSet's responsibility is to ensure specified number of Pods exist. ([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/?utm_source=chatgpt.com))

#### Issues to Pay Attention To
- Did Deployment successfully create ReplicaSet?
- Are ReplicaSet's desired / current / ready consistent?
- Does ReplicaSet have Pod creation failure information?

#### Typical Judgments
- Deployment exists but ReplicaSet doesn't: more likely Deployment/controller/admission issue
- ReplicaSet exists but Pod doesn't: more likely ReplicaSet failed to create Pod

---

### 3. Confirm Pod Actually Exists
Many "no Event" scenarios essentially mean Pod wasn't created.

Common commands:
    kubectl get pod -n <ns> -o wide

#### Two Scenarios

##### Scenario A: No Pod Exists
Then don't say "Pod has no Event", but instead:

Pod hasn't been created successfully, continue to troubleshoot Deployment/ReplicaSet/namespace-level Events.

##### Scenario B: Pod Already Exists
If Pod exists, continue to check Pod's detailed status.

---

### 4. Check Pod's Detailed Status
Common commands:
    kubectl describe pod <pod-name> -n <ns>
    kubectl get pod <pod-name> -n <ns> -o yaml

Focus on:
- phase
- conditions
- initContainerStatuses
- containerStatuses
- nodeName

Kubernetes official documentation states that Pod status, conditions, and containerStatuses are critical information for determining runtime status. Even if Events are not obvious, these fields can help locate issues.([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/?utm_source=chatgpt.com))

---

### 5. Check if Event query scope is correct  
Often, the issue isn't the absence of Events, but an incorrect query scope.

Common commands:  
    kubectl get events -n <ns> --sort-by=.lastTimestamp  
    kubectl events -n <ns>  
    kubectl events -n <ns> --for deployment/<deploy-name>  
    kubectl events -n <ns> --for pod/<pod-name>  

Kubernetes official documentation states that `kubectl events` supports filtering Events by namespace and object.([kubernetes.io](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_events/?utm_source=chatgpt.com))

#### Key understanding here  
Failure information may not be attached to the Pod itself, but could be attached to:  
- Deployment  
- ReplicaSet  
- Other related objects within the namespace scope  

---

### 6. Continue forked troubleshooting based on Pod status  

#### If Pod remains Pending  
Focus on scheduling-related issues:  
- Resource insufficiency (CPU / memory)  
- nodeSelector / affinity mismatch  
- taints / tolerations mismatch  
- PVC not bound  
- ResourceQuota / LimitRange restrictions  

Kubernetes official debugging documentation explicitly identifies scheduling failure as a common cause for Pending status.([kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/?utm_source=chatgpt.com))

#### If Pod is scheduled to node but containers fail to start  
Focus on:  
- Image pull failure  
- Volume mount failure  
- kubelet anomalies  
- Container runtime anomalies  
- Startup command errors  

At this point, you can further combine:  
    kubectl logs <pod-name> -n <ns> --previous  

---

### 7. Inspect node-side components if necessary  
If Pod already has `nodeName`, it indicates scheduling is largely complete.  
Next, the fault may lie on the node-side:  

- kubelet  
- container runtime  
- Node network  
- Storage mounting  
- CNI  

Kubernetes debugging documentation also recommends checking node-side components when Pod is bound to a node.([kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/?utm_source=chatgpt.com))

---

## Key knowledge points  

### 1. Deployment does not directly create Pod  
Deployment manages ReplicaSet, which in turn maintains Pod count.  
This is the core understanding of the chain in this question.([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com))

### 2. Events are not the sole basis  
No Events does not mean there is no problem.  
Events are just auxiliary troubleshooting information; status fields, conditions, and describe are equally important.([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/?utm_source=chatgpt.com))

### 3. Prioritize layering, then drill down  
The correct order is not "check application logs first," but:  

Deployment → ReplicaSet → Pod → Status / Event → Node  

---

## Common mistakes  

### Mistake 1: Focus only on Pod immediately  
This may miss issues at the Deployment and ReplicaSet levels.  

### Mistake 2: Interpret "no Events" as "no problem"  
Incorrect.  
It could mean the object was never created, or the wrong object is being checked, or Events have expired.  

### Mistake 3: Ignore ReplicaSet  
If the answer lacks ReplicaSet, it indicates incomplete understanding of the Deployment control chain.([kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/?utm_source=chatgpt.com))

### Mistake 4: Fail to distinguish between Pending and scheduled but not started  
These two types of issues have completely different troubleshooting directions:  
- Pending leans more toward scheduling  
- Scheduled but not started leans more toward kubelet / runtime / image / mount  

---

## Interview oral template  
If Pod is created via Deployment but no Events are present, I would not immediately focus on the container. Instead, I would first troubleshoot along the control chain.  
Because Deployment itself does not directly create Pod; it first creates ReplicaSet, which then manages Pod replicas.  
So I would first check the status and describe of Deployment, then verify if corresponding ReplicaSet is generated, and whether ReplicaSet continues to create Pod.  
If Pod is not created at all, the issue is more likely related to controller chain, admission, or resource limits; if Pod is created, I would then check Pod's describe, status, conditions, and namespace-level Events.  
If Pod remains Pending, I would focus on scheduling; if Pod is scheduled to node but containers fail to start, I would further check kubelet, container runtime, image pull, volume mount, and node logs.([kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/?utm_source=chatgpt.com))

---

## Memory template  
First remember this main line:  

Deployment  
→ ReplicaSet  
→ Pod  
→ Event / Status  
→ Node  

Then remember two forked paths:  

- No Pod: First check controller chain  
- With Pod: Then check scheduling, startup, and node  

---

## Tags  
#Kubernetes  
#Deployment  
#ReplicaSet  
#Pod  
#Event  
#FaultCheck.  
#TheYunwonInterview.  
#TransportInterview  
#Controller  
#kubelet