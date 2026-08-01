# Obsidian Graph View Configuration Guide

This document records the **recommended Graph View configuration for the ops-skill-tree knowledge base**, ensuring the knowledge graph is clear, structurally stable, and has distinct modular layers.

---

# I. Graph View Objectives

The objective of the Graph view is not "aesthetics", but:

- Observe knowledge base structure
    
- Discover knowledge gaps
    
- Quickly locate modules
    
- Form a technical system
    
    

Ideal structure resembles:

```
           Kubernetes
          /     |      \
       Pod   Service   PV
        |      |        |
     Calico  Ingress  Longhorn
```

---

# II. Graph Force Parameters (Forces)

Open Graph's top-right settings → **Forces**

Recommended parameters:

```
Central:0.30
Links:1.00
Exclusion:16
Link distance:200
```

If the graph becomes crowded:

```
Exclusion:18
```

If the graph becomes scattered:

```
Exclusion:12
```

Parameter explanations:

|Parameter|Function|
|---|---|
|Center Force|Controls node aggregation toward the center|
|Link Force|Controls connection strength between nodes|
|Repulsion|Controls spacing between nodes|
|Link Distance|Controls distance between modules|

---

# III. Display (Display Settings)

Recommended settings:

```
Show arrow: Close
Show attachments: Close
Show Isolated Nodes: Open
Dilution unconnected nodes: open
Node size: Medium
Text size: Medium
```

Explanations:

- To observe all knowledge nodes in the current phase, **isolated nodes are enabled**
    
- Disabling attachments avoids Graph clutter
    

---

# IV. Filters (Filter Settings)

Recommended:

```
Label: Open
Annex: Closed
Unsolved Link: Close
Isolated Node: Open
```

Explanations:

- Tags are used for Graph coloring
    
- Disabling unresolved links reduces interference
    

---

# V. Graph Color Grouping Rules

Graph colors only use **primary tags**.

Primary tags include:

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

Color reference:

See file:

```
99-Knowledge base management/Level 1 Tab Color Table.md
```

Rules:

- Primary tags → Graph coloring
    
- Secondary tags → Used for search
    
- Tertiary tags → Technical subcategories, not involved in coloring
    

---

# VI. Graph Search Filtering

To avoid management files entering the Graph, recommend entering in the Graph search bar:

```
-path:"99-Knowledge base management" -path:"scripts" -path:"attachments"
```

Effect:

The Graph only displays technical knowledge nodes.

---

# VII. Common Graph Viewing Methods

## 1. Global Knowledge Graph

Purpose:

- View overall knowledge structure
    
- View module scale
    
    

Search bar:

```
-path:"99-Knowledge base management"
```

---

## 2. View Kubernetes Module

```
tag:#kubernetes
```

You can see:

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

## 4. View Storage System

```
tag:#storage
```

---

# VIII. Final Graph Structure

The knowledge graph will eventually form a **technical galaxy structure**:

```
Linux
 ├─Network
 ├─Storage
 ├─System
 └─Error

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

Each module is an independent knowledge center.

---

# IX. Graph Usage Recommendations

Daily usage:

1. Write notes
    
2. Add tags
    
3. Add [[Internal links]]
    
4. View Graph
    
    

Avoid frequent structural adjustments.

The Graph will automatically optimize as the knowledge base grows.

---

# X. Knowledge Base Development Stages

Current stage:

```
Structure building
```

Next stage:

```
Fill technical content
```

Final stage:

```
Developing an integrated knowledge system
```

---

# XI. Graph Usage Principles

The Graph is an **auxiliary tool**, not the core.

Core is:

- Knowledge structure
    
- Technical understanding
    
- Practical experience