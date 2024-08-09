---
layout: post
title: Continuous Deployment of a Quarkus Application On Google Kubernetes Engine Using GitHub Actions
date: 2024-06-12 21:46:01 +09:00
categories: [Cloud, GKE]
tags: [quarkus, java, gke, githubaction, jib]                    
---

> In this post, we'll explore how to set up a GitHub Actions workflow for continuously deploying a Quarkus application to Google Kubernetes Engine (GKE).
> 
> This post will take you through the process of building a solid CI/CD pipeline, starting with the development experience in Quarkus, followed by the build and containerization of our application using Jib, and finally deploying our Quarkus app to Google Kubernetes Engine (GKE) with GitHub Actions.

## Meet Quarkus: A Modern Java Framework for Cloud-Native Development

Codes of the application are at: [GitHub Repository](https://github.com/CynicDog/archeio)

Quarkus is a Java runtime designed for Kubernetes, supporting MicroProfile and other specifications in microservices architecture. It offers a Java execution environment with built-in tools for developing modern applications. The result is a developer experience that meets the expectations of Kubernetes platform administrators.

Okay, lots of big words here! Let me rephrase the above introduction of Quarkus in my personal understanding: Quarkus is a Kubernetes-native Java execution environment with its native support for packaging an application into an executable image and deploy on Kubernetes engine. There are lots of other features supported by Quarkus but I’ll mainly be highlighting its Kubernetes support in this post. 

Before we check out Quarkus's native Kubernetes support, let’s quickly review the layers of my application, Archeio, which is a simple rich-text note app. The stacks of each layer are the familiar ones for Java developers in the familiar project structure, since Quarkus is effectively a Java runtime that has excellent compatibility. 

### Persistence with JPA 

For developer joy, Quarkus provide a zero-config database out of the box in development environment. 

With database extension included in [pom.xml](https://github.com/CynicDog/archeio/blob/master/pom.xml), and no explicit configuration for a database connection, Quarkus runs a container based on the official Docker image of the database. My choice for this application was PostgreSQL. For production, of course, a persistent service is needed, so we're going to configure the connection to the database server: 

{% raw %} 
```properties 
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://postgres:5432/${quarkus.application.name}
```
{% endraw %}

Note that it's prefixed with {% raw %} `%prod` {% endraw %} to single out the production environment, effectively giving no explicit configuration for development so we can use the test container before production.

In Quarkus, we don't have to write up `persistence.xml` file to configure the JPA driver, since it's configured by Quarkus with sensible defaults. All we need is to wire up `EntityManager` in usage context as [below](https://github.com/CynicDog/archeio/blob/master/src/main/java/io/cynicdog/Folder/FolderRepository.java):

```java
@ApplicationScoped
public class FolderRepository {

    @Inject
    EntityManager em;

    // database operations .. 
}
```
Quarkus is compatible with Spring Data JPA APIs, and the reason of using `EntityManager` is that I simply love the direct interaction with it.

For production, we will use a standalone persistence service in our deployment environment. For that, we need a Kubernetes [manifest](https://github.com/CynicDog/archeio/blob/master/postgresql_kubernetes.yml) configuration for Postgres service to be deployed on our target engine in future. 

That covers the persistence implementation of the application. Now, let’s look at the application APIs, which combines Vert.x and JAX-RS.

### Oauth2 Implementation in Vert. and RESTful services in JAX-RS 

Quarkus also supports Vert.x, a low-level toolkit for building RESTful web applications. While Vert.x deserves its own dedicated articles, this post will focus on how it's practically used in my application. 

I directly used the Vert.x instance `vertx` that exposes core APIs without deploying any [Verticles](https://vertx.io/docs/vertx-core/java/#_verticles) as below: 

```java
@ApplicationScoped
public class RouterRegistry {

    @Inject
    Vertx vertx;

    public void init(@Observes Router router,
                    // ... 
                    ) {

        var githubAPI = new GithubAPI(vertx, clientId, clientSecret, redirectionUrl, host, port);

        router.get("/sign-in").handler(githubAPI::signIn);
        router.get("/callback").handler(githubAPI::callback);
    }
}
```
Quarkus is certainly capable of managing multiple Vert.x verticles, but we won’t go into that, as we are using Vert.x to implement the GitHub OAuth2 [authentication flow](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps). 

Now, let’s take a quick look at the implementation of GitHub OAuth2 in Vert.x. The first step is to set up the OAuth2 client. 

Quarkus provides `@ConfigProperty`, a MicroProfile Config implementation for CDI injection, allowing us to provide sensitive information at runtime as environment variables. Alternatively, we can create a Kubernetes `ConfigMap` and have the deployment reference it at runtime as [below](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/.github/workflows/deploy-quarkus-to-gke.yml#L58): 

{% raw %} 
```yml
    - name: Create Kubernetes ConfigMap
      run: |
        
        kubectl create configmap archeio-github-app \
          --from-literal=archeio.github.app.client.id=${{ secrets.ARCHEIO_GITHUB_APP_CLIENT_ID }} \
          --from-literal=archeio.github.app.client.secret=${{ secrets.ARCHEIO_GITHUB_APP_CLIENT_SECRET }};
```
{% endraw %} 

> To enable Quarkus to identify and reference the ConfigMap, we need to activate Kubernetes configuration in the [application.properties](https://github.com/CynicDog/archeio/blob/master/src/main/resources/application.properties#L38) file, along with adding [the necessary extension](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/pom.xml#L81). This is what really defines Quarkus as a Kubernetes-native Java runtime. 

Next is to map a [handler](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/src/main/java/io/cynicdog/GithubAPI.java#L48) for the `/sign-in` route, where we define the scopes of access. We will also map the `/callback` [handler](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/src/main/java/io/cynicdog/GithubAPI.java#L64) to complete the implementation of Oauth2 flow by verifying the state value and obtaining the access token from GitHub.

Any further business logic after the authentication flow, such as creating a folder and writing a new post on it, is managed by JAX-RS endpoints. 

### Quinoa: A Quarkus extension to create Modern UI with no hassle

[Quinoa](https://github.com/quarkiverse/quarkus-quinoa) is a Quarkus extension that simplifies the integration of modern frontend web applications built with Node.js into Quarkus-based backends. The main requirement is that the Node.js project includes a `build` script that outputs static files—like index.html, JavaScript, and CSS—into a build artifact directory. Quinoa packages these static files into the Quarkus application—whether as a JAR, binary, or even within a container image—during the build process, ensuring they are served efficiently when the application runs.

I decided to use React for the UI client because of my familiarity with it ([see the UI directory](https://github.com/CynicDog/archeio/tree/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/src/main/webui)), but Quinoa also supports other frontend frameworks and libraries like Angular, Vue, Svelte, and more.  

## Quarkus and Jib: A Dynamic Duo for Modern Java Deployment

So far we’ve covered the fluency of Quarkus across various levels—from persistence to RESTful controllers, and finally the UI. Now it’s time to capture our Quarkus application into an executable image and generate the Kubernetes manifests for deployment to our target environment. 

Developing Kubernetes-native applications with Quarkus means packaging the application into an executable container image and generating a Kubernetes manifest automatically, ready to work with Kubernetes as the deployment platform.

By including the `quarkus-kubernetes` extension in our dependencies, running `mvn clean install` will produce Service and Deployment manifests for our application in the /target/kubernetes directory. It provides instructions on how the application will be deployed, meaning we still need a pre-built image to deploy it.

With the help of the [Jib](https://github.com/GoogleContainerTools/jib) extension for Quarkus, `quarkus-container-image-jib`, the command `mvn clean package -Dquarkus.container-image.build=true` enables you to build a Docker image along with the deployment manifest file. Jib manages the process of optimizing layers and dependencies of an application, generating the image specified in the Kubernetes manifest. This is one of the goals that Jib promotes: Daemonless. Jib effectively abstracts Docker CLI interactions, allowing you to manage everything with just a few Maven command lines, which proves especially useful in GitHub Actions where such declarative commands are employed. 

So, let’s take a look at what the generated manifest looks like.

First configuration we're going to talk about is Ingress resource. The Ingress resource plays a vital role in routing external HTTP traffic to our Quarkus application inside the Kubernetes cluster. It defines rules to control how requests are directed to specific services based on the request path or hostname. 

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    app.quarkus.io/quarkus-version: 3.9.2
    # ... 
    kubernetes.io/ingress.allow-http: "true"
    kubernetes.io/ingress.class: gce
    kubernetes.io/ingress.global-static-ip-name: archeio-global
  labels:
    app.kubernetes.io/name: archeio
    # ... 
  name: archeio
spec:
  rules:
    - http:
        paths:
          - backend:
              service:
                name: archeio
                port:
                  name: http
            path: /
            pathType: Prefix
```
> Since the application will utilize the DNS name `www.archeio.xyz`, having a static IP (named `archeio-global` for this project) is a necessity since it ensures consistent and reliable mapping of domain names to a fixed server location. For instructions on creating the IP address and linking it to the Quarkus context, refer to the details [here](https://github.com/CynicDog/archeio?tab=readme-ov-file#reserve-static-ip-address-for-the-deployed-app).
 
Ingress acts as a reverse proxy, using a standardized declarative configuration that we are seeing right now to define rules for routing web traffic to services within the cluster. That's the key advantage of using Ingress: its ability to bring together and simplify the management of multiple route(s) and service(s) under a single external IP address.

You need an Ingress controller to fulfill an Ingress resource; creating an Ingress on its own won’t accomplish anything. Fortunately, Google Kubernetes Engine (GKE) comes with a built-in and managed Ingress controller called GKE Ingress, so we are good to go. 




## GitHub Actions: Powering Your Code with Automated Flow

### Introduction to GitHub Actions
- Overview of GitHub Actions
- Benefits for CI/CD pipelines
The GitHub Actions workflow is defined in a YAML file. Here’s a breakdown of the key components and steps involved:

### 1. Triggering the Workflow

The workflow is triggered manually using the `workflow_dispatch` event, allowing you to start the deployment process on demand.

```yaml
on:
  workflow_dispatch: 
```

### 2. Setting Environment Variables

Environment variables store project-specific information such as the GCP project ID, GKE cluster name, zone, and deployment name.

```yaml
env: 
  PROJECT_ID: '{YOUR_PROJECT_ID}'
  GKE_CLUSTER: '{YOUR_CLUSTER_NAME}'
  GKE_ZONE: '{YOUR_ZONE}'
  DEPLOYMENT_NAME: '{SOME_NAME_FOR_DEPLOYMENT}'
```

### 3. Job Definition

A job named `setup-build-publish-deploy` is defined to run on the latest Ubuntu runner. This job includes various steps to set up the environment, build the application, and deploy it to GKE.

```yaml
jobs:
  setup-build-publish-deploy:
    name: Setup, Build, Publish, and Deploy
    runs-on: ubuntu-latest
    environment: production
    
    permissions:
      contents: 'read'
      id-token: 'write'
```

### 4. Steps Breakdown

Let's go through each step in detail:

#### Checkout the Code

The first step checks out the code from the repository.

```yaml
steps:
- name: Checkout
  uses: actions/checkout@v4
```

#### Setup Java Environment

The Java environment is set up using the `actions/setup-java` action.

```yaml
- uses: actions/setup-java@v4.2.1
  with:
    distribution: 'temurin'
    java-version: '17'
```

#### Authenticate with Google Cloud

This step uses the `google-github-actions/auth` action to authenticate with Google Cloud using a service account key stored in GitHub secrets.
{% raw %} 
```yaml
- id: 'auth'
  uses: 'google-github-actions/auth@v2'
  with:
    credentials_json: '${{ secrets.GOOGLE_CREDENTIALS }}'
```
{% endraw %}

To get the credentials JSON, run the following commands in your terminal:

```bash
gcloud init 
gcloud iam service-accounts create {A_SERVICE_ACCOUNT_NAME}
gcloud iam service-accounts list
```

This will return an address in the format of `{A_SERVICE_ACCOUNT_NAME}@{YOUR_PROJECT_ID}.iam.gserviceaccount.com`. Next, grant permissions:

```bash
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/container.admin
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/storage.admin
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/container.clusterViewer
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/artifactregistry.writer
```

Fetch the credentials secret:

```bash
gcloud iam service-accounts keys create "key.json" --iam-account "{SERVICE_ACCOUNT_ADDRESS}"
```

Convert the JSON file into a `base64` string and register it in GitHub action secrets:

```bash
$GKE_SA_KEY = [Convert]::ToBase64String([IO.File]::ReadAllBytes("key.json"))    
```

#### Get GKE Cluster Credentials

Fetch the GKE cluster credentials to interact with the cluster.
{% raw %} 
```yaml
- id: 'get-credentials'
  uses: 'google-github-actions/get-gke-credentials@v2'
  with:
    cluster_name: ${{ env.GKE_CLUSTER }}
    location: ${{ env.GKE_ZONE }}
    project_id: ${{ env.PROJECT_ID }}
```
{% endraw %}

#### Setup gcloud CLI

Install the `kubectl` component using the `google-github-actions/setup-gcloud` action. `kubectl` is needed to create a ConfigMap.

{% raw %} 
```yaml
- uses: google-github-actions/setup-gcloud@v2.1.0
  with:
    project_id: ${{ env.PROJECT_ID }}
    install_components: 
      kubectl
```
{% endraw %}

#### Configure Docker Authentication

Configure Docker to use the gcloud command-line tool for authentication.

```yaml
- name: Configure Docker authentication
  run: |
    gcloud auth configure-docker
```

#### Create Kubernetes ConfigMap

Create a Kubernetes ConfigMap for storing sensitive information, such as GitHub app credentials. This addresses the Quarkus static initialization issue, ensuring necessary services are available without encountering the chicken-egg problem during static initialization.

{% raw %} 
```yaml
- name: Create Kubernetes ConfigMap
  run: |
    if ! kubectl get configmap {A_CONFIG_MAP_NAME}; then
      kubectl create configmap {A_CONFIG_MAP_NAME} \
        --from-literal={A_CONFIG_PROPERTY_1}=${{ secrets.SOME_CONFIG_VALUE_YOU_WANT_TO_INJECT }} \
        --from-literal={A_CONFIG_PROPERTY_2}=${{ secrets.ANOTHER_CONFIG_VALUE_YOU_WANT_TO_INJECT }};
    else
      echo "ConfigMap {A_CONFIG_MAP_NAME} already exists. Skipping creation.";
    fi
```
{% endraw %}

Ensure the following options are configured in `application.properties` for the Quarkus app to recognize the Kubernetes ConfigMap:

```properties
%prod.quarkus.kubernetes-config.enabled=true
%prod.quarkus.kubernetes-config.config-maps={A_CONFIG_MAP_NAME}
A_CONFIG_PROPERTY_1=
A_CONFIG_PROPERTY_2=
```

#### Build and Push Quarkus App Image

Build the Quarkus application and push the Docker image to the image registry.

```yaml
- name: Build Quarkus App and Push to Image Registry 
  run: |
    mvn clean package -Dquarkus.container-image.build=true \
    -Dquarkus.container-image.push=true \
    -Dquarkus.jib.platforms=linux/arm64/v8 \
    --file pom.xml
```

#### Deploy Quarkus App to GKE

Deploy the built Quarkus application to GKE.

```yaml
- name: Deploy Quarkus App to GKE
  run: | 
    mvn clean package -Dquarkus.kubernetes.deploy=true \
    --file pom.xml
```

### 5. Sample Workflow Code

Find a sample code at the link below:

[GitHub Repository](https://github.com/CynicDog/archeio/blob/master/.github/workflows/deploy-quarkus-to-gke.yml)

### Conclusion

By following this GitHub Actions workflow, you can achieve continuous deployment of your Quarkus application to GKE. This setup ensures that every time you manually trigger the workflow, your application is built, containerized, and deployed to your Kubernetes cluster seamlessly. This approach automates the deployment process, reducing the potential for human error and speeding up the deployment cycle.

