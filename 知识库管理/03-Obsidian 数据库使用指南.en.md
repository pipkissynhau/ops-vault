# Obsidian Database Usage Guide

This document explains how to create "database views" in the Obsidian knowledge base to organize technical notes, Runbooks, fault cases, and more.

---

# I. Obsidian's Database Concept

Obsidian itself is not a traditional database system.

Databases are typically implemented through the following methods:

1. Markdown tables
2. Tags
3. Metadata (Frontmatter)
4. The Dataview plugin

The most recommended method is to use the **Dataview plugin**.

---

# II. Installing the Dataview Plugin

Steps:

1. Open Settings
2. Choose Community plugins
3. Browse
4. Search for:
```
Dataview
```
5. Install it and enable it.

Once installed, you can start creating database views.

---

# III. Adding Metadata to Notes

Add Frontmatter at the top of your note:

Example:

```markdown
---
type: kubernetes
category: resource
status: learning
---
```

Meaning:

|Field|Description|
|---|---|
|type|Technical category|
|category|Classification|
|status|Learning status|

---

# IV. Creating Database Views

Create a new file:

```
00-Operational Architecture/Technology Database.md
```

Write in it:

```markdown
```dataview
table type, category, status
from ""
```

Dataview will automatically scan all notes and generate a table.

---

# V. Generating Databases by Tag

For example, to view only Kubernetes technology:

```markdown
```dataview
table file.link as Technology
from #kubernetes
```

Example output:

|Technology|
|---|
|Pod|
|Deployment|
|Service|
|Ingress|

---

# VI. Viewing a Database for a Specific Directory

For example, the Kubernetes directory:

```markdown
```dataview
table file.link
from "04-Kubernetes"
```

---

# VII. Runbook Databases

Create in the Runbook directory:

```
20-Runbook/Runbook Database.md
```

Write in it:

```markdown
```dataview
table file.link as Runbook
from "20-Runbook"
```

---

# VIII. Fault Case Databases

Create in the fault case directory:

```
19-Fault Cases/Incident Database.md
```

Write in it:

```markdown
```dataview
table file.link as Incident
from "19-Fault Cases"
```

---

# IX. Common Dataview Queries

View Kubernetes technology:

```markdown
```dataview
table file.link
from #kubernetes
```

View storage technologies:

```markdown
```dataview
table file.link
from #storage
```

View monitoring components:

```markdown
```dataview
table file.link
from #monitor
```

---

# X. Database Best Practices

It is recommended to create the following database pages:

```
00-Operational Architecture/Technology Map.md
04-Kubernetes/k8s Database.md
02-Linux/Linux Database.md
19-Fault Cases/Incident Database.md
20-Runbook/Runbook Database.md
```

This way, you can quickly browse the entire knowledge system.

---

# XI. The Role of Databases

Database pages can be used for:

```markdown
Technical index
Technical learning list
Runbook list
Fault case list
```

As more notes are added, the database will automatically update.