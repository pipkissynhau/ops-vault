Even with Ingress, it cannot directly locate Pods. This is because the main role of Ingress is to receive external HTTP/HTTPS traffic and route it to different Services based on domain names and path rules. The Service is responsible for providing a stable access entry point for the backend Pods and manages the list of backend instances through selectors and Endpoints. This design ensures business scalability, security, and ease of maintenance. From an operations perspective, using Pods directly as business entry points presents several significant issues:

- Pods may be recreated.
- Pod IP addresses might change.
- The number of replicas could vary.
- The list of backend instances needs to be dynamically managed.

Services exist precisely to address these challenges.

Therefore, Ingress typically relies on Services rather than replacing them.