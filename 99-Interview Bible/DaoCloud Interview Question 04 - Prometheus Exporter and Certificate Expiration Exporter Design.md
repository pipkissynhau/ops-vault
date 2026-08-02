# DaoCloud Interview Question 04: Designing a Prometheus Exporter and a Certificate Expiry Time Exporter

## Question Description
Interview question: Have you ever written a Prometheus exporter?  
Follow-up: If you were to write an exporter to monitor certificate expiry times, how would you design it?

---

## Core Conclusion
If you haven't truly written a complete exporter from scratch, don't claim experience.

A more stable answer approach is:

- Haven't officially written a full generic exporter
- But understand how exporters work
- Know how to design a lightweight scenario solution
- Know when to directly instrument and when to write an exporter

Prometheus officially defines an exporter as: converting third-party system metrics into a format that Prometheus can scrape. It also specifically distinguishes between "directly using client library instrumentation" and "writing an exporter/custom collector for systems that cannot be modified" scenarios. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com)) ([prometheus.io](https://prometheus.io/docs/instrumenting/writing_exporters/?utm_source=chatgpt.com))

---

## Correct Understanding
Don't simply understand an exporter as "writing an interface to return data".

A more accurate understanding is:

- An exporter is a metric conversion layer
- It retrieves data from external systems, existing systems, command outputs, interface returns, and script results
- Converts these data into Prometheus metrics
- Exposes them via HTTP `/metrics`
- Finally, Prometheus scrapes them periodically

Prometheus official documentation clearly states that an exporter's responsibility is to convert third-party metrics exposed by existing systems into Prometheus metrics. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com))

---

## How to Answer in an Interview

### One-Sentence Version
I haven't written a complete generic Prometheus exporter from scratch, but I understand its working principle: an exporter essentially converts data from external systems into Prometheus metrics and exposes them via `/metrics`; if I were to design a certificate expiry time exporter, I would have it connect to the target service to read the TLS certificate's `NotAfter` expiry time, then expose metrics like expiry timestamp, remaining seconds, and probe success status. ([prometheus.io](https://prometheus.io/docs/instrumenting/writing_exporters/?utm_source=chatgpt.com))

### More Complete Version
I haven't officially written a full generic exporter from scratch, but I understand its use cases and design principles.  
If business code can be modified, I would prioritize using Prometheus client library for direct instrumentation; if the system itself cannot be modified, or if the data sources are third-party services, existing middleware, external interfaces, or script results, I would use an exporter/custom collector to convert the data into Prometheus metrics.  
If I were to design a certificate expiry time exporter, I would have it connect to the target address during each scrape, read the TLS certificate's `NotAfter` expiry time, convert it into Unix timestamp and remaining seconds, and output a probe success status metric.  
For multi-target scenarios, I would implement it in a multi-target exporter pattern, allowing Prometheus to pass the target address as a parameter to the exporter during requests. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Essence of an Exporter

### 1. Exporter is a Metric Conversion Layer
It is not the business system itself, but located between:

External System / Existing System  
→ Exporter  
→ `/metrics`  
→ Prometheus scrape

### 2. When is an exporter suitable?
Suitable for the following types of scenarios:

- Business code cannot be modified
- Third-party systems cannot be directly instrumented
- Need to convert command outputs, interface returns, and inspection results into metrics
- Need to uniformly expose monitoring data

Prometheus official documentation states that if direct instrumentation in code is possible, it's typically preferred to use the client library; if the target system cannot be directly instrumented, writing an exporter is suitable. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com))

---

## Difference Between Direct Instrumentation and Exporters

### Direct Instrumentation
Suitable for:
- Self-developed business systems
- Can modify code
- Need more granular business metrics

Approach:
- Introduce Prometheus client library
- Define and report metrics in business code

### Exporter
Suitable for:
- Cannot modify existing systems
- Third-party middleware
- Operations scripts, external interfaces, certificate checks, backup tasks, etc.

Approach:
- Retrieve data from the target system
- Convert it into metrics
- Expose `/metrics`

Prometheus official documentation clearly distinguishes these two approaches. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com))

---

## Designing a Certificate Expiry Time Exporter

### Design Goals
Monitor certificate expiry times for one or more HTTPS/TLS services to enable early alerts.

### Two Common Patterns

#### 1. Fixed Target Exporter
Suitable for:
- Monitoring a fixed number of domains or addresses
- Few targets
- Targets can be written into a configuration file

Operation:
- Exporter reads configuration itself
- Connects to these fixed targets during each scrape
- Outputs metrics for each target

#### 2. Multi-Target Exporter
Suitable for:
- Many targets
- Need to uniformly probe different addresses
- Prometheus specifies targets via parameters

Prometheus officially provides documentation for the multi-target exporter pattern.  
In this pattern, Prometheus can pass the target address as a URL parameter when scraping the exporter, and the exporter then probes the target and returns results. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Core Metric Design
A certificate exporter should at least expose the following 3 types of metrics:

### 1. Absolute Certificate Expiry Time
Example:
- `ssl_cert_not_after_timestamp_seconds`

Meaning:
- Unix timestamp converted from the certificate's `NotAfter`

### 2. Seconds Until Expiry
Example:
- `ssl_cert_expires_in_seconds`

Meaning:
- Difference between current time and certificate expiry time

### 3. Probe Success Status
Example:
- `ssl_cert_probe_success`

Meaning:
- Returns 1 for success
- Returns 0 for failure

This design offers:
- Ability to view absolute expiry time
- Convenience for alerting based on remaining days
- Differentiation between "certificate is about to expire" and "probe failed entirely" issues

Prometheus exporter design recommendations also emphasize clear metric naming, labels, types, and observability semantics. ([prometheus.io](https://prometheus.io/docs/instrumenting/writing_exporters/?utm_source=chatgpt.com))

### 1. Prometheus Scrape exporter
Prometheus periodically accesses the exporter's `/metrics`, or in multi-target mode accesses similar `/probe?target=...` interfaces. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

### 2. exporter Connects to Target Service
exporter establishes a TCP/TLS connection with the target address.

### 3. Read Certificate Information
Read certificate content from the TLS handshake result, focusing on:
- `NotAfter`

### 4. Convert to Metrics
Convert the read certificate expiration time into:
- Absolute timestamp
- Seconds remaining until expiration
- Probe status

### 5. Return to Prometheus
exporter outputs metrics text in Prometheus format, completing the scrape.

---

## Failure Scenario Handling
A qualified exporter cannot only consider success paths, but also failure scenarios:

- Target domain cannot be resolved
- TCP connection failed
- TLS handshake failed
- Certificate reading failed
- Timeout
- Upstream service anomaly

At this time, it is recommended:
- `probe_success = 0`
- Do not output expiration time-related metrics, or output default values as agreed
- Record error logs for troubleshooting

Prometheus's official multi-target probing mode also emphasizes the semantic of probe success/failure status. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Alerting Strategy
The actual value of certificate exporter lies in early alerting.

You can design multi-level thresholds based on `ssl_cert_expires_in_seconds`:

- Less than 30 days: Warning
- Less than 7 days: High-priority alert
- Less than 3 days: Emergency alert

Additionally, add one:
- `ssl_cert_probe_success == 0`: Probe failure alert

This avoids only seeing no data due to "target access failure" and not knowing what happened.

---

## Engineering Practice: Safe Supplement
If you just need to monitor certificate expiration for websites or services, many scenarios don't require writing your own exporter.

Prometheus's official multi-target exporter documentation uses blackbox exporter as a typical example, explicitly mentioning it can output certificate expiration time as probing results. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

Therefore, a more secure engineering approach is:

- Prioritize existing blackbox exporter for simple scenarios
- Only consider custom exporter when existing solutions don't meet requirements

---

## Key Knowledge Points

### 1. exporter is not the business instrumentation itself
It's more used for objects that cannot directly modify code.

### 2. The core of exporter is not the interface, but the metric semantic design
At least clarify:
- Which metrics to output
- Metric types
- How to express failure
- How to alert later

### 3. The most critical thing in certificate scenarios is not reading the certificate
But converting it into a Prometheus-friendly monitoring model:
- Absolute expiration time
- Remaining time
- Probe success status

### 4. multi-target exporter is a high-frequency interview follow-up point
If there are many targets, you should not start a separate exporter for each target, but let Prometheus pass target parameters into the exporter. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Common Mistakes

### Common Mistake 1: Claiming to have written an exporter from scratch
If you haven't officially written one, don't claim it.  
A safer approach is to clearly explain the principles, design ideas, and applicable scenarios.

### Common Mistake 2: Confusing exporter with direct instrumentation
Their applicable scenarios are different.

### Common Mistake 3: Only saying "read certificate and output"
This is too vague.  
You need to clarify:
- Metric design
- Failure handling
- Alerting methods

### Common Mistake 4: Forgetting status metrics like probe_success
Only outputting "remaining days" is insufficient.  
You must consider how to express probe failure.

### Common Mistake 5: Ignoring existing solutions
Engineering-wise, you don't necessarily need to build your own solution.  
blackbox exporter is already a common mature solution. ([prometheus.io](https://prometheus.io/docs/guides/multi-target-exporter/?utm_source=chatgpt.com))

---

## Interview Verbal Template
I haven't written a complete general-purpose Prometheus exporter from scratch, but I understand its working principle.  
exporter essentially converts data from external systems or existing systems into Prometheus metrics, and exposes them via `/metrics` for Prometheus to scrape.  
If business code can be modified, I would prioritize using client libraries for direct instrumentation; if not, I would use exporter/custom collector to convert interface responses, command outputs, or script results into metrics.  
If I were to design a certificate expiration time exporter, I would have it connect to the target service during each scrape, read the `NotAfter` time from the TLS certificate, and expose it as Prometheus metrics like expiration timestamp, remaining seconds, and probe success status.  
For multi-target scenarios, I would follow the multi-target exporter approach, letting Prometheus pass target addresses as parameters to the exporter.  
For regular website certificate monitoring, I would also prioritize existing solutions like blackbox exporter in engineering. ([prometheus.io](https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com))

---

## Memory Template
First remember this main line:

exporter = Metric Conversion Layer

Then distinguish two categories:

Can modify code  
→ Direct instrumentation

Cannot modify code  
→ exporter

For certificate scenarios, remember this additional point:

Connect to target  
→ Read TLS certificate  
→ Extract NotAfter  
→ Convert to timestamp/remaining seconds  
→ Expose `/metrics`  
→ Alert based on thresholds

---

## Tags
#Prometheus
#Exporter
#BlackboxExporter
#TLS
#CertificateMonitor
#Observation
#TransportInterview
#TheYunwonInterview.
§
#FaultCheck.