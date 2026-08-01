# 03-values.yaml Basics: Customizing Helm Deployment Parameters via Configuration Items

## Documentation Notes
- Documentation Focus: Helm Parameterized Deployment Fundamentals
- Applicable Stage: After understanding Helm basics and common commands, entering the understanding of values.yaml's practical role
- Recommended Path: `04-Kubernetes/07-Apply deployment/11-Helm and application package management/03-values.yaml Foundation: Customise by configuration item Helm Deployment parameters`

## Tags
#Kubernetes #Helm #values.yaml #Chart #Release #ParametricDeployment #ApplyPackageManagement #Clouds. #Transport

---

## One: Why values.yaml is Important in Helm

When learning Helm, if you only focus on:

- `helm repo add`
- `helm search repo`
- `helm install`
- `helm upgrade`
- `helm rollback`

You can typically only achieve "installing a Chart".

But Helm's true value isn't just installation - it's:

> **Using a single template to generate different deployment results across environments.**

The core of this lies in:

- `values.yaml`
- `--set`
- Chart Default values
- Custom values overriding

These parameter mechanisms.

### Why Can't You Just Use 'install'?
Because real-world application deployments almost always have parameter differences, such as:

- Development environment has fewer replicas
- Production environment has higher resource limits
- Some environments need persistence enabled
- Some environments require different Service types
- Different image tags
- Different domains
- Different log, probe, resource, and authentication parameters

Without the values layer, you'd have to:

- Manually edit templates
- Duplicate YAML files
- Maintain multiple object definitions for different environments

This is exactly what Helm aims to avoid.

---

## Two: What is values.yaml Exactly

You can first understand it directly as:

> **values.yaml is the parameter configuration file for a Helm Chart.**

Its purpose is:

- Provide parameters to Chart templates
- Override default configurations
- Influence the final rendered Kubernetes objects

### A Basic Understanding
Charts typically have templates, such as:

- Deployment template
- StatefulSet template
- Service template
- ConfigMap template

These templates aren't hard-coded, but instead reference variables.

For example, a template might contain:

    replicas: {{ .Values.replicaCount }}

This indicates:
- The final replica count isn't fixed
- It comes from values in `replicaCount`

### What Does This Mean?
The essence of values.yaml isn't "another YAML object", but rather:

> **The parameter input that controls the result of Helm template rendering.**

---

## Three: What's the Relationship Between Chart Default Values and Custom Values

Helm typically has two layers of values to distinguish:

### 1. Chart's Built-in Default Values
Most Charts internally have a default `values.yaml`.

This file defines the default parameters for the Chart, such as:
- Default image
- Default replica count
- Default resource configuration
- Default Service type
- Default persistence toggle

### 2. User Custom Values
During actual installation, you can provide your own values file to override defaults.

For example:

    helm install my-nginx bitnami/nginx -f values.yaml

Here, `values.yaml` is the user's custom parameters.

### A Basic Understanding
You can initially think of it as:

- Chart Default values = Default configuration
- User values = Override configuration for actual deployment

---

## Four: What Core Problems Does values.yaml Solve

The value of values.yaml isn't in "having many parameters", but in giving Helm true parameterized deployment capabilities.

It mainly solves the following categories of problems.

### 1. Environmental Differences
Development, testing, and production environments can reuse the same Chart with different values.

### 2. Component Customization
For example:
- Replica count
- Image tag
- Port
- Service type
- Persistence
- Resource limits

None of these need direct template modifications.

### 3. Version and Change Management
Values files can be version-controlled, making it easier to track deployment parameter changes.

### 4. Reusability
A single Chart can be configured with different values for different teams, environments, and business instances.

### A Basic Conclusion
The core value of values.yaml can be summarized as:

> **Templates remain stable, parameters change according to scenarios.**

---

## Five: What Common Content Do Values Configurations Involve

Although different Charts vary greatly, the most common content in values typically focuses on the following directions.

### 1. Replica Count
For example:

    replicaCount: 2

### 2. Image
For example:

    image:
      repository: nginx
      tag: "1.27.0"
      pullPolicy: IfNotPresent

### 3. Service
For example:

    service:
      type: ClusterIP
      port: 80

### 4. Resource Limits
For example:

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

### 5. Persistence
For example:

    persistence:
      enabled: true
      size: 10Gi

### 6. Ingress
For example:

    ingress:
      enabled: true
      hostname: demo.example.com

### 7. Authentication Information or Additional Configuration
Some Charts also provide through values:
- Username
- Password
- Existing Secret name
- Custom configuration snippets

### Operations Understanding Focus
At this stage, you don't need to memorize all fields of each Chart, but should first form a sense:

> **Values mainly control key parameters in the deployment result.**

---

## Six: Why is values.yaml Suitable for Environment Differentiation

This is one of the most common practical uses of values.

### A Typical Scenario
The same application usually differs between development and production environments.

For example: /think

| Configuration Item | Development Environment | Production Environment |
|---|---|---|
| Replica Count | 1 | 3 |
| Resource Limits | Low | High |
| Service Type | ClusterIP | LoadBalancer / Ingress |
| Persistence | Optional | Usually Enabled |
| Log Level | debug | info / warn |

### What Happens Without Values
Normally you can only:
- Copy a template and modify a set of parameters
- Copy another template and modify another set of parameters

Over time, it's easy to encounter:
- Template Forking
- Parameter Drift
- Maintenance Difficulty

### Using Values
You can keep the same Chart and differentiate environments via values, for example:

- `values-dev.yaml`
- `values-test.yaml`
- `values-prod.yaml`

### Operations Understanding Focus
One very important use of values is:

> **The same template adapts to different environments.**

---

## VII. A Minimalized Values Example

Here's a teaching example:

    replicaCount: 2

    image:
      repository: nginx
      tag: "1.27.0"
      pullPolicy: IfNotPresent

    service:
      type: ClusterIP
      port: 80

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

### What Does This Values File Express

#### `replicaCount`
Controls replica count.

#### `image.repository`
Controls image repository or image name.

#### `image.tag`
Controls image version.

#### `service.type`
Controls Service type.

#### `service.port`
Controls Service exposed port.

#### `resources`
Controls container resource requests and limits.

### Operations Understanding Focus
Values itself is not a Kubernetes workload object, but rather:

> **A parameter set that influences the generation result of workload objects.**

---

## VIII. How Templates Read Values

Helm templates read parameters through the `Values` object.

### Common Examples

Assume the template writes:

    spec:
      replicas: {{ .Values.replicaCount }}

This indicates:
- Replica count comes from values' `replicaCount`

If the template writes:

    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

This indicates:
- Image address comes from `image.repository`
- Image version comes from `image.tag`

If the template writes:

    type: {{ .Values.service.type }}

This indicates:
- Service type comes from `service.type`

### A Basic Understanding
You can initially understand Helm templates as:

- Templates write variable placeholders
- Values provide variable values
- Rendered to produce the final YAML

---

## IX. Why values.yaml Is Not Equal to ConfigMap

This point is often confusing, especially after learning ConfigMap.

### What Is values.yaml
It is a parameter input during Helm rendering.

### What Is ConfigMap
It is an actual object created in Kubernetes, used to store configuration data.

### Differences Between the Two
You can simply understand as:

- values: Parameters used to control how templates generate objects during deployment
- ConfigMap: A configuration object that exists in the cluster after generation

### An Example
values might have:

    config:
      logLevel: info

The template might generate a ConfigMap based on it:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-app-config
    data:
      LOG_LEVEL: "info"

### Operations Understanding Focus
Values are not runtime objects,  
They are parameters used to "generate objects."

---

## X. How to Use Custom Values Files During Installation

The most common way is through `-f` to specify the file.

### Common Syntax

    helm install my-nginx bitnami/nginx -f values.yaml

This indicates:
- Install `my-nginx`
- Use `bitnami/nginx` Chart
- And override default parameters with `values.yaml`

### Common Syntax for Specifying Namespace

    helm install my-nginx bitnami/nginx -n web --create-namespace -f values.yaml

### If It's an Upgrade
You can also write it like this:

    helm upgrade my-nginx bitnami/nginx -n web -f values.yaml

### Operations Understanding Focus
The significance of `-f values.yaml` can be understood as:

> **Custom parameters to customize this installation or upgrade.**

---

## XI. How to Use Multiple Values Files for Overriding Parameters

Helm supports passing multiple values files simultaneously.

### Common Syntax

    helm install my-nginx bitnami/nginx -f values-common.yaml -f values-prod.yaml

### Overriding Rules
Usually, later files override earlier ones with the same field names.

This means you can commonly split them as:

- `values-common.yaml`: Place common parameters
- `values-dev.yaml`: Place development environment differences
- `values-prod.yaml`: Place production environment differences

### A Basic Understanding
This is equivalent to:

- Load common configuration first
- Then load environment-specific configuration
- The latter overrides the former

### Operations Understanding Focus
The value of multiple values files lies in:

> **Separating common parameters and environment-specific parameters for layered management.**

---

## XII. What Is the Relationship Between `--set` and values.yaml

Besides `-f values.yaml`, Helm also supports using `--set` to pass parameters directly via command line.

### Common Syntax

    helm install my-nginx bitnami/nginx --set replicaCount=2

Or: /think

helm install my-nginx bitnami/nginx --set service.type=NodePort

### What does this mean
This indicates overriding a value directly via command line during installation.

### What scenarios is it suitable for
More suitable for:
- Temporary testing
- Small-scale parameter overrides
- Quick validation of field changes

### What scenarios is it not suitable for
Not suitable for:
- Managing large numbers of parameters
- Long-term maintenance across multiple environments
- Team collaboration configuration consolidation

### A basic conclusion
You can remember it as:

- `--set`: Suitable for temporary minor changes
- `values.yaml`: Suitable for formal maintenance

---

## ThirteenI don't know.Why formal environments are better suited for values files rather than numerous `--set`

### Reason 1: Better readability
Values files are more suitable for overall viewing and review.

### Reason 2: More suitable for version management
Values files can be placed in Git, making it easier to track parameter changes.

### Reason 3: More suitable for team collaboration
Team members find it easier to understand a structured values file rather than a long command string.

### Reason 4: More suitable for environment differentiation
Different environments can maintain separate values files.

### A basic judgment
If parameters start to increase, it's typically not advisable to continue using many `--set`, but rather switch to values file management.

---

## FourteenI don't know.How to view which values a Chart supports by default

This step is very important because when actually using it, you often don't guess parameter names but first check the default values of the Chart.

### Common commands

    helm show values bitnami/nginx

### What this command does
It outputs the default values content of this Chart.

### Why this is valuable
It directly tells you:

- Which parameters this Chart supports
- What the default values are
- How the parameter hierarchy is organized
- Where changes can be made

### A basic understanding
If the values file is "the user's configuration entry point",  
then `helm show values` is:

> **View what parameters this Chart exposes for adjustment.**

---

## FifteenI don't know.How to view the actual values used by the current Release

This command was already learned in the previous section, but its importance is being emphasized again here.

### Common commands

    helm get values my-nginx -n web

If you want to see a more complete result:

    helm get values my-nginx -n web -a

### What this command signifies
It doesn't show the Chart's default values, but rather:

- What parameters this Release actually used

### Why this step is important
Because when troubleshooting or reviewing, you typically care more about:

- What parameters this instance was deployed with
- Whether it matches the expected values
- Whether parameters were overwritten after an upgrade

---

## SixteenI don't know.A complete values usage example

Below is a basic values file and installation command example.

### File: `values-prod.yaml`

    replicaCount: 3

    image:
      repository: nginx
      tag: "1.27.0"
      pullPolicy: IfNotPresent

    service:
      type: ClusterIP
      port: 80

    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi

### Installation command

    helm install my-nginx bitnami/nginx -n web --create-namespace -f values-prod.yaml

### Upgrade command

    helm upgrade my-nginx bitnami/nginx -n web -f values-prod.yaml

### What does this show
The same Chart:
- Can be controlled by different values to achieve different deployment results
- The same values file can be reused for both installation and upgrades

---

## SeventeenI don't know.Common misconceptions about values.yaml

### 1. Treating values as Kubernetes objects
It's not `Deployment`, `Service`, or similar objects; it's a template parameter input.

### 2. Assuming values names are universal standards
Field names and structures can vary greatly between different Charts.  
You cannot assume all Charts have the same fields.

### 3. Guessing field names without first checking default values
This often leads to incorrect path or field name errors.

### 4. Using `--set` for all parameters
It's acceptable in the short term, but not ideal for long-term maintenance.

### 5. Believing changes to values automatically take effect on the cluster
Values files are just a source of parameters; actual effect requires coordination with:

- `helm install`
- `helm upgrade`

### Operations understanding focus
Values files themselves do not automatically change the cluster,  
They need to be rendered and delivered through Helm commands.

---

## EighteenI don't know.The most important cognitive takeaways from this section

### 1. values.yaml is the parameter configuration entry point for Helm
This is the first understanding.

### 2. The essence of values is controlling the result of template rendering
This is the second understanding.

### 3. The same Chart can be adapted to different environments through different values
This is the third understanding.

### 4. `--set` is more suitable for temporary overrides, values files are better for formal maintenance
This is the fourth understanding.

### 5. Before using values, you should first check the default values of the Chart
This is the fifth understanding.

---

## NineteenI don't know.Phase summary

The fundamental value of Helm is not just installing a Chart, but enabling parameterized deployment through values.

Through this section, you should at least establish the following basic understandings:

- values.yaml is a parameter file
- Chart templates read parameters via `.Values`
- Different environments can use different values files
- `helm show values` is used to view default parameters
- `helm get values` is used to view actual parameters of the current instance
- Values files are more suitable for long-term maintenance in formal environments

As long as this layer of understanding is clear, proceeding to:

- Helm installation, viewing, upgrading, and uninstalling practices
- Chart result verification
- Helm integration with application release, updates, and rollback

Will be smoother.

---

## TwentyI don't know.Keyword quick reference /think

- values.yaml: Helm Parameters Configuration File
- `.Values`: Entry point for reading parameters in the template
- `-f values.yaml`: Specify custom values file
- `--set`: Temporarily override parameters via command line
- `helm show values`: View Chart's default values
- `helm get values`: View actual values used by the Release
- Parameterized deployment: Same template adapts to different environments

---

## 21. Operational Deep Dive Understanding

After learning about values in Helm, the approach to application delivery undergoes a significant shift.

In the hand-written YAML stage, the focus was on:

- How to define objects
- How objects interact with each other

At the values stage, the focus shifts to:

- What parameters are available in this template
- How these parameters map to actual deployment outcomes
- How to reuse the same template across different environments

In other words, the perspective transitions from:

> **Defining objects**

to:

> **Customizing application delivery results**

This is where Helm truly begins to demonstrate its "application package management" capabilities.

---

## References
- Helm Official Documentation
- Kubernetes Official Documentation

---

## Next Day Recommendation
Next post suggestion: 

[[04-Helm Deployment Practice - Installation Viewing Upgrade and Uninstallation of an Application]]