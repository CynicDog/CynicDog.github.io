---
layout: post
title: Keycloak Identity Brockering Integration in Cloud Native Spring  
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

## 0. Project Overview 
Codes of the application are at: [GitHub Repository](https://github.com/CynicDog/spa-spring-keycloak-oauth2)

```mermaid 
flowchart TD
    A([React UI Client / Browser ]) ----> |"[1] Login Request / Data Request"| B(Backend for Frontend)
    B ---> |"[2] Auth Request"| C(Keycloak Identity Broker)
    C ---> |"[3] Identity Provider Login / Consent to Scopes"| D(Identity Providers - GitHub, Microsoft ...)
    C --> |"[4] Auth Response"| B
    B ---> |"[5] Fetch Data Request - Token Relay"| E(Remote Service)
    B --> |"[6] Return Data"| A

    classDef container stroke:#333,stroke-width:1px;
    class ,E container;

    classDef react stroke:#BBDEFB,stroke-width:2px;
    class A react

    classDef idp stroke:#A020F0,stroke-width:2px;
    class C,D idp

    classDef remote stroke:#01796F,stroke-width:2px;
    class B,E remote
```

## 1. Backend for Frontend: a Spring Cloud Gateway Project 
The Backend for Frontend (BFF) pattern serves as an essential edge service that interacts with Single Page Applications (SPAs) while handling security concerns. As a gateway, the BFF acts as an intermediary between the SPA and various backend services. It efficiently routes requests from the SPA to the appropriate services. Spring Cloud Gateway is an ideal choice for implementing a BFF in this scenario, since it provides a lightweight, scalable, and highly customizable API gateway solution, perfectly suited to act as the edge service for SPAs. 

When it comes to security management, the BFF is capable of performing a central role in authentication. It can integrate with identity providers, such as Keycloak, to manage user sessions and validate tokens. The gateway service, equipped with the Spring Security dependency, will securely handle user interactions with the project's features. I'll use 'gateway' and 'backend-for-frontend' interchangeably to refer to the project.

Let's start a project with dependencies in place: 
```gradle
dependencies { 
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
}
```
> See [build.gradle](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/main/backend-for-frontend/build.gradle) for for the complete list of dependencies and project configuration details

Since the BFF acts as a gateway for both the React UI and the remote server (referred to as the resource server in the security context), we can configure the security settings accordingly.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: react-ui-route
          uri: ${REACT_UI_URL:http://localhost:8880}
          predicates:
            - Path=/,/*.css,/*.js,/favicon.ico,/assets/**,/*.svg
        - id: authenticated-service-route
          uri: ${REMOTE_SERVICE_URL:http://localhost:9001}/remote-service
          predicates:
            - Path=/remote-service/**
      default-filters:
        - TokenRelay
```
> Any request URI for static files used in React rendering, such as `/`, `/*.css`, `/*.js`, and `/assets/**`, will be routed to the React UI service via the injected URI. The rest will be routed to the remote service at the specified URI, ensuring that any requests matching the path `/remote-service/**` are directed to the appropriate backend resource. See [application.yml](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/main/backend-for-frontend/src/main/resources/application.yml) for for the complete configuration.

You may have noticed the default `TokenRelay` filter configuration applied to every request coming into the BFF service. A Token Relay occurs when an OAuth2 consumer, in our case the BFF service, acts as a client and forwards the incoming token with outgoing resource requests, such as those to the React UI and remote service.


## 2. Securing OAuth2 Flows in SPAs with Spring Security

### 2.1. Configuring Web Filters on the gateway server 

Now that we have configured the routing conditions on requests, let's apply authentication logics on incoming request to the BFF service. 

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
      return http
          .authorizeExchange(exchange -> exchange
                .pathMatchers("/assets/**").permitAll()
                .pathMatchers("/", "/*.css", "/*.js", "/favicon.ico", "/*.svg").permitAll()
                .anyExchange().authenticated()
          )
          .exceptionHandling(exceptionHandling -> exceptionHandling
                .authenticationEntryPoint(new HttpStatusServerEntryPoint(HttpStatus.UNAUTHORIZED)))
          .oauth2Login(Customizer.withDefaults())
          .csrf(csrf -> csrf
                .csrfTokenRepository(CookieServerCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new SpaServerCsrfTokenRequestHandler())
          )
          .build();
      }
      // ...
```
> In Spring Security 6, the annotations `@EnableWebFluxSecurity` and `@EnableReactiveMethodSecurity` no longer include the `@Configuration` annotation by default. Therefore, whenever you use these annotations in your security configuration, you must explicitly add the `@Configuration` annotation to the class. See [SecurityConfig.java](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/main/backend-for-frontend/src/main/java/io/cynicdog/backendforfrontend/config/SecurityConfig.java) for for the complete configuration.

Here, we configure requests routed to the React UI to be permitted, while all other requests, such as those to the remote service, require authentication, which will be handled by Token Relay.

Another important take-away from this snippet is the CSRF configuration for single-page applications (SPAs) that enables secure interactions with the server while maintaining usability: 
  - `CookieServerCsrfTokenRepository.withHttpOnlyFalse` allows the browser to access cookies using JavaScript code.
  - `SpaServerCsrfTokenRequestHandler` allows JavaScript applications to access not just the plain token values, but also encoded ones. 

There is one more filter configuration, [csrfWebFilter](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/9bcc8bbfcce0c77c9c106f42b01601e71cc07b12/backend-for-frontend/src/main/java/io/cynicdog/backendforfrontend/config/SecurityConfig.java#L43), to be registered for handling CSRF tokens since we are using the reactive core of Spring WebFlux for our web server. Reactive repositories won't save the token unless there is a subscription to the result, so we need to subscribe to the token-recognition behavior. 

### 2.2. Keycloak as a Identity Brocker  

The Spring Gateway project needs the Spring Security OAuth2 Client dependency to support OAuth2 functionality for successful startup, which means that the Keycloak identity server must be up and running with a configuration that registers the gateway project as a security client. Run the following commands inside the Keycloak container: 

```bash
./opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 \
  --realm master \
  --user cynicdog \
  --password cynicdog
```
> Logs into the server as user `cynicdog` of realm master. This is needed to create another realm for our project before any operation.

```
./opt/keycloak/bin/kcadm.sh create realms -s realm=cynicdog -s enabled=true 
```
> Creates a dedicated realm for our project. 

```
./opt/keycloak/bin/kcadm.sh create clients -r cynicdog \
    -s clientId=backend-for-frontend \
    -s enabled=true \
    -s publicClient=false \
    -s secret=bff_client_secret \
      -s 'redirectUris=["http://{ENV_HOST}:9000", "http://{ENV_HOST}:9000/login/oauth2/code/*"]'
```
> Registers the `backend-for-frontend` service as a security client to Keycloak server. 

```
/opt/keycloak/bin/kcadm.sh create identity-provider/instances \
	-r cynicdog \
	-s alias=github \
	-s providerId=github \
	-s enabled=true  \
	-s 'config.useJwksUrl="true"' \
	-s config.clientId={GITHUB_APP_CLIENT_ID} \
	-s config.clientSecret={GITHUB_APP_CLIENT_SECRET}
```
> Registers GitHub identity provider credentials. See [keycloak-config-docker.sh](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/main/manifests/keycloak-config-docker.sh) and [keycloak-config-minikube.sh](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/main/manifests/keycloak-config-minikube.sh) for for the complete configuration. Since we are using GitHub as one of the identity providers for our application, we also need to create a GitHub App on GitHub's server to obtain the client ID and client secret necessary for OAuth2 authentication. Follow the instruction [here](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/9e350f824ddba65cf9cc9bacd60978ccbba040e9/README.md?plain=1#L221) for GitHub App registration.  

Now that we have configured the gateway server and Keycloak server, it's time to follow and understand the journey of OAuth2 authentication flow with Keycloak as a identity brocker. 

React is where we write the entrypoint of OAuth2. Here's the very trigger of the authentication flow, where users click a login button on UI : 

```javascript
const LoginButton = () => {
    return (
        <>
            <div style={{margin: "5px"}}>
                <button onClick={() => { window.open('/oauth2/authorization/keycloak', '_self'); }}>
                    login
                </button>
            </div>
        </>
    )
}
```
> `window.open('/oauth2/authorization/keycloak', '_self')` opens the URL that initiates the OAuth2 login flow. Since the React project is behind the gateway service, `_self` refers to the gateway's host, where Spring Security handles the `/oauth2/authorization/{REGISTRATION_ID}` endpoint — `keycloak` in our case. Spring Security then delegates to Keycloak, redirecting the browser to a login page where users can choose identity providers.

When redirected to a default login page by Keycloak, users will be seeing two login options: a credential-based authentication and social login. If a user opts to login using their credentials, Keycloak acts as the identity provider. 

On the other hand, when a user selects a social login option such as signing in with GitHub, Keycloak serves as an identity broker, where the identity provider is now external identity providers. When a user clicks the GitHub option, the browser gets redirected to GitHub's consent page, where the user is prompted to authorize the application to access their data.

Then GitHub's server sends a series of requests to the redirect URIs, including an authorization code via a request (`https://github.com/login/oauth/authorize?...`), and, eventually, an access token through a request (`https://github.com/login?...`), allowing the OAuth2 client (the gateway server) to retrieve user data based on the user's consent.

It's important to note that the redirect URIs are defined by our own configuration, whether as values in the [application.yml](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/2538c8bcbdb6b1ed9ba4ecdeed29efda31a6cf28/backend-for-frontend/src/main/resources/application.yml#L31) , or as environment variables in [Docker Compose](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/2538c8bcbdb6b1ed9ba4ecdeed29efda31a6cf28/manifests/docker-compose.yml#L16), or as in the Kubernetes [manifest](https://github.com/CynicDog/spa-spring-keycloak-oauth2/blob/2538c8bcbdb6b1ed9ba4ecdeed29efda31a6cf28/manifests/backend-for-frontend.yml#L27). 

Since Keycloak has internally implemented the handling for such requests from identity providers, we are all set to proceed to the entry point of our application, whether it's `http://localhost:9000` (Docker Compose) or `http://127.0.0.1/ (Minikube)`. Note that these are the redirect URI values we configured when we registered the `backend-for-frontend` service as a security client to Keycloak server. 

So the browser will navigate to the base URL, we need to come up with a script that checks the authentication status from the gateway server. 

```javascript
export const AuthProvider = ({ children }) => {

    const [isAuthenticated, setIsAuthenticated] = useState(false);
    useEffect(() => {
        const authenticate = async () => {
            try {
                const userData = await getUser();
                if (userData) {
                    setIsAuthenticated(true);
                } else {
                    setIsAuthenticated(false);
                }
            }
        };
        authenticate();
    }, []);

    return (
        <AuthContext.Provider value={{ isAuthenticated, user }}>
            {children}
        </AuthContext.Provider>
    );
};
```
With this context set up, we can differentiate between authenticated and unauthenticated renders in our UI. When the browser serves an authenticated user, it makes a request to the remote server, which we will discuss in the next section.

## 3. Token Relay 
