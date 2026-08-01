---
tags: "[cicd, devops, kubernetes, Release Process, Interview High Frequency]"
---

# Interview Question 31: CICD Process Details (Operations Perspective)

## 🧭 One: Core Concepts

CI/CD = Continuous Integration + Continuous Deployment

---

## 🔁 Two: Full Process

Code Commit → Build → Test → Image → Deploy → Validate

---

## 🧱 Three: CI Stage

- Pull Code
- Build
- Test

Output: Docker Image

---

## 🚀 Four: CD Stage

- Pull Image
- Deploy to K8s
- Rolling Update

---

## 🐳 Five: K8s Process

1. Git Commit  
2. CI Build  
3. Push Image  
4. Deploy Deployment  
5. Rolling Update  

---

## 🔄 Six: Release Strategy

- Rolling Update
- Blue-Green Deployment
- Canary Release

---

## 🔙 Seven: Rollback

kubectl rollout undo deployment xxx

---

## ⚠️ Eight: Common Issues

- Image Error  
- Configuration Error  
- Insufficient Resources  

---

## 🎯 Nine: Interview Summary

- CI: Ensure Quality  
- CD: Automated Deployment  
- K8s: Deployment Management