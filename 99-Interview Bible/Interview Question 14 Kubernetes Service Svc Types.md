---
tags: "[Kubernetes, Service, Svc, Type, Interview]"
---

# Interview Question 14: How Many Types of Services (Svc) Does Kubernetes Have

## Explanation
Kubernetes Service is used to expose Pod as a network service, with **4 types**:

1. **ClusterIP** (default type)  
   - Accessible within the cluster, cannot be directly accessed from outside  
   - Typical Scenario: Communication between microservices  

2. **NodePort**  
   - Opens port on each Node, accessible via `<NodeIP>:<NodePort>`  
   - Typical Scenario: Simple external access or testing  

3. **LoadBalancer**  
   - Cloud platform external load balancer, automatically assigns public IP  
   - Typical Scenario: Production environment external service  
   -Bottom will simultaneously create NodePort + ClusterIP  

4. **ExternalName**  
   - Maps service to external DNS, does not create ClusterIP  
   - Typical Scenario: Cluster internal access to external services  

## Configuration Examples

### ClusterIP
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-clusterip-service
spec:
  type: ClusterIP  # Default
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # 30000-32767
```

### LoadBalancer
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-lb-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### ExternalName
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-external-service
spec:
  type: ExternalName
  externalName: external.example.com
```

## Key Points Summary

- **ClusterIP**: Cluster internal access, default type  
- **NodePort**: Exposes service to cluster external via NodeIP  
- **LoadBalancer**: Cloud platform external load balancer, production environment external service  
- **ExternalName**: Maps to external DNS, enables cluster access to external resources  

## Interview Answer Example

> "Kubernetes Service has four main types: ClusterIP is used for internal cluster communication and is the default type; NodePort opens ports on each Node for external access, suitable for testing; LoadBalancer creates an external load balancer on the cloud platform for production environment external services; ExternalName maps the service to an external DNS, facilitating cluster access to external resources. By selecting different types, we can flexibly manage access requirements inside and outside the cluster."