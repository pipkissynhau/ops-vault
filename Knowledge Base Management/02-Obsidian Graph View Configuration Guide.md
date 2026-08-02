# Obsidian Graph View Configuration Guide

This document records the **recommended configurations for the Graph View in the ops-skill-tree knowledge base**, ensuring that the knowledge graph is clear, structurally stable, and has well-defined module hierarchies.

---

# I. Goals of the Graph View

The purpose of the Graph View is not to look visually appealing, but rather to:

- Observe the structure of the knowledge base
- Identify knowledge gaps
- Quickly locate specific modules
- Establish a technical system framework

An ideal structure might look like this:

```
           Kubernetes
          /     |      \
       Pod   Service   PV
        |      |        |
     Calico  Ingress  Longhorn
```

---

# II. Graph Forces Parameters

Open the settings at the top-right corner of the Graph → **Forces**

Recommended parameters:

```
Central force: 0.30
Link force: 1.00
Repulsion force: 16
Link distance: 200
```

If the graph is too crowded:

```
Repulsion force: 18
```

If the graph is too scattered:

```
Repulsion force: 12
```

Parameter explanations:

| Parameter | Function |
|-----------|----------|
| Central force | Controls node aggregation towards the center |
| Link force | Determines the strength of connections between nodes |
| Repulsion force | Regulates the spacing between nodes |
| Link distance | Sets the distance between modules |

---

# III. Display Settings

Recommended settings:

```
Show arrows: Off
Show attachments: Off
Show isolated nodes: On
Diminish disconnected nodes: On
Node size: Medium
Text size: Medium
```

Explanation:

- At this stage, it is important to observe all knowledge nodes, so **isolated nodes should be displayed**.
- Turning off attachments helps keep the Graph clutter-free.

---

# IV. Filters Settings

Recommendations:

```
Tags: On
Attachments: Off
Unresolved links: Off
Isolated nodes: On
```

Explanation:

- Tags are used to color code elements in the Graph.
- Disabling unresolved links reduces visual distractions.

---

# V. Graph Color Scheme Rules

The Graph uses only **first-level tags** for coloring.

First-level tags include:

```
#windows
#linux
#docker
#kubernetes
#storage
#network
#middleware
#cicd
#monitor
#log
#gpu
#virtualization
#cloud
#security
#cni
#tools
#pve
#sre
#incident
#runbook
```

Color references:

See the file:

```
99-Knowledge Base Management/First-Level Tag Color Scheme.md
```

Rules:

- First-level tags → Used for Graph coloring
- Second-level tags → Used for search purposes
- Third-level tags → Used for technical subdivision but not for coloring

---

# VI. Graph Search Filtering

To prevent management files from appearing in the Graph, it is recommended to use the following filter in the search bar:

```
-path:"99-Knowledge Base Management" -path:"scripts" -path:"attachments"
```

Effect:

The Graph will only display technical knowledge nodes.

---

# VII. Common Ways to Use the Graph

## 1. Global Knowledge Graph

Use case:

- View the overall structure of the knowledge base
- Assess the scale of different modules

Search bar:

```
-path:"99-Knowledge Base Management"
```

---

## 2. View Kubernetes Modules

```
tag:#kubernetes
```

This will display:

- Pod
- Deployment
- Service
- Ingress
- PV
- PVC
- StorageClass

---

## 3. View Linux Networking

```
tag:#linux tag:#network
```

---

## 4. View the Storage System

```
tag:#storage
```

---

# VIII. Final Goal Structure of the Graph

The knowledge graph will eventually take on a **technical galaxy structure**:

```
Linux
 ├─Network
 ├─Storage
 └─System
 └─Troubleshooting

Kubernetes
├─Pod
 ├─Service
 ├─Ingress
 └─Storage

Storage
 ├─LVM
 ├─NFS
 ├─Ceph
 └─Longhorn
```

Each module serves as an independent knowledge center.

---

# IX. Tips for Using the Graph

Daily usage recommendations:

1. Take notes
2. Add tags
3. Include [[internal links]]
4. Consult the Graph regularly

Avoid frequently adjusting the graph structure, as it will automatically optimize as the knowledge base grows.

---

# X. Stages of Knowledge Base Development

Current stage:

```
Structure building
```

Next stage:

```
Filling with technical content
```

Final stage:

```
Forming a complete operations and maintenance knowledge system
```

---

#