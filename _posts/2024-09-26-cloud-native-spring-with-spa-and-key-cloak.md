---
layout: post
title: Cloud Native Spring, Bringing Identity Provider Authentication Flows
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
> This project involves a **React UI**, a **Spring-based Backend for Frontend (BFF)**, and **Keycloak** as an identity broker integrated with **GitHub** as the identity provider (IdP). The goal is to log in via GitHub through Keycloak and display simple text data from a **remote service** on the React UI. The focus is on implementing OAuth2 authentication using Keycloak and understanding identity brokering.
> 
> By walking through the details, we will explore Spring's ready-to-use integrations and address some security and networking challenges encountered on the journey of deploying the application in a local Kubernetes context. So let's get started! 

## 0. Backend for Frontend: a Spring Cloud Gateway project   

The Backend for Frontend (BFF) pattern serves as an essential edge service that interacts with Single Page Applications (SPAs) while handling security concerns. As a gateway, the BFF acts as an intermediary between the SPA and various backend services. It efficiently routes requests from the SPA to the appropriate services.

When it comes to security management, the BFF plays a central role in authentication. It can integrate with identity providers, such as Keycloak, to manage user sessions and validate tokens. 

Spring Cloud Gateway is an ideal choice for implementing a BFF in this scenario, since it provides a lightweight, scalable, and highly customizable API gateway solution, perfectly suited to act as the edge service for SPAs. Along with Spring Security dependency, the gateway service provides a well-managing interface between a user and the whole features of our project.
