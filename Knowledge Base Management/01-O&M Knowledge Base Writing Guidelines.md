# Ops Knowledge Base Writing Guidelines

This document outlines the writing standards for the ops-skill-tree knowledge base.

---

# 1 Note Naming

Use **technical names** as filenames.

Correct example:

```
Pod.md
Deployment.md
LVM.md
NFS.md
Calico.md
MySQL.md
```

Avoid:

§

---

# 2 Mandatory Tags

Every note must include:

```
Level 1 Label
Second Label
```

Example:

```
#kubernetes
#storage
#longhorn
```

---

# 3 Use Unified Template

All notes must follow the same structure:

```
#Label1
#Label2

# Title

## Official documents

Network:
Documents:
GitHub:

## Overview

## Rationale

## Structure

## Install

## Configure

## Common commands

## Fault check.

## Best practices

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
Longhorn Use [[PV]] [[PVC]] [[StorageClass]]
```

---

# 5 Prioritize Official Documentation

Every note should prioritize filling:

```
Network of officials
Documentation
GitHub
```

---

# 6 Keep Overview Brief

Overview should be limited to **3-5 lines**.

Avoid lengthy copies from official documentation.

---

# 7 Prioritize Commands

Ops notes should prioritize supplementing:

```
Common commands
Fault check.
```

---

# 8 Graphs Only Use First-Level Tags

Graph colors should only use:

```
Level 1 Label
```

---

# 9 Avoid Frequent Directory Adjustments

Once directory structure is determined, **do not modify frequently**.

---

# 10 Knowledge Base Core Principles

```
Structure > Contents > Beautiful.
```

Prioritize clear structure.