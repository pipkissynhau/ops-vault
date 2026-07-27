---
tags: [cicd, devops, kubernetes, release process, common interview questions]
---

# Interview Question 31: Detailed Explanation of the CICD Process (From an Operations Perspective)

## 🧭 I. Core Concepts

CI/CD = Continuous Integration + Continuous Deployment

---

## 🔁 II. Complete Process

Code submission → Building → Testing → Image creation → Deployment → Verification

---

## 🧱 III. CI Phase

- Pulling code
- Building
- Testing

Output: Docker image

---

## 🚀 IV. CD Phase

- Pulling the image
- Deploying to K8s
- Rolling updates

---

## 🐳 V. K8s Process

1. Git commit  
2. CI building  
3. Pushing the image  
4. Deploying a Deployment  
5. Rolling updates  

---

## 🔄 VI. Release Strategies

- Rolling updates
- Blue-green deployment
- Canary release

---

## 🔙 VII. Rollback

kubectl rollout undo deployment xxx

---

## ⚠️ VIII. Common Issues

- Image errors  
- Configuration issues  
- Insufficient resources  

---

## 🎯 IX. Interview Summary

- CI: Ensures quality  
- CD: Automates deployment  
- K8s: Manages Deployments  