# Writing Specifications for the Ops Knowledge Base

This document is designed to standardize the formatting of notes in the ops-skill-tree knowledge base.

---

# 1 Note Naming

Use **technical terms** as file names.

Correct examples:

```
Pod.md
Deployment.md
LVM.md
NFS.md
Calico.md
MySQL.md
```

Avoid:

```
k8s-notes.md
linux-learning.md
mysql-records.md
```

---

# 2 Required Tags

Each note must include:

```
Primary Tag
Secondary Tag
```

Example:

```
#kubernetes
#storage
#longhorn
```

---

# 3 Use a Unified Template

All notes should follow the same structure:

```
#Tag1
#Tag2

# Title

## Official Documentation

Website:
Documentation:
GitHub:

## Overview

## Principle

## Architecture

## Installation

## Configuration

## Common Commands

## Troubleshooting

## Best Practices

## References
```

---

# 4 Add Internal Links

Notes should reference each other.

Syntax:

```
[[Note Name]]
```

Example:

```
Longhorn uses [[PV]] [[PVC]] and [[StorageClass]].
```

---

# 5 Prioritize Official Documentation

Each note should first include:

```
Website
Documentation
GitHub
```

---

# 6 Keep the Overview Short

Keep the overview within **3-5 lines**.

Avoid copying entire official documentation.

---

# 7 Focus on Commands

Ops notes should emphasize:

```
Common Commands
Troubleshooting
```

---

# 8 Use Graphs for Primary Tags Only

Graph colors should reflect:

```
Primary Tag
```

---

# 9 Avoid Frequent Changes to the Directory Structure

Once established, the directory structure should not be frequently changed.

---

# 10 Core Principles of the Knowledge Base

```
Structure > Content > Aesthetics
```

Ensure the structure is clear and well-organized.