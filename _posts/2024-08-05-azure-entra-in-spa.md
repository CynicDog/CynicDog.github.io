---
layout: post
title: Azure EntraID in Single Page Application published in Teams for organization.  
date: 2024-08-05 00:00:00 +09:00
categories: [.NET, EntraID, MSAL, Teams]
tags: [azure, aspire, ollama, microservices, docker]            
mermaid: true
---

> In this article, we will explore the the vast field of **Microsoft Entra identity platform**, with highlighting key concepts such as **Entra ID**, **Microsoft Authentication library(MSAL)**, **multi-tenant App Registrations**, **Service Principal**, **Graph API**, and exposing the application in organization's **Teams**.
> 
> To understand the complete development process of an application that leverages organizational resources using Entra ID and Graph API, and ultimately gets published on the organization's Teams, I've created a simple [project](https://github.com/CynicDog/azure-entra-in-spa) with minimum implementation. The application's features include displaying the logged-in user's profile picture and presence, fetched by Graph API, after signing in with a Microsoft account email.
>
> The application is registered on the Microsoft identity platform to be exposed as an Enterprise Application to my own tenant, and with that, the application will be published in my organization's Teams.
>
> To keep the cost of implementation and deployment at lowest, the application is deployed on GitHub Pages as Single Page Application, and yes, it's a static web application, so naturally its features are limited. But it serves to demonstrate the development process of an application that sits on Microsoft identity platform, so let's get started.

## 0. A Warm Welcome 

```mermaid
C4Deployment
    title  
    
    Deployment_Node(github, "GitHub SPA host") {
        Container(React, "React", "", "")
    }

    Deployment_Node(azure, "Azure") {
        Deployment_Node(organizationalResources, "Organizational Resources") {
        ContainerDb(Organizational_Resources, "Organizational Resources", "")
        
        Deployment_Node(appRegistrations, "App Registrations") {
                Container(helloworld_app, "helloworld-app", "Graph API permissions", "User.ReadWrite, Presence.ReadWrite")
            }
        }

        Deployment_Node(publishingOrganization, "Publishing Organization") {
            Container(Service_Principal, "Service Principal", "", "")
            Container(Enterprise_Application, "Enterprise Application", "", "")        
        }
    }
        
    Deployment_Node(teams, "Teams") {
        Container(Teams, "Teams", "", "")
    }
    
    Rel(helloworld_app, Enterprise_Application, "Exposes as")
    Rel(Teams, Service_Principal, "Delegates to")
    Rel(React, Teams, "Provides endpoint")
    BiRel(Service_Principal, Organizational_Resources, "Accesses")
    BiRel(Teams, Organizational_Resources, "On behalf of user")

    UpdateElementStyle(azure, $borderColor="gray")
    UpdateElementStyle(github, $borderColor="gray")
    UpdateElementStyle(teams, $borderColor="gray")
    UpdateElementStyle(organizationalResources, $borderColor="gray")
    UpdateElementStyle(appRegistrations, $borderColor="gray")
    UpdateElementStyle(publishingOrganization, $borderColor="gray")
    
    UpdateRelStyle(helloworld_app, Enterprise_Application, $textColor="white", $lineColor="gray", ,$offsetX="5")
    UpdateRelStyle(Teams, Service_Principal, $textColor="white", $lineColor="gray")
    UpdateRelStyle(React, Teams, $textColor="white", $lineColor="gray", $offsetY="-15", $offsetX="-40")
    UpdateRelStyle(Service_Principal, Organizational_Resources, $textColor="white", $lineColor="gray", $offsetY="-15", $offsetX="-40")
    UpdateRelStyle(Teams, Organizational_Resources, $textColor="white", $lineColor="gray", $offsetY="-15", $offsetX="-40")
```
