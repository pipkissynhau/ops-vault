Scanning is crucial because it helps identify potential security vulnerabilities in images, ensuring that only secure versions are deployed in production environments. Because images are not neutral files; they will ultimately be deployed in a cluster.  
If the base image has critical vulnerabilities, or if certain software package versions pose known risks, you should at least be aware of this.

### Key Understanding for This Step

Scanning is not meant to ensure perfect security from the start,  
but rather to help you establish:

> Just because an image has been pushed does not mean it is necessarily “trusted and ready for use.”

---

## Section 8: Finding the Scan Entry on the Harbor Page

It is recommended that you try this out yourself.

### Step 1: Enter the Repository

Go to:

- Project: `test`
- Repository: `manual-web`

### Step 2: Click on a Specific Tag

For example:

- `v18`
- Or `dev-f1e2d3c-1801`

Check if there are options like:

- Scan
- Vulnerabilities
- Scan Results
- Number of Critical / High-Risk / Low-Risk Issues

The UI may vary depending on the Harbor version, but scan-related options should be available.

### Step 3: If the Page Allows Manual Scanning

Try scanning a specific tag.

### Key Understanding for This Step

At this stage, you don’t need to strive for “all scan results being zero.”  
Focus on two things:

1. Finding where to perform scans.
2. Understanding that scan results serve as a reference for decision-making regarding deployments.

---

## Section 9: Interpreting Minimum Scan Results

If the scanning feature is available, you will typically see categories such as:

- Critical
- High
- Medium
- Low

### Current Best Practices

#### 1) Focus on Clearly Identifying High-Risk Issues

It’s not about avoiding learning from any vulnerabilities, but rather ensuring you know:

- Whether the current image poses significant risks.

#### 2) Check the Source of Base Images

For example, if you commonly use:

- `nginx:1.27`
- `busybox:1.35`
- `alpine:3.20`

If scan results are unsatisfactory, don’t immediately assume “Harbor is broken.”  
Instead, consider:

- The version of the base images.
- The software packages included in the image.

### Key Understanding for This Step

The value of scan results lies first in identifying issues and then in determining how to address them.

---

## Section 10: What is a Webhook and How to Understand It Initially

In Harbor, a Webhook can be understood as:

**A mechanism that sends notifications to external systems when certain events occur in a repository.**

For example:

- A new image is successfully pushed.
- An image is deleted.
- Certain project-related events take place.

Harbor then sends these event notifications to a specified URL.

### Why It’s Useful

Later on, you might want to use Webhooks for:

- Notifying a system when a new image is available in Harbor.
- Triggering automated actions.
- Allowing external platforms to stay informed about changes in your repository.

### Key Understanding for This Step

A Webhook is not an image-building or deployment command.  
Rather, it serves as a **connection between Harbor and external automation systems**.

---

## Section 11: Finding the Webhook Entry on the Harbor Page

### Step 1: Access Project-Level Management

Go to the `test` project and check if there are options like:

- Webhooks
- Notifications
- Events

### Step 2: Review Configuration Details

You will usually see:

- Target URL
- Types of events triggered
- Enable/Disable status

### Key Understanding for This Step

You may not have a suitable external URL to receive notifications yet,  
but it’s important to understand that:

**Webhooks are one of the links between Harbor and external automation systems.**

---

## Section 12: How to Understand Webhooks in the Context of Current Mainline Practices

Considering what you’ve learned so far, the most natural way to understand Webhooks is as follows:

### In Current Mainline Processes

- **Image Building:** Create images.
- **Pushing to Harbor:** Upload images to Harbor.
- **Deployment with kubectl/Helm:** Deploy images using relevant commands.

### The Role of Harbor Webhooks

Webhooks complement these processes by:

- Notifying external systems immediately after an image is added to Harbor.
- Ensuring that changes in the repository are promptly communicated outside the system.

So, you can think of Webhooks as a **bridge between Harbor events and external systems**.

For example, they become particularly useful when you integrate more complex platforms later on.

---

## Section 13: Habits Worth Establishing for Current Phase of Harbor Governance

Here are some practical habits to adopt now:

### Habit 1: Clearly Define Tag Semantics

At least be able to distinguish between:

- Learning purposes
- Testing environments
- Production use