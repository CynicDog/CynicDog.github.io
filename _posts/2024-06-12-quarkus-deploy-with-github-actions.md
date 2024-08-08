---
layout: post
title: Continuous Deployment of a Quarkus Application Using GitHub Actions
date: 2024-06-12 21:46:01 +09:00
categories: [Cloud, GKE]
tags: [quarkus, java, gke, githubaction, jib]                    
---

> In this post, we'll explore how to set up a GitHub Actions workflow for continuously deploying a Quarkus application to Google Kubernetes Engine (GKE).
> 
> This post will walk you through 1) the development experience in Quarkus, 2) the build and containerization of our Quarkus application with Jib tool, then 3) the deployment on GitHub Actions on GKE, that makes a secured, streamlined CI/CD process all together the factors of Quarkus and Jib.

## Meet Quarkus: A Modern Java Framework for Cloud-Native Development

### Introduction to Quarkus
- Overview of Quarkus
- Key features: Fast startup time, low memory usage

### Development with Quarkus
- **Persistence with JPA**: Integrating a PostgreSQL database
- **Configuration Injection**: Managing application settings
- **Vert.x Compatibility**: Reactive programming support
- **GitHub OAuth2 Flow**: Implementing OAuth2 authentication
- **React Integration**: Using Quinoa for SPA and routing

## Quarkus and Jib: A Dynamic Duo for Modern Java Deployment

### Why Jib?
- Benefits of using Jib for containerization
- Integration with Maven for seamless builds

### Jib Containerization Process
- **Generated Manifests Overview**
- **Artifacts Overview**

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

