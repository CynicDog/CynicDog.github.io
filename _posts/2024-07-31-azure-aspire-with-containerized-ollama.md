---
layout: post
title: Weaving up Microservices with Azure Aspire (Part 1, local Kubernetes deployment) 
date: 2024-07-31 00:00:00 +09:00
categories: [.NET, Aspire]
tags: [azure, aspire, ollama, microservices, docker]            
mermaid: true
---

> In this post, we will learn how to create a microservices application using Azure Aspire, a code-level orchestrator tool. I built and deployed a microservices application on a local Kubernetes context, which includes:
> 
> 1) a containerized Ollama service with a Python wrapper server,
>
> 2) a .NET-textbook-material weather forecast server with an attached containerized PostgreSQL database, and
>
> 3) a React UI client.
> 
> I hope this article provides insights into the development of such microservices and what it's like to work with Azure Aspire. In my experience, Azure Aspire is a real boon for building and composing a microservices application.
> 
> So, let's get started!

## Architecture of the application and a first look 

``` mermaid
flowchart TD
    A[React UI Client] --> B[Python Ollama server]
    B --> E[Containerized Ollama]
    A --> C[.NET weather API server]
    C --> D[Containerized PostgreSQL]
   
    classDef container stroke:#333,stroke-width:1px;
    class A,B,C,D,E,F container;
```
The application features a straightforward architecture designed, since my purpose in this project is to explore the full development experience in Azure Aspire with multiple polyglot services.

## Okay what is Azure Aspire though? 
