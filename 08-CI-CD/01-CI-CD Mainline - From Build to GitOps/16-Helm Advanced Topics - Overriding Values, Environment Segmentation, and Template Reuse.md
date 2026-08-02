# 16-Helm Advanced Topics: Overriding Values, Environment Segmentation, and Template Reuse

## Document Description

This is the 16th note in the 08-CI-CD learning pathway.

In the previous 8th note, we completed the basic introduction to Helm:

- What are Charts?
- What are templates?
- What are values?
- How to perform helm install / upgrade / rollback?

In the 13th note, we started exploring multi-environment deployment concepts:

- dev
- test
- prod
- How to handle differences in namespace, image tag, and replicas.

In this note, we will connect these two aspects and focus on solving the following problem:

**When Helm is actually used for multi-environment configuration, how should values be segmented, overridden, and reused?**

This note will not stop at just ensuring that a single `values.yaml` file can work properly. Instead, it will delve into practical applications you will frequently use later on:

- Where to place default values in values.
- How to segment values for dev, test, and prod environments.
- What should be included in values files and what should remain in templates.
- How to reuse the same Chart for multiple environments.
- Why overly complicated segmentation of values can make maintenance more difficult.

This document will continue to use the current experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- Existing basic Helm Chart: `manual-web`

## Tags

#Kubernetes #CI-CD #Helm #Values #Multi-environment Deployment #Template Reuse #Chart #Helm Advanced #Practical Notes

## Learning Objectives

After completing this note, you should be able to:

1. Understand why Helm values need to be structured in layers and overridden.
2. Know where default values and environment-specific values should be placed.
3. Learn how to segment multi-environment values files properly.
4. Distinguish between template structure and value parameters.
5. Create dev and test sets of values within the current Helm Chart.
6. Perform an install or upgrade operation with value overrides.
7. Clearly explain the basic concept of Helm template reuse.

## Main Experimental Approach for This Note

This note is divided into 4 sections:

1. First, organize the default values and environmental differences of the current Chart.
2. Extract `values-dev.yaml` and `values-test.yaml`.
3. Render and deploy them using the `-f` option for value overrides.
4. Summarize the minimum reuse rules for Helm in multi-environment deployment.

---

## Section 1: Review the Current State of Your Helm Chart

Enter the Helm Chart directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

View the current files:

    ls
    ls templates
    cat values.yaml

If you have already followed the 8th and 13th notes, you should have seen these files at least once:

- `Chart.yaml`
- `values.yaml`
- `templates/deployment.yaml`
- `templates/service.yaml`
- Possibly also `values-dev.yaml` and `values-test.yaml`.

### Understanding Required for This Step

At this point, your Helm Chart is no longer just capable of running; it is now handling multi-environment differences.

Therefore, the focus of this section is not to create a brand-new Chart but rather:

**To establish a clearer structure for values within the existing Chart.**

---- The environment files only cover the differences.What is the actual rendering result of a `helm template`?

### Question 2: Why are dev and test versions of the same Chart different?

Because the Chart reuses the same template, but the final outcome is determined by different `values`.

### Question 3: Should all fields be turned into `values`?

No.  
Only fields that “change frequently or need to be managed based on environments” should be extracted as `values`.

---

## Key Points Learned

1. The concept of hierarchical and overlapping `values`.
2. The distinction between default `values` and environment-specific `values`.
3. How Helm enables template reuse across multiple environments.
4. Which environmental parameters are most suitable for being separated out.
5. How to use `-f values-xxx.yaml` for installation or upgrade.

## In One Sentence

The essence of Helm’s multi-environment management lies not in creating duplicate configurations for each environment, but in allowing the same Chart to share a common template structure. Environmental differences are then managed through default and custom `values`.