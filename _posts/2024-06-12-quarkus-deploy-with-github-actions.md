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

## 0. Meet Quarkus: A Modern Java Framework for Cloud-Native Development

- Codes of the application are at: [GitHub Repository](https://github.com/CynicDog/archeio)
- GitHub Actions CI/CD workflow to deploy Quarkus app on GKE is at: [deploy-quarkus-to-gke.yml](https://github.com/CynicDog/archeio/blob/master/.github/workflows/deploy-quarkus-to-gke.yml)

Quarkus is a Java runtime designed for Kubernetes, supporting MicroProfile and other specifications in microservices architecture. It offers a Java execution environment with built-in tools for developing modern applications. The result is a developer experience that meets the expectations of Kubernetes platform administrators.

Okay, lots of big words here! Let me rephrase the above introduction of Quarkus in my personal understanding: Quarkus is a Kubernetes-native Java execution environment with its native support for packaging an application into an executable image and deploy on Kubernetes engine. There are lots of other features supported by Quarkus but I’ll mainly be highlighting its Kubernetes support in this post. 

Before we check out Quarkus's native Kubernetes support, let’s quickly review the layers of my application, Archeio, which is a simple rich-text note app. The stacks of each layer are the familiar ones for Java developers in the familiar project structure, since Quarkus is effectively a Java runtime, which 'happens to have' an excellent compatibility. 

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
Quarkus is certainly capable of managing multiple Vert.x verticles, but we won’t go into that, as we are using Vert.x only to implement the GitHub OAuth2 [authentication flow](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps). 

Now, let’s take a quick look at the implementation of GitHub OAuth2 in Vert.x. The first step is to set up the OAuth2 client. 

Quarkus provides `@ConfigProperty`, a MicroProfile Config implementation for CDI, allowing us to provide sensitive information at runtime as environment variables. Alternatively, we can create a Kubernetes `ConfigMap` and have the deployment environment reference it at runtime as [below](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/.github/workflows/deploy-quarkus-to-gke.yml#L58): 

{% raw %} 
```yml
    - name: Create Kubernetes ConfigMap
      run: |
        
        kubectl create configmap archeio-github-app \
          --from-literal=archeio.github.app.client.id=${{ secrets.ARCHEIO_GITHUB_APP_CLIENT_ID }} \
          --from-literal=archeio.github.app.client.secret=${{ secrets.ARCHEIO_GITHUB_APP_CLIENT_SECRET }};
```
{% endraw %} 
> To enable Quarkus to identify and reference the ConfigMap, we need to activate Kubernetes configuration in the [application.properties](https://github.com/CynicDog/archeio/blob/master/src/main/resources/application.properties#L38) file, along with adding [an extension](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/pom.xml#L81). Such affinity with Kubernetes is what really defines Quarkus as a Kubernetes-native Java runtime. 

Next is to map a [handler](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/src/main/java/io/cynicdog/GithubAPI.java#L48) for the `/sign-in` route, where we define the scopes of access. We will also map the `/callback` [handler](https://github.com/CynicDog/archeio/blob/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/src/main/java/io/cynicdog/GithubAPI.java#L64) to complete the implementation of Oauth2 flow by verifying the state value and obtaining the access token from GitHub.

Any further business logic after the authentication flow, such as creating a folder and writing a new post on it, is managed by JAX-RS endpoints. 

### Quinoa: A Quarkus extension to create Modern UI with no hassle

[Quinoa](https://github.com/quarkiverse/quarkus-quinoa) is a Quarkus extension that simplifies the integration of modern frontend web applications built with Node.js into Quarkus-based backends. The main requirement is that the Node.js project includes a `build` script that outputs static files—like index.html, JavaScript, and CSS—into a build artifact directory. Quinoa packages these static files into the Quarkus application—whether as a JAR, binary, or even into a container image—during the build process, ensuring they are served efficiently when the application runs.

I decided to use React for the UI client because of my familiarity with it ([see the UI directory](https://github.com/CynicDog/archeio/tree/3cdb23b4675c72d0a7a4da483e624cf9af7afe4d/src/main/webui)), but Quinoa also supports other frontend frameworks and libraries like Angular, Vue, Svelte, and more.  

## 1. Quarkus and Jib: A Dynamic Duo for Modern Java Deployment

So far we’ve covered the fluency of Quarkus across various levels—from persistence to RESTful controllers, and finally the UI. Now it’s time to capture our Quarkus application into an executable image and generate the Kubernetes manifests for deployment to our target environment. 

Developing Kubernetes-native applications with Quarkus means packaging the application into an executable container image and generating a Kubernetes manifest automatically, ready to work with Kubernetes as the deployment platform.

By including the `quarkus-kubernetes` extension in our dependencies, running `mvn clean install` will produce Service and Deployment manifests for our application in the `/target/kubernetes` directory. It provides instructions on how the application will be deployed, meaning we still need a pre-built image to deploy it.

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
> Since the application will utilize the DNS name `www.archeio.xyz`, having a static IP (named `archeio-global` for this project) is a necessity, since it ensures a consistent and reliable connection between the domain name and a specific endpoint in the cloud. For instructions on creating the IP address and linking it to the Quarkus context, refer to the details [here](https://github.com/CynicDog/archeio?tab=readme-ov-file#reserve-static-ip-address-for-the-deployed-app).

Ingress acts as a reverse proxy, using a standardized declarative configuration that we are seeing right now to define rules for routing web traffic to services within the cluster. That's the key advantage of using Ingress: its ability to bring together and simplify the management of multiple route(s) and service(s) under a single external IP address.

You need an Ingress controller to fulfill an Ingress resource; creating an Ingress on its own won’t accomplish anything. Fortunately, Google Kubernetes Engine (GKE) comes with a built-in and managed Ingress controller called GKE Ingress, so we are good to go. 

Next, we have RBAC resources specifications in the generated manifest. 

```yml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: archeio
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: view-secrets
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: archeio-view-secrets
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: view-secrets
subjects:
  - kind: ServiceAccount
    apiGroup: ""
    name: archeio
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: archeio-view
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: view
subjects:
  - kind: ServiceAccount
    name: archeio
```
The `ServiceAccount` provides an identity for our Quarkus application, allowing it to authenticate with the Kubernetes API and access resources. The `Role` and two `RoleBindings` define permissions for reading `Secrets` and general read-only access to resources, ensuring secure and effective interaction with the Kubernetes environment.

Then we have the Service and Deployment configurations, which are the essential building blocks for orchestrating our Quarkus application within the Kubernetes environment.

```yml
apiVersion: v1
kind: Service
metadata:
  # ... 
  name: archeio
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
  selector:
    app.kubernetes.io/name: archeio
    app.kubernetes.io/version: 1.0.0-SNAPSHOT
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  # ... 
  name: archeio
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: archeio
      app.kubernetes.io/version: 1.0.0-SNAPSHOT
  template:
    # ... 
    spec:
      containers:
        - env:
            - name: KUBERNETES_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          image: us.gcr.io/encoded-etching-425009-t7/archeio:1.0.0-SNAPSHOT
          imagePullPolicy: Always
          name: archeio
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
      serviceAccountName: archeio
---
```
> The specified image tag above (`us.gcr.io/encoded-etching-425009-t7/archeio:1.0.0-SNAPSHOT`) is determined by the configuration set in the [application.properties](https://github.com/CynicDog/archeio/blob/746aca0e2316dc7243f89ede2abbd387f8969fcb/src/main/resources/application.properties#L21) file. The image tag corresponds to the container image we push to Google Cloud Platform's Container Registry using the command `mvn clean package -Dquarkus.container-image.build=true -Dquarkus.container-image.push=true`. Deployment to GCP will be detailed in the upcoming section.

The manifest above is a standard, practical configuration generated for Kubernetes. Services expose our application to network traffic, while Deployments manage the rollout and scaling of your application instances. 

With the foundational Kubernetes configurations in place, our next step is to automate the deployment process using GitHub Actions for continuous integration and continuous deployment (CI/CD). 

## 2. GitHub Actions: Powering Your Code with Automated Flow

The Kubernetes-native features of Quarkus, combined with the effortless, daemonless generation of container images and manifests, make GitHub Actions a perfect fit for automating the deployment of our application to Google Kubernetes Engine (GKE).

GitHub Actions doesn’t just automate your CI/CD pipelines; it also ensures that the credentials and sensitive data are handled with the utmost security, all within a runner-provided environment that keeps everything running securely.

Let's see details of the [workflow](https://github.com/CynicDog/archeio/blob/master/.github/workflows/deploy-quarkus-to-gke.yml) to deploy a Quarkus application to GKE. 

### a. Triggering the Workflow

```yaml
on:
  workflow_dispatch: 
```
> By using the `workflow_dispatch` event, you can manually trigger the workflow whenever you need to, giving you control over the deployment process. While you can certainly add other triggers like `push` or `pull_request`, I prefer to keep it straightforward with just `workflow_dispatch` for a more intentional approach to deployments. Of course, as codebase expands, more sophisticated triggering rules will be required.

### b. Configuring Information for the Target Environment

```yaml
env: 
  PROJECT_ID: '{YOUR_PROJECT_ID}'
  GKE_CLUSTER: '{YOUR_CLUSTER_NAME}'
  GKE_ZONE: '{YOUR_ZONE}'
  DEPLOYMENT_NAME: '{SOME_NAME_FOR_DEPLOYMENT}'
```
> Environment variables store project-specific information such as the GCP project ID, GKE cluster name, zone, and deployment name. If these values are considered as sensitive information, you may hide them as repository secrets. 

### c. Defining the Runner and Java Environment 

```yaml
jobs:
  setup-build-publish-deploy:
    name: Setup, Build, Publish, and Deploy
    runs-on: ubuntu-latest
    environment: production
    
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - uses: actions/setup-java@v4.2.1
      with:
        distribution: 'temurin'
        java-version: '17'
```
> This part of the workflow sets up the workflow environment, configures necessary permissions, and establishes the Java setup for building and deploying the application.

### d. Authenticate with Google Cloud

In order to authenticate the GitHub Actions runtime on Google Cloud environment, we need to first create a service account and give required permissions to push the container image and deploy it on GKE. 

Let's start with creating the service account by running the below commands on a local terminal: 

```bash
gcloud init 
gcloud iam service-accounts create {A_SERVICE_ACCOUNT_NAME}
gcloud iam service-accounts list
```
> This will return an address in the format of `{A_SERVICE_ACCOUNT_NAME}@{YOUR_PROJECT_ID}.iam.gserviceaccount.com`.

Next, grant permissions to the service account:

```bash
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/container.admin
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/storage.admin
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/container.clusterViewer
gcloud projects add-iam-policy-binding {YOUR_PROJECT_ID} --member=serviceAccount:{SERVICE_ACCOUNT_ADDRESS} --role=roles/artifactregistry.writer
```

Issue the credentials secret for for the service account:

```bash
gcloud iam service-accounts keys create "key.json" --iam-account "{SERVICE_ACCOUNT_ADDRESS}"
```

In order to save the credential on GitHub repository, we need to convert the JSON file into a `base64` string and ultimately register it on repository secrets:

```bash
GKE_SA_KEY=$(base64 -i key.json)
```
> In case you are a Window user, run : `$GKE_SA_KEY = [Convert]::ToBase64String([IO.File]::ReadAllBytes("key.json"))`

Save the string as a repository secret, so a GitHub Action runtime can refer to it in the workflow as below: 

{% raw %} 
```yaml
- id: 'auth'
  uses: 'google-github-actions/auth@v2'
  with:
    credentials_json: '${{ secrets.GOOGLE_CREDENTIALS }}'
```
{% endraw %}

### e. Get GKE Cluster Credentials

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
> This action configures authentication to a GKE cluster. 

### f. Setup gcloud CLI

{% raw %} 
```yaml
- uses: google-github-actions/setup-gcloud@v2.1.0
  with:
    project_id: ${{ env.PROJECT_ID }}
    install_components: 
      kubectl
```
{% endraw %}
> Install the `kubectl` component using the `google-github-actions/setup-gcloud` action. `kubectl` is needed to create a ConfigMap in later step.

### g. Configure Docker Authentication

```yaml
- name: Configure Docker authentication
  run: |
    gcloud auth configure-docker
```
> Configure Docker to use the gcloud command-line tool for authentication. This step is needed for us to push the container image that we are going to generate with Jib tool in maven command. 

### h. Create Kubernetes ConfigMap

To address Quarkus's static initialization challenges, let's create a Kubernetes ConfigMap to securely store GitHub app credentials. This ensures Quarkus can access necessary configurations at runtime, avoiding issues during application startup.

{% raw %} 
```yaml
- name: Create Kubernetes ConfigMap
  run: |      
    kubectl create configmap archeio-github-app \
      --from-literal=archeio.github.app.client.id=${{ secrets.ARCHEIO_GITHUB_APP_CLIENT_ID }} \
      --from-literal=archeio.github.app.client.secret=${{ secrets.ARCHEIO_GITHUB_APP_CLIENT_SECRET }};
```
{% endraw %}

Quarkus will then recognize the ConfigMap and bring these values into the application configuration context.

```properties
%prod.quarkus.kubernetes-config.enabled=true
%prod.quarkus.kubernetes-config.config-maps=archeio-github-app
archeio.github.app.client.id=
archeio.github.app.client.secret=
```

### i. Build and Push Quarkus App Image

```yaml
- name: Build Quarkus App and Push to Image Registry 
  run: |
    mvn clean package -Dquarkus.container-image.build=true \
    -Dquarkus.container-image.push=true \
    -Dquarkus.jib.platforms=linux/arm64/v8 \
    --file pom.xml
```
> This step builds the Quarkus application using Jib and subsequently pushes the Docker image to the Container Registry. The target location for the image push is pre-configured in our application.properties file, directing the image to the specified repository in the Container Registry. 

### j. Deploy Quarkus App to GKE

```yaml
- name: Deploy Quarkus App to GKE
  run: | 
    mvn clean package -Dquarkus.kubernetes.deploy=true \
    --file pom.xml
```
> Deploy the built Quarkus application to GKE. 

## 3. Conclusion

In this post, we explored how to set up a CI/CD pipeline to deploy a Quarkus application to Google Kubernetes Engine (GKE). We walked through the process of using Quarkus’s Kubernetes features, building Docker images with Jib, and automating deployments with GitHub Actions. One of the important take-aways was to to create Kubernetes ConfigMaps to manage sensitive information securely while ensuring the application functions correctly on cloud deployment process. 

This project gave me some valuable insights into coding within a cloud-native platform and deploying a functioning service on Google Kubernetes Engine (GKE). I’ve gained a deep appreciation for the benefits of using GitHub Actions for automation and secured configuration injection, and I surely enjoyed the development experience with Kubernetes-native Quarkus. 
