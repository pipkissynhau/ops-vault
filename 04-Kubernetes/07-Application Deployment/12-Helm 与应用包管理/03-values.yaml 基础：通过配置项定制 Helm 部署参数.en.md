`values.yaml` 是 Helm Chart 的参数配置文件。它的作用是为 Chart 模板提供参数，覆盖默认配置，并影响最终渲染出来的 Kubernetes 对象。

### values.yaml 和 ConfigMap 的区别
虽然 `values.yaml` 和 ConfigMap 都用于配置 Kubernetes 对象，但它们的用途和机制有所不同。`values.yaml` 主要用于为 Helm Chart 提供动态参数配置，而 ConfigMap 则用于存储配置数据，并在需要时将其应用到 Kubernetes 对象中。It is the parameter input during the Helm rendering phase.

### What is a ConfigMap?
It is an object actually created in Kubernetes, used to store configuration data.

### The difference between the two
It can be simply understood as follows:

- `values`: Used to control how the template generates objects during deployment.
- `ConfigMap`: The actual configuration object that exists in the cluster after generation.

### An example
The `values` might contain:

    config:
      logLevel: info

Based on this, the template may generate a `ConfigMap` like this:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-app-config
    data:
      LOG_LEVEL: "info"

### Key points for operations and maintenance personnel to understand
`values` is not an object at runtime;
it is a parameter used to “generate objects”.During the handwritten YAML phase, the focus is more on:

- How to define objects
- How objects work together

In the values phase, the focus shifts to:

- What adjustable parameters this template offers
- How these parameters are mapped to the actual deployment outcome
- How the same template can be reused in different environments

In other words, the perspective gradually changes from:

> **Defining objects**

to:

> **Customizing the application delivery results**

This is where Helm truly demonstrates its "application package management capabilities."

---

## References
- Helm official documentation
- Kubernetes official documentation

---

## Next Steps
It is recommended to organize the following content in the next article:

[[04-Helm Deployment Practice Introduction: Installing, Viewing, Upgrading, and Uninstalling an Application]]