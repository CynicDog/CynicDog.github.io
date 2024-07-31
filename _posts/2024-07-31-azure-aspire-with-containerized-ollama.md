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

## Architecture of the application and a first look.  

Codes of the application are at: [GitHub Repository](https://github.com/CynicDog/Aspiring-Ollama)

``` mermaid
flowchart TD
    
    A([React UI Client]) --> B([Python Ollama server])
    B --> E([Containerized Ollama])
    A --> C([.NET weather API server])
    C --> D([Containerized PostgreSQL])
    F([Aspire AppHost]) --- |react.bindings.http.targetPort| A
    F --- |ollamaservice.bindings.http.targetPort| B
    F --- |apiservice.bindings.http.targetPort| C
    F --- |connectionString| D
    F --- |ollama.bindings.ollama-uri.url| E
     
    classDef container stroke:#333,stroke-width:1px;
    class A,B,C,D,E container;

    linkStyle 4,5,6,7,8 stroke-width:.3px,color:grey;
```
The application features a straightforward architecture designed to explore the full development experience in Azure Aspire with multiple polyglot services.

The React project provides a UI client where users can send requests to pull LLM models to the Python server, where the requests are simply propagated to the containerized Ollama. After successfully pulling models, users can interact with these LLM models by sending prompts, again, to the Python server. The Python server retrieves generated answers from Ollama and delivers the data in the form of a stream.

The React UI client also has a simple visual representation of weather data that is stored in PostgreSQL in its own container and fetched through the .NET API server. The weather business logic here is textbook material in .NET projects that you'd often find in any other sample codes in the .NET frameworks ecosystem. I made slight modifications to how they are implemented in the application, but the purpose of the domain remains the same: to provide a quick look at how the apps communicate with each other.

What's happening behind the UI is a bit more complicated, and that's where the aspiring features of Azure Aspire help in.  A key point is that it provides network configurations for service bindings and connections through the use of environment variables.

## Charms of Azure Aspire.  
