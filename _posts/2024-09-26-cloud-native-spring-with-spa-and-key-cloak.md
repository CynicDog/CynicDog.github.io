---
layout: post
title: Bringing Identity Providers' Authentication Flows into a Cloud Native Spring Framework. 
date: 2024-09-26 00:00:00 +09:00
tags: [spring, docker, minikube, jib, keycloak]            
mermaid: true
---

> In this article, we are going to explore the implementation of identity providers' OAuth2 flow in a cloud-native Spring development.
>
> In a cloud-native development environment, there’s a lot to consider, from secure resource/configs management to precise integration of various services. Tools with a strong cloud affinity like Jib and GitHub Actions simplifiy the development process much smoother, helping you easily manage containerization and automate deployments. Keycloak also stands out with its great portability and compatibility, ensuring fluent integration of authentication services into applications regardless of the infrastructure used.
>
> Although there are numerous competent solutions to requirements of cloud-native development nowadays, Spring remains a vital framework even in this landscape.
>
> By walking through the details, we will explore Spring's ready-to-use integrations and address some security and networking challenges encountered on the journey of deploying the application in a local Kubernetes context.
