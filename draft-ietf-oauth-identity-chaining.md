---
stand_alone: true
ipr: trust200902
cat: info
submissiontype: IETF
area: sec
wg: oauth

docname: draft-ietf-oauth-identity-chaining-latest

title: OAuth Identity and Authorization Chaining Across Domains
abbrev:
lang: en
kw: []
# date: 2022-02-02 -- date is filled in automatically by xml2rfc if not given
author:
- name: Arndt Schwenkschuster
  org: SPIRL
  email: arndts.ietf@gmail.com
- name: Pieter Kasselmann
  org: SPIRL
  email: pieter@spirl.com
- name: Kelley Burgin
  org: MITRE
  email: kburgin@mitre.org
- name: Mike Jenkins
  org: NSA-CCSS
  email: mjjenki@cyber.nsa.gov
- name: Brian Campbell
  org: Ping Identity
  email: bcampbell@pingidentity.com
contributor:
- name: Atul Tulshibagwale
  org: SGNL
  email: atuls@sgnl.ai
- name: George Fletcher
  org: Capital One
  email: george.fletcher@capitalone.com
- name: Rifaat Shekh-Yusef
  org: EY
  email: rifaat.shekh-yusef@ca.ey.com
- name: Hannes Tschofenig
  email: hannes.tschofenig@gmx.net

normative:
  RFC6749: # OAuth 2.0 Authorization Framework
  RFC8693: # OAuth 2.0 Token Exchange
  RFC7523: # JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants
  RFC8707: # Resource Indicators for OAuth 2.0
  RFC8414: # OAuth 2.0 Authorization Server Metadata
  RFC9449: # OAuth 2.0 Demonstrating Proof of Possession (DPoP)
  RFC8705: # OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens

informative:

  I-D.ietf-oauth-security-topics:
  I-D.ietf-oauth-resource-metadata:


--- abstract

This specification defines a mechanism to preserve identity and authorization information across trust domains that use the OAuth 2.0 Framework.

--- middle

# Introduction
Applications often require access to resources that are distributed across multiple trust domains where each trust domain has its own OAuth 2.0 authorization server. As a result, developers are often faced with the situation that a protected resource is located in a different trust domain and thus protected by a different authorization server. A request may transverse multiple resource servers in multiple trust domains before completing. All protected resources involved in such a request need to know on whose behalf the request was originally initiated (i.e. the user), what authorization was granted and optionally which other resource servers were called prior to making an authorization decision. This information needs to be preserved, even when a request crosses one or more trust domains. This document refers to this as "chaining" and defines a mechanism for preserving identity and authorization information across domains using a combination of OAuth 2.0 Token Exchange {{RFC8693}} and JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants {{RFC7523}}.

## Requirements Language

{::boilerplate bcp14-tagged}

# Identity and Authorization Chaining Across Domains

This specification describes a combination of OAuth 2.0 Token Exchange {{RFC8693}} and JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants {{RFC7523}} to achieve identity and authorization chaining across domains.

A client in trust domain A that needs to access a resource server in trust domain B requests a JWT authorization grant from the authorization server for trust domain A via a token exchange. The client in trust domain A presents the received grant as an assertion to the authorization server in domain B in order to obtain an access token for the protected resource in domain B. The client in domain A may be a resource server, or it may be the authorization server itself.

## Overview

The identity and authorization chaining flow outlined below describes how a combination of OAuth 2.0 Token Exchange {{RFC8693}} and JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants {{RFC7523}} are used to address the use cases identified. The (Appendix, (#Examples)) include two additional examples that describe how this flow is used when the resource server acts as the client or the authorization server acts as the client.

~~~~
+-------------+                            +-------------+ +---------+
|Authorization|         +--------+         |Authorization| |Protected|
|Server       |         |Client  |         |Server       | |Resource |
|Domain A     |         |Domain A|         |Domain B     | |Domain B |
+-------------+         +--------+         +-------------+ +---------+
       |                    |                     |             |
       |                    |----+                |             |
       |                    |    | (A) discover   |             |
       |                    |<---+ Authorization  |             |
       |                    |      Server         |             |
       |                    |      Domain B       |             |
       |                    |                     |             |
       |                    |                     |             |
       | (B) exchange token |                     |             |
       |   [RFC 8693]       |                     |             |
       |<-------------------|                     |             |
       |                    |                     |             |
       | (C) <authorization |                     |             |
       |       grant JWT>   |                     |             |
       | - - - - - - - - - >|                     |             |
       |                    |                     |             |
       |                    | (D) present         |             |
       |                    | authorization grant |             |
       |                    | [RFC 7523]          |             |
       |                    | ------------------->|             |
       |                    |                     |             |
       |                    | (E) <access token>  |             |
       |                    | <- - - - - - - - - -|             |
       |                    |                     |             |
       |                    |               (F) access          |
       |                    | --------------------------------->|
       |                    |                     |             |
       |                    |                     |             |
~~~~
{: title='Identity and Authorization Chaining Flow'}

The flow illustrated in Figure 1 shows the steps the client in trust Domain A needs to perform to access a protected resource in trust domain B. In this flow, the client is in posession of a token that an authorization server will accept as part of a token exchange flow as defined in [Token Exchange](#token-exchange). How the client obtained this token is out of scope of this specification. The client has a way to discover the authorization server in Domain B and a trust relationship exists between Domain A and Domain B (e.g., through federation). It includes the following:

* (A) The client of Domain A needs to discover the authorization server of Domain B. See [Authorization Server Discovery](#authorization-server-discovery).

* (B) The client exchanges a token it has in its posession at the authorization server of its own domain (Domain A) for a JWT authorization grant that can be used at the authorization server in Domain B. See [Token Exchange](#token-exchange).

* (C) The authorization server of Domain A processes the request and returns a JWT authorization grant that the client can use with the authorization server of Domain B. This requires a trust relationship between Domain A and Domain B (e.g., through federation).

* (D) The client presents the authorization grant to the authorization server of Domain B. See [Access Token Request](#access-token-request).

* (E) Authorization server of Domain B validates the JWT authorization grant and returns an access token.

* (F) The client in Domain A uses the access token received from the authorization server in Domain B to access the protected resource in Domain B.

## Authorization Server Discovery
This specification does not define authorization server discovery. A client MAY maintain a static mapping or use other means to identify the authorization server. The `authorization_servers` property in {{I-D.ietf-oauth-resource-metadata}} MAY be used.

## Token Exchange

The client performs token exchange as defined in {{RFC8693}} with the authorization server for its own domain (e.g., Domain A) in order to obtain a JWT authorization grant that can be used with the authorization server of a different domain (e.g., Domain B) as specified in section 1.3 of {{RFC6749}}.

### Token Exchange Request

The parameters described in section 2.1 of {{RFC8693}} apply here with the following restrictions:

{:vspace}
scope
: OPTIONAL. Additional scopes to indicate scopes included in the returned JWT authorization grant. See [Claims transcription](#claims-transcription).

resource
: REQUIRED if audience is not set. URI of authorization server of targeting domain (domain B).

audience
: REQUIRED if resource is not set. Well known/logical name of authorization server of targeting domain (domain B).

### Processing rules

* If the request itself is not valid or if the given resource or audience are unknown, or are unacceptable based on policy, the authorization server MUST deny the request.
* The authorization server MAY add, remove or change claims. See [Claims transcription](#claims-transcription).

### Token Exchange Response

All of section 2.2 of {{RFC8693}} applies. In addition, the following applies to implementations that conform to this specification.

* The "aud" claim in the returned JWT authorization grant MUST identify the requested authorization server. This corresponds with [RFC 7523 Section 3, Point 3](https://datatracker.ietf.org/doc/html/rfc7523#section-3) and is there to reduce misuse and to prevent clients from presenting access tokens as an authorization grant to an authorization server in a different domain.

* The "aud" claim included in the returned JWT authorization grant MAY identify multiple authorization servers, provided that trust relationships exist with them (e.g. through federation). It is RECOMMENDED that the "aud" claim is restricted to a single authorization server to prevent an authorization server in one domain from presenting the client's authorization grant to an authorization server in a different trust domain. For example, this will prevent the authorization server in Domain B from presenting the authorization grant it received from the client in Domain A to the authorization server for Domain C.

### Example

The example below shows the message invoked by the client in trust domain A to perform token exchange with the authorization server in domain A (https://as.a.org/auth) to receive a JWT authorization grant for the authorization server in trust domain B (https://as.b.org/auth).

~~~
POST /auth/token HTTP/1.1
Host: as.a.org
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&resource=https%3A%2F%2Fas.b.org%2Fauth
&subject_token=ey...
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
~~~
{: title='Token exchange request'}

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
  "access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJo
  dHRwczovL2FzLmEub3JnL2F1dGgiLCJleHAiOjE2OTUyODQwOTIsImlhdCI6MTY5N
  TI4NzY5Miwic3ViIjoiam9obl9kb2VAYS5vcmciLCJhdWQiOiJodHRwczovL2FzLm
  Iub3JnL2F1dGgifQ.304Pv9e6PnzcQPzz14z-k2ZyZvDtP5WIRkYPScwdHW4",
  "token_type":"N_A",
  "issued_token_type":"urn:ietf:params:oauth:token-type:jwt",
  "expires_in":60
}
~~~
{: title='Token exchange response'}

## JWT Authorization Grant

The client presents the JWT authorization grant it received from the authorization server in its own domain as an authorization grant to the authorization server in the domain of the resource server it wants to access as defined in {{RFC7523}}.

### Access Token Request

The authorization grant is a JWT bearer token, which the client uses to request an access token as described in the JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants {{RFC7523}}. For the purpose of this specification the following descriptions apply:

{:vspace}
grant_type
: REQUIRED. As defined in Section 2.1 of {{RFC7523}} the value `urn:ietf:params:oauth:grant-type:jwt-bearer` indicates the request is a JWT bearer assertion authorization grant.

assertion
: REQUIRED. Authorization grant returned by the token exchange (`access_token` response).

scope
: OPTIONAL.

The client MAY indicate the audience it is trying to access through the `scope` parameter or the `resource` parameter defined in {{RFC8707}}.

### Processing rules

The authorization server MUST validate the JWT authorization grant as specified in Sections 3 and 3.1 of {{RFC7523}}. The following processing rules also apply:

* The "aud" claim MUST identify the Authorization Server as a valid intended audience of the assertion using either the token endpoint as described Section 3 {{RFC7523}} or the issuer identifier as defined in Section 2 of {{RFC8414}}.
* The authorization server SHOULD deny the request if it is not able to identify the subject.
* Due to policy the request MAY be denied (for instance if the federation from domain A is not allowed).

### Access Token Response

The authorization server responds with an access token as described in section 5.1 of {{RFC6749}}.

### Example

The example belows shows how the client in trust domain A presents an authorization grant to the authorization server in trust domain B (https://as.b.org/auth) to receive an access token for a protected resource in trust domain B.

~~~
POST /auth/token HTTP/1.1
Host: as.b.org
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=ey...
~~~
{: title='Assertion request'}

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
  "access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJo
  dHRwczovL2FzLmIub3JnL2F1dGgiLCJleHAiOjE2OTUyODQwOTIsImlhdCI6MTY5N
  TI4NzY5Miwic3ViIjoiam9obi5kb2UuMTIzIiwiYXVkIjoiaHR0cHM6Ly9iLm9yZy
  9hcGkifQ.CJBuv6sr6Snj9in5T8f7g1uB61Ql8btJiR0IXv5oeJg",
  "token_type":"Bearer",
  "expires_in":60
}
~~~
{: title='Assertion response'}

## Claims transcription

Authorization servers MAY transcribe claims when either producing JWT authorization grants in the token exchange flow or access tokens in the assertion flow.

* **Transcribing the subject identifier**: Subject identifier can differ between the parties involved. For instance: A user is known at domain A by "johndoe@a.org" but in domain B by "doe.john@b.org". The mapping from one identifier to the other MAY either happen in the token exchange step and the updated identifier is reflected in returned JWT authorization grant or in the assertion step where the updated identifier would be reflected in the access token. To support this both authorization servers MAY add, change or remove claims as described above.
* **Selective disclosure**: Authorization servers MAY remove or hide certain claims due to privacy requirements or reduced trust towards the targeting trust domain.
* **Controlling scope**: Clients MAY use the scope parameter to control transcribed claims (e.g. downscoping). Authorization Servers SHOULD verify that the requested scopes are not higher privileged than the scopes of the presented subject_token.
* **Including JWT authorization grant claims**: The authorization server performing the assertion flow MAY leverage claims from the presented JWT authorization grant and include them in the returned access token. The populated claims SHOULD be namespaced or validated to prevent the injection of invalid claims.

The representation of transcribed claims and their format is not defined in this specification.

# IANA Considerations {#IANA}

To be added.

# Security Considerations {#Security}

## Client Authentication
Authorization Servers SHOULD follow the OAuth 2.0 Security Best Current Practice {{I-D.ietf-oauth-security-topics}} for client authentication.

## Sender Constraining Tokens
Access tokens SHOULD be sender constrained using mechanisms such as OAuth Demonstration of Proof of Possession (DPoP) {{RFC9449}} or Mutual TLS for OAuth 2.0 {{RFC8705}} to minimise the risks of token theft and replay.

# Use cases

This sections outlines some use cases where the identity and authorization chaining described in this document can be applied. The use cases described are not exhaustive, but are representative of the type of use cases enabled by this specification. Other use cases may also be supported by this specification.

## Preserve User Context across Multi-cloud, Multi-Hybrid environments
A user attempts to access a service that is implemented as a number of on-premise and cloud-based microservices. Both the on-premise and cloud-based services are segmented by multiple trust boundaries that span one or more on-premise or cloud service environments. Every microservice can apply an authorization policy that takes the context of the original user, as well as intermediary microservices into account, irrespective of where the microservices are running and even when a microservice in one trust domain calls another service in another trust domain.

## Continuous Integration Accessing External Resources
A continuous integration system needs to access external resources, for example to upload an artifact or to run tests. These resources are protected by different authorization servers. The identity information of the build, for example metadata such as commit hashes or repository, should be preserved and carried across the domain boundary. This not just prevents maintaining credentials it also allows fine grained access control at the resource.

## API Security Use Case
A home devices company provides a "Camera API" to enable access to home cameras. Partner companies use this Camera API to integrate the camera feeds into their security dashboards. Using OAuth between the partner and the Camera API, a partner can request the feed from a home camera to be displayed in their dashboard. The user has an account with the camera provider. The user may be logged in to view the partner provided dashboard, or they may authorize emergency access to the camera. The home devices company must be able to independently verify that the request originated and was authorized by a user who is authorized to view the feed of the requested home camera.

## Extend Single-Sign-On to API Access
A user that authenticated to an enterprise Identity Provider (IdP) does not have to sign-in to multiple SaaS applications if the SaaS applications are configured to trust the enteprise IdP. It is possible to extend this SSO relationship to API access by allowing the Client to contact the enterprise IdP and exchange the identity assertion (ID Token or SAML Token) that it previously received from the enteprise IdP for an authorization grant. The authorization grant can be used to obtain an access token from the SaaS application's authorization server, provided that a trust relationship has been established between the enterprise IdP which issues the authorization grant and the SaaS authorization server. As a result SaaS servers that trust the enterprise IdP do not require the user to complete an interactive delegated OAuth 2.0 flow to obtain an access token to access the SaaS provider's APIs.

## Cross-domain API authorization
An e-mail client can be used with arbitrary email servers, without requiring pre-established relationships between each email client and each email server. An e-mail client obtains an identity assertion (ID Token or SAML token) from an IdP. When the e-mail client needs access to a separate API, such as a third-party calendaring application, the email client exchanges the identity assertion for an authorization grant and uses this authorization grant to obtain an access token for the third-party calendaring application from the authorization server trusted by the third-party calendaring application. If the authorization server trusts the issuer of the authorization grant, the e-mail client obtains an access token without any additional user interaction.

# Examples {#Examples}

This section contains two examples, demonstrating how this specification may be used in different environments with specific requirements. The first example shows the resource server acting as the client and the second example shows the authorization server acting as the client.

## Resource server acting as client

As part of completing a request, a resource server in Domain A may need to access a resource server in Domain B. This requires the resource server in Domain A to obtain an Access Token from an authorization server in Domain B, which it may then be presented to the resource server in Domain B. A Resource server in Domain B may use the flows described in this specification by assuming the role of a client when attempting to access the resource server in Domain B. Resources servers may act as clients if the following is true:

* The resource server has the ability to determine the authorization server of the protected resource outside its trust domain.
* Authorization Server B is reachable by the resource server and is able to perform the appropriate client authentication (if required).

The flow would look like this:

~~~
+-------------+       +---------------+     +-------------+ +---------+
|Authorization|       |Resource Server|     |Authorization| |Protected|
|Server       |       |Domain A       |     |Server       | |Resource |
|Domain A     |       |(acting as     |     |Domain B     | |Domain B |
|             |       | Client)       |     |             | |         |
+-------------+       +---------------+     +-------------+ +---------+
       |                     |                     |             |
       |                     |   (A) request protected resource  |
       |                     |      metadata                     |
       |                     | --------------------------------->|
       |                     | <- - - - - - - - - - - - - - - - -|
       |                     |                     |             |
       | (B) exchange token  |                     |             |
       |   [RFC 8693]        |                     |             |
       |<--------------------|                     |             |
       |                     |                     |             |
       | (C) <authorization  |                     |             |
       |        grant>       |                     |             |
       | - - - - - - - - -  >|                     |             |
       |                     |                     |             |
       |                     | (D) present         |             |
       |                     |  authorization      |             |
       |                     |  grant [RFC 7523]   |             |
       |                     |-------------------->|             |
       |                     |                     |             |
       |                     | (E) <access token>  |             |
       |                     |<- - - - - - - - - - |             |
       |                     |                     |             |
       |                     |               (F) access          |
       |                     | --------------------------------->|
       |                     |                     |             |
       |                     |                     |             |
~~~
{: title='Resource server acting as client'}

The flow contains the following steps:

(A) The resource server of Domain A needs to access protected resource in Domain B. It requires an access token to do so which it does not possess. In this example {{I-D.ietf-oauth-resource-metadata}} is used to receive information about the authorization server which protects the resource in domain B. This step MAY be skipped if discovery is not needed and other means of discovery MAY be used. The protected resource returns its metadata along with the authorization server information.

(B) Now, after the resource server has identified the authorization server for Domain B, the resource server requests a JWT authorization grant for the authorization server in Domain B from its own authorization server (Domain A). This happens via the token exchange protocol.

(C) If successful, the authorization server returns a JWT authorization grant to the resource server.

(D) The resource server presents the JWT authorization grant to the authorization server of Domain B.

(E) The authorization server of Domain B uses claims from the JWT authorization grant to identify the user and its access. If access is granted an access token is returned.

(F) The resource server uses the access token to access the protected resource at Domain B.

## Authorization server acting as client

Authorization servers may act as clients too. This can be necessary because of following reasons:

* Clients in Domain A may not have knowledge of authorization servers in Domain B.
* Clients in domain A may not have network access to other authorization servers in Domain B.
* Strict access control on resources outside the trust domain is required and enforced by authorization servers in Domain B.
* Authorization servers in domain B require client authentication, but are unable to manage clients outside of Domain B.

Under these conditions, an authorization server in Domain A may obtain an Access Token from an authorization server in Domain B on-behalf-of any client in Domain A. This enables clients in Domain A to access a protected resource server in Domain B. Resource servers in domain A may act as a client to the authorization server in Domain A in order to obtain an access token to access a protected resource in Domain B in order to complete a request.

The authorization server may use the flows described in this specification by acting first as a client to itself to obtain an assertion grant and then act as a client to the authorization server in domain B to request an access token for protected resources in Domain B. The flow when authorization servers act as a client on-behalf of another client in it's own trust domain would look like this:

~~~
+--------+          +-------------+         +-------------+ +---------+
|Client  |          |Authorization|         |Authorization| |Protected|
|        |          |Server       |         |Server       | |Resource |
|Domain A|          |Domain A     |         |Domain B     | |Domain B |
|        |          |(acting as   |         |             | |         |
|        |          |client)      |         |             | |         |
+--------+          +-------------+         +-------------+ +---------+
    |                      |                       |             |
    | (A) request or       |                       |             |
    | exchange token for   |                       |             |
    | protected resource   |                       |             |
    | in domain B.         |                       |             |
    | -------------------->|                       |             |
    |                      |                       |             |
    |                      |----+                  |             |
    |                      |    | (B) determine    |             |
    |                      |<---+ authorization    |             |
    |                      |      server B         |             |
    |                      |                       |             |
    |                      |                       |             |
    |                      |----+                  |             |
    |                      |    | (C) issue        |             |
    |                      |<---+ authorization    |             |
    |                      |      grant ("internal |             |
    |                      |      token exchange") |             |
    |                      |                       |             |
    |                      |                       |             |
    |                      | (D) present           |             |
    |                      |   authorization grant |             |
    |                      |   [RFC 7523]          |             |
    |                      | --------------------->|             |
    |                      |                       |             |
    |                      | (E) <access token>    |             |
    |                      | <- - - - - - - - - - -|             |
    |                      |                       |             |
    |  (F) <access token>  |                       |             |
    | <- - - - - - - - - - |                       |             |
    |                      |                       |             |
    |                      |           (G) access  |             |
    | ---------------------------------------------------------->|
    |                      |                       |             |
    |                      |                       |             |
~~~
{: title='Authorization server acting as client'}

The flow contains the following steps:

(A) The client in Domain A requests a token for the protected resource in Domain B from the authorization server in Domain A. This specification does not define this step. A profile of Token Exchange {{RFC8693}} may be used.

(B) The authorization server of Domain A determines the authorization server of Domain B. This could have been passed by the client, is statically maintained or dynamically resolved.

(C) Once the authorization server in Domain B is determined, the authorization server in Domain A issues a JWT authorization grant to itself. This reflects to [Token exchange](#token-exchange) of this specification and can be seen as an "internal token exchange".

(D) The authorization server in Domain A acts as a client and presents the JWT authorization grant to the authorization server of Domain B. This presentation happens between the authorization servers. Authorization server A may be required to perform client authentication while doing so. This reflects to [See [Access Token Request](#access-token-request)] of this specification.

(E) The authorization server of Domain B returns an access token for the protected resource in Domain B to the authorization server in Domain A.

(F) The authorization server of Domain A returns the access token to the client in Domain A.

(G) The client in Domain A uses the received access token to access the protected resource in Domain B.

# Acknowledgements {#Acknowledgements}
The editors would like to thank Joe Jubinski, Justin Richer, Aaron Parecki, Dean H. Saxe, and others (please let us know, if you've been mistakenly omitted) for their valuable input, feedback and general support of this work.

# Document History

\[\[ To be removed from the final specification ]]
-latest
* Clarified diagrams and description of authorization server acting as a client.
* Remove references to sd-jwt.

-03

* Editorial updates

-02

* remove recommendation to not use RFC8693's requested_token_type
* Corrected discrepancy between alphabetic numbering of the diagram and text in the resource acting as client example


-01

* limit the authorization grant format to RFC7523 JWT
* minor example fixes
* editorial fixes
* added Aaron Parecki to acknowledgements
* renamed section headers to be more explicit
* use more specific term "JWT authorization grant"
* changed name to "OAuth Identity and Authorization Chaining Across Domains"
* move use cases to appendix and add continuous integration use case

-00

* initial working group version (previously draft-schwenkschuster-oauth-identity-chaining)
