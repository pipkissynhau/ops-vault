# DaoCloud Interview Question 04: Design Philosophy of Prometheus Exporters and Certificate Expiration Time Exporters

## Question Description
Interview question: Have you ever written a Prometheus exporter?  
Follow-up question: If you were asked to design an exporter for monitoring certificate expiration times, how would you approach it?

---

## Key Points to Remember
If you have not actually written a complete exporter from scratch, avoid claiming experience with them.

A more cautious way to answer is:

- I have not formally written a complete and generic exporter.
- However, I understand how exporters work.
- I know how to design a lightweight solution for specific scenarios.
- I am aware of when it is appropriate to directly integrate metrics or when an exporter is needed.

Prometheus officially defines an exporter as a tool that converts metrics from third-party systems into a format that Prometheus can capture. It also distinguishes between two main use cases: "directly integrating metrics using the client library" and "writing exporters/custom collectors for systems that cannot be modified." ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com)) ([prometheus.io](https://prometheus.io/docs/instrumenting/writing_exporters/?utm_source=chatgpt.com))

---

## Correct Understanding
Do not simply think of an exporter as "writing an interface to return data."

A more accurate understanding is:

- An exporter acts as a layer for converting metrics.
- It retrieves data from external systems, existing systems, command outputs, API responses, or script results.
- It converts this data into Prometheus-compatible metrics.
- These metrics are then exposed via the `/metrics` endpoint.
- Finally, Prometheus periodically scrapes these metrics.

Prometheus's official documentation clearly states that the purpose of an exporter is to transform third-party metrics made available by existing systems into Prometheus-compatible formats. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com))

---

## How to Answer During an Interview

### Short Answer
I have not written a complete and generic Prometheus exporter from scratch, but I understand its fundamental functionality: an exporter converts data from external systems into Prometheus metrics, which are then accessible via `/metrics`. If I were asked to design an exporter for monitoring certificate expiration times, it would retrieve the `NotAfter` timestamp from the TLS certificate, convert it into a Unix timestamp and the number of seconds remaining before expiration, and also include an indicator indicating whether the retrieval was successful. ([prometheus.io](https://prometheus.io/docs/instrumenting/writing_exporters/?utm_source=chatgpt.com))

### More Detailed Answer
I have not formally written a complete and generic exporter from scratch, but I am familiar with its application scenarios and design principles.  
If the business code can be modified, I would prefer to use the Prometheus client library directly for metric integration. However, if the system itself cannot be altered, or if the data comes from third-party services, existing middleware, external APIs, or script results, an exporter/custom collector would be necessary.  
For a certificate expiration time exporter, it would retrieve the `NotAfter` timestamp from the TLS certificate during each scrape, convert it into Unix time and seconds remaining, and also report whether the retrieval was successful. In a multi-target scenario, Prometheus could pass the target addresses as parameters to the exporter. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## The Essence of an Exporter

### 1. An Exporter is a Metric Conversion Layer
It is not part of the business system itself but acts as a intermediary between:

External systems / Existing systems  
→ Exporter  
→ `/metrics` endpoint  
→ Prometheus scrape process

### 2. Suitable Scenarios for Exporters
Exporters are useful in the following situations:

- When the business code cannot be modified.
- When third-party systems do not support direct metric integration.
- When it is necessary to convert data from command outputs, API responses, or inspection results into metrics.
- When monitoring data needs to be uniformly exposed.

Prometheus recommends using the client library directly if metrics can be integrated into the code; otherwise, an exporter should be used for systems that cannot be modified. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com))

---

## Differences Between Exporters and Direct Metric Integration

### Direct Metric Integration
Suitable for:
- Self-developed business systems where code can be modified.
- When detailed business metrics are required.

Approach:
- Integrate the Prometheus client library into the business code to define and report metrics directly.

### Exporters
Suitable for:
- Existing systems whose code cannot be altered.
- Third-party middleware,运维 scripts, external APIs, or data sources that require conversion.

Approach:
- Retrieve data from the target system, convert it into metrics, and expose them via `/metrics### Prometheuses Official Multi-Target Detection Model Also Emphasizes the Semantic Meaning of Successful/Failed Detection. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Alarm Approach
The real value of a certificate exporter lies in providing early alerts.

Multiple threshold levels can be designed based on `ssl_cert_expires_in_seconds`:

- Less than 30 days: Warning
- Less than 7 days: High-priority alarm
- Less than 3 days: Urgent alarm

Additionally, include this category:
- `ssl_cert_probe_success == 0`: Failed detection alarm

This helps avoid missing issues due to "complete access failure to the target."

---

## Safer Engineering Practices
If you only need to monitor website or service certificate expiration, there's no need to create a custom exporter in many cases.

Prometheus uses the blackbox exporter as a typical example in its multi-target exporter documentation, noting that it can output certificate expiration times. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

Therefore, a more prudent approach is:

- For simple scenarios, prefer the existing blackbox exporter.
- Only consider custom exporters when existing solutions cannot meet your needs.

---

## Key Points

### 1. An Exporter Is Not About Direct Code Modification
It's mainly used for objects where code changes are not feasible.

### 2. The Core of an Exporter Lies in Metric Semantic Design
Consider at least:
- Which metrics to output
- What type of metrics they are
- How to represent failures
- How to trigger alerts

### 3. In Certificate Scenarios, the Key Isn’t Simply Reading the Certificate
It’s about converting it into a format that Prometheus can understand:
- Absolute expiration time
- Remaining time
- Detection success status

### 4. The Multi-Target Exporter Is Often Asked About in Interviews
If there are many targets, avoid creating separate exporters for each one. Let Prometheus pass the target parameters directly. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Common Mistakes

### Mistake 1: Claiming to Have Written an Exporter
If you haven’t officially done so, don’t claim it. It’s better to explain the principles, design, and use cases.

### Mistake 2: Confusing Exporters with Direct Code Insertion
Their applications are different.

### Mistake 3: Only Mentioning “Reading and Outputting Certificates”
That’s too vague. Clearly define:
- Metric design
- Failure handling
- Alert mechanisms

### Mistake 4: Forgetting About Status Metrics Like Probe_success
Just providing the remaining days is incomplete. Include how failures are indicated.

### Mistake 5: Ignoring Existing Solutions
Creating something from scratch may not be necessary. The blackbox exporter is a commonly used and mature solution. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Interview Oral Templates
I haven’t written a complete, general-purpose Prometheus exporter from scratch, but I understand how it works.  
An exporter essentially converts data from external or existing systems into Prometheus metrics and exposes them through `/metrics` for Prometheus to collect.  
If the business code can be modified, I would prefer using client libraries for direct integration; otherwise, I would use exporters/custom collectors to convert API responses, command outputs, or script results into metrics.  
For a certificate expiration time exporter, it would connect to the target service during each scrape, read the `NotAfter` time from the TLS certificate, and expose it as Prometheus metrics, such as expiration timestamp, remaining seconds, and detection success status.  
In multi-target scenarios, I would follow the multi-target exporter approach, passing target addresses as parameters to the exporter in Prometheus requests.  
For regular website certificate monitoring, I would prioritize using existing solutions like the blackbox exporter. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com))

---

## Memory Aid
Remember this main concept:

**Exporter = Metric Conversion Layer**

Then distinguish between two types:

- **Code can be modified → Direct integration**
- **Code cannot be modified → Use an exporter**

For certificate scenarios, remember:

1. Connect to the target.
2. Read the TLS certificate.
3. Extract the `NotAfter` time.
4. Convert it into a timestamp or remaining seconds.
5. Expose it through `/metrics`.
6. Trigger alerts based on thresholds.

---

## Tags
#Prometheus #Exporter #BlackboxExporter #TLS #CertificateMonitoring #Observability #OpsInterview #CloudNativeInterview #MonitoringDesign #Troubleshooting