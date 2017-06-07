---
layout: post
authors: [kevin_van_houtte]
title: 'Securing your cloud-native microservice architecture: Level 1'
image: /img/microservices/CloudSecurity.jpg
tags: [Microservices, Security, OAuth2, JWT, Redis, Session]
category: Microservices
comments: true
---

When developing cloud-native microservices, we have to think of a way how to secure them.
There are a few challenges to face when thinking about distributed systems. 
In these series of posts we will cover and guide you on how to apply the security layer to your cloud-native blueprint.
There are two kind of platforms:
* Platform with one frontend
* Platform with multiple frontends
Both have different insights for security. 
So ask yourself, what kind of platform are you using?
And which security will you need?

# The Overview Wizard


## Our cloud-native architecture
We are using an example architecture, just to visualize how security flows through these components
* User Authentication & Authorization Server: [Spring Cloud Security OAuth2](https://goo.gl/LZdjJO)
* Service Registry: [Spring Cloud Eureka](https://goo.gl/J4pBTi)
* Resilience: [Spring Cloud Hystrix](https://goo.gl/Qqyza8)
* Load Balancer & Routing: [Spring Cloud Zuul](https://goo.gl/qRCQAB)
* Communication client: [Spring Cloud Feign](https://goo.gl/tTUYKT)
* Externalized Config: [Spring Cloud Config Server](https://goo.gl/cKFP88)

## Security Protocol: OAuth2 & JWT
When searching for a security protocol for our architecture, 
we won't reinvent the wheel and looked at what is supported by the Spring framework and most common way in working with first and third party applications.
The OAuth2 delegation protocol can assist us in the creation of scalable microservices,
it will allow us to secure the user's credentials from third party applications and gain access to a microservice through a JWT.
When applying the OAuth2 framework to our architecture, we will be using three grant types to cover our authentication. 
These three grant types are different ways how to obtain a JWT, some clients are more trusted than others. 

### Third party applications: Authorization Code grant type
This grant type is the most commonly used for third party applications,
where user's confidentiality can be maintained.
The user won't have to share his credentials with the application that is requesting resources from our backend. 
This is a redirection-based flow, which means that the application must be capable of interacting with the user's web browser
and receiving API authorization codes that are routed through the user's web browser.

* The frontend(application) makes a request to the UAA server on behalf of the user
* The UAA server pops up a permission window for the user to grant permission, the user authenticates and grants permission
* The UAA server returns an authorization code with a redirection
* The frontend uses the authorization code to exchange a JWT token from the UAA server
* The UAA verifies the authorization code and returns a JWT token

<script async class="speakerdeck-embed" data-id="57b5f3f256a3449b9b3038bc69bf2d5f" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

### First party applications: Password grant type
This grant type is best used for first party applications,
where the user is in a trust relationship with the application.
The application authenticates on behalf of the user and receives the proper JWT.

* The user provides his credentials to the frontend, commonly done with a login form
* The frontend assembles a POST request with the credentials to the UAA server
* The UAA validates the user and returns a valid JWT token



### Trusted Service to Service communication: Client Credentials grant type
The trusted service can request a JWT using only its client id and client secret
when the client is requesting access to the protected resources under its control, or those of another resource owner that have been previously arranged with the authorization server (the method of which is beyond the scope of this specification).
It is very important that the client credentials grant type MUST only be used by confidential clients:

* The gateway authenticates with his app id and secret
* The UAA validates the credentials and returns a valid JWT token


### Obtaining the JWT
When successfully ending one of these grant type flows, we receive a JWT from the UAA server.
JSON Web Tokens (pronounced “jot”) are compact, URL friendly and contains a JSON structure.
The structure is assembled in some standard attributes (called claims), such as issuer, subject (the user’s identity), and expiration time.
There is room available for claims to be customized, allowing additional information to be passed along.
While building your application, you will likely be told that your JWT is not valid for reasons that are not apparent.
For security and privacy reasons, it is not a good idea to expose (to the front end) exactly what is wrong with the login.
Doing so can leak implementation or user details that could be used maliciously.
JWT.IO provides a quick way to verify whether the encoded JWT that scrolled by in your browser console or your trace log is valid or not.

One of the challenges in our microservice architecture is Identity propagation.
After the authentication, the identity of the user needs to be propagated to the next microservice in a trusted way.
When we want to verify the identity, frequent calls back to the UAA server is inefficient, 
especially given that communication between microservices is preferred to routing through the gateway whenever possible to minimize latency.
JSON web tokens is used here to carry along a representation of information about the user.
In essence, a token should be able to:
* Know that the request was initiated from a user request
* Know the identity that the request was made on behalf of
* Know that this request is not a malicious replay of a previous request

#### Dealing with time

When propagating the identity of the user, you don’t want it to last for a infinite of time.
That’s where JWTs come in place, they expire.
This triggers a refresh token of the identity that results in a new JWT.
JWTs have three fields that relate to time and expiry, all of which are optional.
Generally, include these fields and validate them when they are present as follows:
* The time the JWT was created (iat) is before the current time and is valid.
* The “not process before” claim time (nbf) is before the current time and is valid.
* The expiration time (exp) is after the current time and is valid.
All of these times are expressed as UNIX epoch time stamps.

#### Signed JWTs

Signing a JWT helps establish trust between services, as the receiver can then verify the identity of the signer, and the contents of the JWT have not been modified in transit.
JWTs are being signed by a public/private key pair (SSL certificates work well with a well known public key).
Common JWT libraries all support signing.

* The user requests a resource
* The frontend assembles a request with an Authorization header and a Bearer token inside, fires off the request to the gateway
* The gateway verifies the token in communication with the UAA server
* If the token is valid, the gateway redirects the frontend to the correct resource on the proper microservice
* The microservice checks for authorization to the resource, if access granted, the correct resource is returned

### Secure Communication Channel

