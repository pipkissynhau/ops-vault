# Obsidian Database Usage Guide

This document explains how to build "database views" in an Obsidian knowledge base to organize technical notes, Runbooks, and incident case content.

---

# I. Obsidian's Database Concept

Obsidian itself is not a traditional database system.

Databases are typically implemented through:

1. Markdown tables
    
2. Tags (Tag)
    
3. Metadata (Frontmatter)
    
4. Dataview Plugin
    

Recommended approach:

```
Dataview Plugin
```

---

# II. Installing Dataview Plugin

Steps:

1. Open Settings
    
2. Community plugins
    
3. Browse
    
4. Search:
    

```
Dataview
```

5. Install
    
6. Enable
    

After installation, you can create database views.

---

# III. Adding Metadata to Notes

Add Frontmatter at the top of notes:

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
|type|Technical type|
|category|Classification|
|status|Learning status|

---

# IV. Creating a Database View

Create a new file:

```
00-Transport architecture/Technical database.md
```

Write:

````markdown
```dataview
table type, category, status
from ""
```
````

Dataview will automatically scan all notes and generate a table.

---

# V. Generating a Database by Tag

For example, viewing only Kubernetes technology:

````markdown
```dataview
table file.link as Technology
from #kubernetes
```
````

Sample output:

|Technology|
|---|
|Pod|
|Deployment|
|Service|
|Ingress|

---

# VI. Viewing a Database for a Specific Directory

For example, the Kubernetes directory:

````markdown
```dataview
table file.link
from "04-Kubernetes"
```
````

---

# VII. Runbook Database

Create in the Runbook directory:

```
20-Runbook/runbookDatabase.md
```

Write:

````markdown
```dataview
table file.link as Runbook
from "20-Runbook"
```
````

---

# VIII. Incident Case Database

Create in the incident case directory:

```
19-Cases of failure/incidentDatabase.md
```

Write:

````markdown
```dataview
table file.link as Incident
from "19-Cases of failure"
```
````

---

# IX. Common Dataview Queries

Viewing Kubernetes technology:

````markdown
```dataview
table file.link
from #kubernetes
```
````

Viewing storage technology:

````markdown
```dataview
table file.link
from #storage
```
````

Viewing monitoring components:

````markdown
```dataview
table file.link
from #monitor
```
````

---

# X. Database Best Practices

Recommended to create the following database pages:

```
00-Transport architecture/Technical maps.md
04-Kubernetes/k8sDatabase.md
02-Linux/linuxDatabase.md
19-Cases of failure/incidentDatabase.md
20-Runbook/runbookDatabase.md
```

This allows quick overview of the entire knowledge system.

---

# XI. Purpose of Database Pages

Database pages can be used for:

```
Technical index
Technical Learning List
RunbookList
List of cases of failure
```

As notes grow, the database will automatically update.