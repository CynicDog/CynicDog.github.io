---
layout: post
title: Azure EntraID in Single Page Application published on Teams for organization.  
date: 2024-08-05 00:00:00 +09:00
categories: [.NET, EntraID, MSAL, Teams]
tags: [azure, aspire, ollama, microservices, docker]            
mermaid: true
---

> In this article, we will explore the the vast field of **Microsoft Entra identity platform**, with highlighting key concepts such as **Entra ID**, **Microsoft Authentication library(MSAL)**, **multi-tenant App Registrations**, **Service Principal**, **Graph API**, and exposing the application in organization's **Teams**.
> 
> To understand the complete development process of an application that leverages organizational resources using Entra ID and Graph API, and ultimately gets published on the organization's Teams, I've created a simple project with minimum implementation. The application's features include displaying the logged-in user's profile picture and presence, fetched by Graph API, after signing in with a Microsoft account email.
>
> The application is registered on the Microsoft identity platform to be exposed as an Enterprise Application to my own tenant, and with that, the application will be published in my organization's Teams.
>
> To keep the cost of implementation and deployment at lowest, the application is deployed on GitHub Pages as Single Page Application, and yes, it's a static web application, so naturally its features are limited. But it serves to demonstrate the development process of an application that sits on Microsoft identity platform, so let's get started.

## 0. A Warm Welcome 

Codes of the application are at: [GitHub Repository](https://github.com/CynicDog/azure-entra-in-spa)

```mermaid
C4Deployment
    title  
    
    Deployment_Node(github, "GitHub SPA host") {
        Container(React, "React", "Single Page Application", "")
    }

    Deployment_Node(teams, "Teams") {
        Container(Teams, "Teams", "", "")
    }

    Deployment_Node(azure, "Azure / Entra ID") {
        Deployment_Node(organizationalResources, "Publishing Organization") {
            ContainerDb(Organizational_Resources, "Organizational Resources", "")
            
            Deployment_Node(publishingOrganization, "Identity / Access Management Unit") {
                Container(Service_Principal, "Service Principal", "", "")
                Container(Enterprise_Application, "Enterprise Application", "", "")        
            }
        }

        Deployment_Node(appRegistrations, "App Registrations") {
                    Container(helloworld_app, "helloworld-app", "Graph API permissions", "User.ReadWrite, Presence.ReadWrite")
        }
    }
    
    Rel(helloworld_app, Enterprise_Application, "Exposes as")
    Rel(Teams, Service_Principal, "Delegates to")
    Rel(React, Teams, "Provides endpoint")
    BiRel(Service_Principal, Organizational_Resources, "Accesses")
    BiRel(Teams, Organizational_Resources, "On behalf of user")
    BiRel(Enterprise_Application, Service_Principal, "")

    UpdateElementStyle(azure, $borderColor="gray")
    UpdateElementStyle(github, $borderColor="gray")
    UpdateElementStyle(teams, $borderColor="gray")
    UpdateElementStyle(organizationalResources, $borderColor="gray")
    UpdateElementStyle(appRegistrations, $borderColor="gray")
    UpdateElementStyle(publishingOrganization, $borderColor="gray")
    
    UpdateRelStyle(helloworld_app, Enterprise_Application, $textColor="white", $lineColor="gray", ,$offsetX="5")
    UpdateRelStyle(Teams, Service_Principal, $textColor="white", $lineColor="gray", $offsetX="-40")
    UpdateRelStyle(React, Teams, $textColor="white", $lineColor="gray", $offsetY="-15", $offsetX="-40")
    UpdateRelStyle(Service_Principal, Organizational_Resources, $textColor="white", $lineColor="gray", $offsetY="-15", $offsetX="-40")
    UpdateRelStyle(Teams, Organizational_Resources, $textColor="white", $lineColor="gray", $offsetY="-15", $offsetX="-40")
    UpdateRelStyle(Enterprise_Application, Service_Principal, $textColor="white", $lineColor="gray")
```

The diagram above illustrates the process of registering an application on the Microsoft identity platform, configuring it for exposure to the publishing organization, and ultimately publishing the app within the Teams client.

**Registering your application** on Entra ID establishes its identity configuration, allowing it to integrate with the Microsoft identity platform. This registration means your app trusts the Microsoft identity platform to handle identity and access management tasks. 

Following registration, an **application object** is created. It serves as a global identity configuration template, functioning as a tenant-wide, or even as a cross-tenant interface. Again, it's an one-and-only global configuration template for the registered application, not a run-time instance of the application. 

So, for an organization that wants to utilize identity resources within an application (whether it's an SPA or a server-endorsed application), it needs a local representation of the registered application, ultimately to be configured with rich features of identity platform. That's exactly what a **service principal** is for: a local representation of the registration. The tenant where the app is registered has a service principal for the application, and any other tenants that grant access to the registered app will have their own service principals. 

To sum up, an application registration has:
- A one-to-one relationship with the software application (in our case, a React SPA application that uses MSAL) 
- A one-to-many relationship with its corresponding service principal object(s) for tenants. 
