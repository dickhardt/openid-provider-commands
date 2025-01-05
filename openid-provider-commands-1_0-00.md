%%%
title = "OpenID Provider Commands - draft 00"
abbrev = "openid-provider-commands"
ipr = "none"
workgroup = "OpenID Connect"
keyword = ["security", "openid", "lifecycle"]

[seriesInfo]
name = "Internet-Draft"
value = "openid-connect-commands-1_0-10"
status = "standard"

[[author]]
initials="D."
surname="Hardt"
fullname="Dick Hardt"
organization="Hellō"
    [author.address]
    email = "dick.hardt@gmail.com"

[[author]]  
initials="K."
surname="McGuinness"
fullname="Karl McGuinness"
organization="Independent"
    [author.address]
    email = "me@karlmcguinness.com"

%%%

.# Abstract

OpenID Connect defines a protocol for an end-user to use an OpenID Provider (OP) to log in to a Relying Party (RP) and assert claims about the end-user using an ID Token. RPs will often use the identity claims about the user to implicitly (or explicitly) establish an account for the user at the RP

OpenID Provider Commands complements OpenID Connect by introducing a set of commands for an OP to directly manage an end-user account at an RP. These commands enable an OP to activate, maintain, suspend, reactivate, archive, restore, delete, and unauthorize an end-user account. Command tokens simplify RP adoption by re-use of the OpenID Connect ID Token schema, security, and verification mechanisms.

{mainmatter}


# Introduction

> **FAQ for early reviewers**
> 
> 
>**1. How does SCIM compare to OpenID Provider (OP) Commands?**
>
> The SCIM protocol is a general purpose protocol for a client to manage resources at a server. When the SCIM protocol is used between an IdP and an RP, the schema is defined by the RP. The resources managed are in the context of the RP tenant in a multi-tenant RP. Any extensions to the schema are defined by the RP. This provided an interoperable protocol to manage RP resources
> OP Commands are an extension of a user account created by OpenID Connect. It uses the same identity claims that the OP issues for the user. It uses the same token claims, and is verified the same way. OP Commands are issued in the context of the OP tenant in a multi-tenant OP.
>
> 
>**2. How do Shared Signals / RISC compare to OP Commands?**
>
> Shared Signals and RISC are events that one party is sharing with another party. The actions a receiving party takes upon receiving a signal are intentionally not defined. 
> The actions taken by the RP when receiving an OP Command is specified. This gives an OP control over the account at the RP.
>
> 
>**3. Are OP Commands a replacement for SCIM, Shared Signals, or RISC?**
>
> No. These standards are deployed by organizations that have complex requirements, and these standards meet there needs. Most OP / RPs do not deploy any of these standards, as the implementation complexity is not warranted. OP Commands are designed to build on OpenID Connect, allowing RPs using OpenID Connect an easy path to offer OPs a standard API for security and lifecycle operations. 
>

OpenID Connect 1.0 (OIDC) is a widely adopted identity protocol that enables client applications, known as relying parties (RPs), to verify the identity of end-users based on authentication performed by a trusted service, the OpenID Provider (OP). OIDC also provides mechanisms for securely obtaining identity attributes, or claims, about the end-user, which helps RPs tailor experiences and manage access with confidence.

OIDC not only allows an end-user to log in and authorize access to an RP but also facilitates creating an account with the RP. However, account creation is only the beginning of an account's lifecycle. Throughout the lifecycle, various actions may be required to ensure data integrity, security, and regulatory compliance.

For example, many jurisdictions grant end-users the "right to be forgotten," enabling them to request the deletion of their accounts and associated data. When such requests arise, OPs may need to notify RPs to fully delete the end-user's account and remove all related data, respecting both regulatory obligations and end-user privacy preferences.

In scenarios where malicious activity is detected or suspected, OPs play a vital role in protecting end-users. They may need to instruct RPs to revoke authorization or delete accounts created by malicious actors. This helps contain the impact of unauthorized actions and prevent further misuse of compromised accounts.

In enterprise environments, where organizations centrally manage workforce access, OPs handle essential account operations across various stages of the lifecycle. These operations include activating, maintaining, suspending, reactivating, archiving, restoring, and deleting accounts to maintain security and compliance.

OpenID Provider Commands enable OPs to manage these account lifecycle stages directly with RPs, building upon the existing OP / RP relationship to cover the full spectrum of account management needs.


## Requirements Notation and Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}.

In the .txt version of this specification,
values are quoted to indicate that they are to be taken literally.
When using these values in protocol messages,
the quotes MUST NOT be used as part of the value.
In the HTML version of this specification,
values to be taken literally are indicated by
the use of *this fixed-width font*.

## Terminology

This specification defines the following terms:

- **Command**: An instruction from an OP to an RP. It is a synchronous request and response.

- **Command Token**: A JSON Web Token (JWT) signed by the OP that contains Claims about the Command being issued.

- **Commands URI**: The URL at the RP where OPs post Command Tokens.

- **Organization**: A register of accounts belonging to one entity. If an OP supports multiple Organizations with the same `iss` claim, the OP MUST include the `org` claim in Commands to identify the Organization.

## Protocol Overview

This specification defines a Command request containing a Command Token sent from the OP to the RP, and a Command response returned from the RP to the OP.

```
+------+  Command request       +------+
|      |---- Command Token ---->|      |
|  OP  |                        |  RP  | 
|      |<-----------------------|      |
+------+      Command response  +------+
```

## Command Usage Overview

An OP will typically send a **describe** Command at the start of the relationship with an RP relationship, and then periodically, to learn the other Commands an RP supports. If the OP represents multiple Organizations, the OP will send a **describe** command for each Organization. The OP may use the **describe** Command response to determine if the RP supports functionality required by the Organization before issuing ID Tokens or **activate** Commands to the RP.

If the RP supports the **groups** Command, the OP will typically send the **groups** Command at the start of the Organization and RP relationship, and then whenever the groups change.

If the RP supports any Lifecycle Commands, the OP will send supported Lifecycle Commands to synchronize the state of accounts at the RP with the state at the Organization. If the RP supports Lifecycle Commands, the RP should also support the **audit** Command. The OP will typically send an **audit** Command at the start of the Organization and RP relationship, and then periodically, to learn the state of the Organization's accounts at the RP and correct any drift between the account state at the Organization and the RP.

If the RP supports the **unauthorize** Command, the OP will send the **unauthorize** Command if the OP suspects an account has been taken over by a malicious actor.

# Command Request

The OP uses an HTTP POST to the registered Commands URI
to send account commands to the RP. The POST body uses the
`application/x-www-form-urlencoded` encoding
and must include a `command_token` parameter
containing a Command Token from the OP for the RP.

The POST body MAY contain other values in addition to
`command_token`.
Values that are not understood by the implementation MUST be ignored.

The following is a non-normative example of such a Command request
(with most of the Command Token contents omitted for brevity):

```
POST /commands HTTP/1.1
Host: rp.example.net
Content-Type: application/x-www-form-urlencoded

command_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```

# Command Response

If the Command succeeded, the RP MUST respond with HTTP 200 OK.
However, note that some Web frameworks will substitute an HTTP 204 No Content response
for an HTTP 200 OK when the HTTP body is empty.
Therefore, OPs should be prepared to also process an HTTP 204 No Content response as a successful response.


The RP's response MUST include the Cache-Control HTTP response header field with a no-store value, keeping the response from being cached to prevent cached responses from interfering with future Command requests. An example of this is:

```
Cache-Control: no-store
```

If there is a response body, it MUST be JSON and use the `application/json` media type.

If the request is not valid, the RP MUST return an `error` parameter, and may include a `error_description` parameter. 
Note that the information conveyed in an error response is intended to help debug deployments;
it is not intended that implementations use different `error` values
to trigger different runtime behaviors.


## Invalid Request Error

If there was a problem with the syntax of the Command request, or the Command Token was invalid, the RP MUST return an HTTP 400 Bad Request and include the `error` parameter with a value of `invalid_request`. 

## Unsupported Command Error

If the RP does not support the Command requested, the RP MUST return an HTTP 400 Bad Request and include the `error` parameter with the value of `unsupported_command`.

Note that the RP may support Commands for some OPs, and not others, and for some Organizations, and not others.

## Server Error

If the RP is unable to process a valid request, the RP MUST respond with a 5xx Server Error status code as defined in RFC 9110 section 15.6.

# Command Token

OPs send a JWT similar to an ID Token to RPs called a Command Token
to issue Commands. ID Tokens are defined in Section 2 of {{OpenID.Core}}.

The following Claims are used within the Command Token:

- **iss**  
  REQUIRED.  
  Issuer Identifier, as specified in Section 2 of {{OpenID.Core}}.

- **sub**  
  REQUIRED except for the **describe**, **groups**, **audit** Commands. 

  PROHIBITED for the **describe**, **groups**, **audit** Commands. 

  Subject Identifier, as specified in Section 2 of {{OpenID.Core}}. 
  
  The combination of the `iss` and `sub` claim uniquely identifies the account for the command.

- **aud**  
  REQUIRED.  
  Audience(s), as specified in Section 2 of {{OpenID.Core}}.

- **iat**  
  REQUIRED.  
  Issued at time, as specified in Section 2 of {{OpenID.Core}}.

- **exp**  
  REQUIRED.  
  Expiration time, as specified in Section 2 of {{OpenID.Core}}.

- **jti**  
  REQUIRED.  
  Unique identifier for the token, as specified in Section 9 of {{OpenID.Core}}.

- **command**  
  REQUIRED.  
  A JSON string. The command for the RP to execute. See [Commands](#commands) for standard values defined in this document. Other specifications may define additional values.

- **org**
  REQUIRED for OPs that support multiple tenants for the same `iss`.
  The Organization the user belongs to. It is a JSON object containing:
    - **id** 
    REQUIRED.
    A JSON string that is a stable OP unique identifier for the Organization.

    - **domain**
    OPTIONAL
    A JSON string for a domain name that the OP has verified the Organization controls. The **domain** MUST not be used as a persistent identifier by the RP for the Organization. The **domain** MAY be used to link Organization data.

- **groups**
  OPTIONAL for the **activate** and **maintain** lifecycle commands.

  The **groups** claim is a JSON array of stable OP unique identifiers for the groups that the user belongs to. The OP SHOULD include the **group** claim if the OP has not send a **groups** Command.

> Author NOTE
>
> The **groups** claim is registered in [IANA JSON Web Token Claims](https://www.iana.org/assignments/jwt/jwt.xhtml), which refers to [RFC 7643 4.1.2](https://www.rfc-editor.org/rfc/rfc7643.html#section-4.1.2) and [RFC 9068 2.2.3.1](https://www.rfc-editor.org/rfc/rfc9068.html#name-claims-for-authorization-ou)
> Besides defining that it is a list, none of the specifications define the properties of a group. To enable interop, this document defines the contents of the groups claim. As there is a **groups** command to provide the mapping between group identifiers and group display names, the **groups** claim only include the group identifiers.


The following Claim MUST NOT be used within the Command Token:

- **nonce**  
  PROHIBITED.  
  A `nonce` Claim MUST NOT be present.
  Its use is prohibited to prevent misuse of the Command Token.

Command Tokens MAY contain other Claims.
Any Claims used that are not understood MUST be ignored.

A Command Token MUST be signed.
The same keys are used to sign Command Tokens
as are used for ID Tokens.


Command Tokens MUST be explicitly typed.
This is accomplished by including a `typ` (type) Header Parameter
with a value of `command+jwt` in the Command Token.
See [Security Considerations](#security-considerations) for a discussion of the security and interoperability considerations
of using explicit typing.

A non-normative example JWT Claims Set for the Command Token for an **activate** command follows:

```json
{
  "iss": "https://op.example.org",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "iat": 1734003000,
  "exp": 1734003060,
  "jti": "bWJq",
  "command": "activate",
  "given_name": "Jane",
  "family_name": "Smith",
  "email": "jane.smith@example.org",
  "email_verified": true,
  "org": {
    "id": "ff6e7c96-7309-4f96-8a3b-39a8c12337b0",
    "domain": "example.com"
  },
  "groups": [
    "b0f4861d-f3d6-4f76-be2f-e467daddc6f6",
    "88799417-c72f-48fc-9e63-f012d8822ad1",
  ]
}
```

A non-normative example JWT Claims Set for the Command Token for an **unauthorize** command follows:

```json
{
  "iss": "https://op.example.org",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "iat": 1734004000,
  "exp": 1734004060,
  "jti": "bWJr",
  "command": "unauthorize",
}
```


# General Commands

The RP MUST support the **describe** command. Support for other general commands is OPTIONAL. 


## **describe** Command

The OP sends this command to the RP to learn what commands the RP supports, and other metadata the RP chooses to share. 

Following is a non-normative example of a *describe* Command Token payload:

```json
{
  "iss": "https://op.example.org",
  "aud": "s6BhdRkqt3",
  "iat": 1734002000,
  "exp": 1734002060,
  "jti": "bWJp",
  "command": "describe",
  "org": {
    "id": "73849284748493"
  },
```


### **describe** Response

If the Command Token is valid, the RP responds with an `application/json` media type that MUST include:

- **context**: a JSON object.
  - **iss**
  REQUIRED. the **iss** value from the Command Token.
  - **org**
  REQUIRED if the Command Token contained an **org** claim, then the **id** value from the **org** claim must be included an  **org** JSON object

- **commands_supported**: a JSON array of commands the RP supports. The **describe** value MUST be included.
- **commands_uri**: the RPs Commands URI. This is the URL the Command Token was sent to.
- **describe_ttl**: the time in seconds the **describe** Command response is valid for.
- **client_id**: the `client_id` for the RP.


The response MAY also include any OAuth Dynamic Client Registration Metadata *TBD [IANA reference](https://www.iana.org/assignments/oauth-parameters/oauth-parameters.xhtml#client-metadata)*

The RP MAY provide a different response for different **iss** claim values, and for different **org** claim values.

Following is a non-normative example of a **describe** response to the example request payload in {{describe-command}}:

```json
{
  "context": {
    "iss": "https://op.example.org",
    "org": {
      "id": "73849284748493"
    },
  },
  "commands_uri": "https://rp.example.net/commands",
  "commands_supported":[
    "describe",
    "unauthorize",
    "suspend",
    "reactivate",
    "delete",
    "audit"
  ],
  "commands_ttl": 86400,
  "client_id": "s6BhdRkqt3",
  "client_name": "Example RP",
  "logo_uri": "https://rp.example.net/logo.png",
  "policy_uri": "https://rp.example.net/privacy-policy.html",
  "tos_uri": "https://rp.example.net/terms-of-service.html",
  "jwks_uri": "https://rp.example.net/jwks",
  "initiate_login_uri": "https://rp.example.net/initiate-login",
  "redirect_uris": [
    "https://rp.example.net/response"
  ]
}
```

## **groups** Command
The OP sends this command to inform the RP of the complete set of possible **groups** claims the OP MAY include in ID and Command Tokens. 

The **groups** claim is REQUIRED.
The **org** claim is REQUIRED when the command is sent from a multi-tenant OP.

A **groups** command overrides the set of possible **group** claims any previous **groups** command provided. 

Following is a non-normative example of a **groups** Command Token payload:

```json
{
  "iss": "https://op.example.org",
  "aud": "s6BhdRkqt3",
  "iat": 1734006000,
  "exp": 1734006060,
  "jti": "bWJt",
  "command": "groups",
  "org": {
    "id": "ff6e7c96-7309-4f96-8a3b-39a8c12337b0",
    "domain": "example.com"
  },
  "groups": [
    {
      "id": "b0f4861d-f3d6-4f76-be2f-e467daddc6f6",
      "display": "Administrators"
    },
    {
      "id": "88799417-c72f-48fc-9e63-f012d8822ad1",
      "display": "Staff"
    }
  ]
}
```

## **unauthorize** Command
The OP sends this command when it suspects a previous OpenID Connect ID Token issued by the OP was granted to a malicious actor. The RP MUST revoke all active sessions and MUST reverse all authorization that may have been granted to applications, including `offline_access`, for account resources identified by the **sub**.

The account MUST be in the **active** state, and remains in the **active** state after executing the command. If the account is in any other state, the RP MUST return an [Incompatible State Response](#incompatible-state-response).

The functionality of the **unauthorize** command is also performed by **suspend**, **archive**, and **delete** commands to ensure an account in the **suspended**, **archived**, and **unknown** state no longer had access to resources.


# Lifecycle Commands

Support for lifecycle commands is OPTIONAL. 

Lifecycle commands are defined for each transition defined in ISO 24760-1 (2019-05 edition), Section 7.2 for accounts in an RP's identity register as defined in {{ISO}} 3.4.5.

## Lifecycle States

The commands transition the account between the following states:

- **unknown**

- **active**

- **suspended**

- **archived**

Following are the potential state transitions:

```
                      +--------------------------------------- reactivate ---+                   
                      |  +--- maintain --+                                   |
                      |  |               |                                   |
+---------+           |  |   +--------+  |                    +-----------+  |
|         |           |  +-> |        | -+                    |           | -+ 
| unknown |           + ---> | active | -------- suspend ---> | suspended | --------+
|         | --- activate --> |        | ----+                 |           | -+      |
+---------+              +-> |        | -+  |                 +-----------+  |      |
  ^  ^  ^                |   +--------+  |  |                                |      |
  |  |  |                |               |  |            +------- archive ---+      | 
  |  |  |                |               |  |            |                          |
  |  |  |                |               |  |            |     +-----------+        |
  |  |  |                |               |  |            +---> |           | ----+  | 
  |  |  |                |               |  +--- archive ----> | archived  | -+  |  |            
  |  |  |                |               |                     |           |  |  |  |
  |  |  |                |               |                     +-----------+  |  |  |
  |  |  |                |               |                                    |  |  |
  |  |  |                +---------------|----------------------- restore ----+  |  | 
  |  |  |                                |                                       |  |   
  |  |  +--------------------- delete ---+                                       |  |
  |  +------------------------ delete -------------------------------------------+  | 
  +--------------------------- delete ----------------------------------------------+
```                                             


## Lifecycle Command Success Response

When an RP successful processes a lifecycle command, the RP returns an HTTP 200 Ok and a JSON object containing **account_state** set to the state of the account after processing. 

Following is a non-normative response to a successful **activate** command:

```json
{
  "account_state": "active"
}
```

## Incompatible State Response

If the account is in an incompatible state in the identity register for the lifecycle command, the RP returns an HTTP 409 Conflict and a JSON object containing:

- **account_state** where the value is the current state of the account in the identity register
- **error** set to the value "incompatible_state"

Following is a non-normative response to a unsuccessful **restore** command where the account was in the **suspended** state:

```json
{
  "account_state": "suspended",
  "error": "incompatible_state"
}
```

Note that if an **activate** command is sent for an account that exists, or one of the other commands are sent for an account that does not exist, 
the account is incompatible state. 

Following is a non-normative response to an unsuccessful **activate** for an existing account in the **active** state:

```json
{
  "account_state": "active",
  "error": "incompatible_state"
}
```

Following is a non-normative response to an unsuccessful **maintain** for an non-existing account:

```json
{
  "account_state": "unknown",
  "error": "incompatible_state"
}
```


## **activate** Command
The RP MUST an account with the included claims in the identity register. The account MUST be in the **unknown** state. The account is in the **active** state after successful processing. If the account is already in the register, 

## **maintain** Command
The RP MUST update an existing account in the identity register with the included claims. The account MUST be in the **active** state. The account remains in the **active** state after successful processing.

## **suspend** Command
The RP MUST perform the `unauthorize` functionality on the account and mark the account as being temporarily unavailable in the identity register. The account MUST be in the **active** state. The account is in the **suspended** state after successful processing.

## **reactivate** Command
The RP MUST mark a suspended account as being active in the identity register. The account MUST be in the **suspended** state. The account is in the **active** state after successful processing. The RP SHOULD support the **reactivate** Command if it supports the **suspend** Command.

## **archive** Command
The RP MUST perform the `unauthorize` functionality on the account and remove the account from the identity register. The account MUST be in either the **active** or **suspended** state. The account is in the **archived** state after successful processing.

## **restore** Command
The RP MUST restore an archived account to the identity register and mark it as being active. The account MUST be in the **archived** state. The account is in the **active** state after successful processing. The RP SHOULD support the **restore** Command if it supports the **archive** Command.

## **delete** Command
The RP MUST perform the `unauthorize` functionality on the account, and delete all data associated with an account. The account can be in any state except **unknown**. The account is in the **unknown** state after successful processing.


# **audit** Command

The OP sends the **audit** Command to learn the state of accounts for an Organization at an RP. The **audit** Command uses [Server-Side Events](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events) (SSE) to transfer **audit** results. The OP MUST include the following HTTP headers when sending the **audit** Command:


- `Accept` with the value of `text/event-stream` to indicate support for Server-Sent Events.
- `Cache-Control` with the value of `no-cache` to signal to intermediaries to not cache the response.
- `Connection` with the value of `keep-alive` to keep the connection open.


It is RECOMMENDED the OP accept compression of the response by sending the `Accept-Encoding` HTTP header with the value of `gzip`, or `gzip, br`. 

The following is a non-normative example of an **audit** Command request:
(with most of the Command Token contents omitted for brevity):

```
POST /commands HTTP/1.1
Host: rp.example.net
Content-Type: application/x-www-form-urlencoded
Accept: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Accept-Encoding: gzip, br

command_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```

## **audit Response**

The RP uses SSE to stream the **audit** response as a sequence of events. If the RP receives a valid **audit** Command, it MUST sent the `HTTP/1.1 200 OK` response, followed by the following headers:

- `Content-Type` with the `text/event-stream` value
- `Cache-Control` with the `no-cache` value
- `Connection` with the `keep-alive` value

If the OP sent a `Content-Encoding` header in the request with a compression the RP understands, the RP MAY include a `Content-Encoding` header with one of the OP provided values.

Per SSE, the body of the response is a series of events. In addition to the required field name `data`, each event MUST include the `id` field with a unique value for each event, and the `event` field with a value of either `account-audit`, or `audit-complete`. The RP sends an `account-audit` event for each account at the RP for the `iss`, and `org` if sent, in the **audit** Command. When all `account-audit` events have been sent, the RP sends an `audit-complete` event.

The `data` parameter of the `account-audit` event MUST contain the following:

- `sub` representing the account
- `account_state` representing the current state for the account from the states supported by the RP.
- `groups` if groups have been provided for the account.
- any identity claims that have been provided to the RP by the OP

The `data` parameter of the `audit-complete` event MUST include the `total_accounts` property with a value for the total number of `account-audit` events the RP has sent.

If there are no accounts at the RP that match the `audit` Command, the RP responds with only the `audit-complete` event with `total-accounts` having a value of 0.

If the connection is lost, The OP SHOULD resend a new `audit` command and include the HTTP header `Last-Event-Id` with the last event `id` property received per SSE. Each event MUST include a unique `id` value.

The following is a non-normative example of an **audit** Command request:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Content-Encoding: gzip

id: 1
event: account-audit
data: {
  "sub": "248289761001",
  "email": "janes.smith@example.com",
  "given_name": "Jane",
  "family_name": "Smith",
  "groups": [
    "b0f4861d-f3d6-4f76-be2f-e467daddc6f6",
    "88799417-c72f-48fc-9e63-f012d8822ad1",
  ],
  "account_state": "active"
}

id: 2
event: account-audit
data: {
  "sub": "98765412345",
  "email": "john.doe@example.com",
  "given_name": "John",
  "family_name": "Doe",
  "groups": [
    "88799417-c72f-48fc-9e63-f012d8822ad1"
  ],
  "account_state": "suspended"
}

id: 3
event: audit-complete
data: {
  "total_accounts": 2
}

```


# OpenID Provider Command Support

## Indicating OP Support

If the OpenID Provider supports
{{OpenID.Discovery}},
it uses this metadata value to advertise its support for lifecycle commands:

- **commands_supported**  
  OPTIONAL.  
  A JSON array containing the OpenID Provider Commands that the OP supports.

## Indicating RP Support

Relying Parties supporting lifecycle commands register an Commands URI with the OP as part of their client registration.

The Commands URI MUST be an absolute URI as defined by
Section 4.3 of {{RFC3986}}.
The commands URI MAY include an
`application/x-www-form-urlencoded` formatted
query component, per Section 3.4 of {{RFC3986}},
which MUST be retained when adding additional query parameters.
The commands URI MUST NOT include a fragment component.

If the RP supports
[OpenID Connect Dynamic Client Registration 1.0](#OpenID.Registration),
it uses this metadata value to register the lifecycle commands URI:

- **commands_uri**  
  OPTIONAL.  
  RP URL that will receive OpenID Provider Commands from the OP.
  This URL MUST use the `https` scheme
  and MAY contain path, and query parameter components.

  And it uses this metadata value to indicate the commands supported:

- **commands_supported**

# Implementation Considerations

This specification defines features used by both Relying Parties and
OpenID Providers that choose to implement OpenID Provider Commands.
All of these Relying Parties and OpenID Providers
MUST implement the features that are listed
in this specification as being "REQUIRED" or are described with a "MUST".
No other implementation considerations for implementations of
OpenID Provider Commands are defined by this specification.

# Security Considerations

The signed Command Token is required in the command request to prevent
denial of service attacks by enabling the RP to verify that
the command request is coming from a legitimate party.

OPs are encouraged to use short expiration times in Command Tokens,
preferably at most two minutes in the future,
to prevent captured Command Tokens from being replayed.


## Cross-JWT Confusion

As described in Section 2.8 of {{RFC8725}},
attackers may attempt to use a JWT issued for one purpose in a context that it was not intended for.
The mitigations described for these attacks can be applied to Command Tokens.

One way that an attacker might attempt to repurpose a Command Token
is to try to use it as an ID Token.
As described in [Command Token](#command-token),
inclusion of a `nonce` Claim in a Command Token
is prohibited to prevent its misuse as an ID Token.

Another way to prevent cross-JWT confusion is to use explicit typing,
as described in Section 3.11 of {{!RFC8725}}.
One would explicitly type a Command Token by including a
`typ` (type) Header Parameter with a value of
`command+jwt`
(which is registered in [Media Type Registration](#media-type-registration)).
Note that the `application/` portion of the
`application/command+jwt` media type name is omitted
when used as a `typ` Header Parameter value,
as described in the `typ` definition
in Section 4.1.9 of {{!RFC7515}}.
Including an explicit type in issued Command Tokens is a best practice.
Note however, that requiring explicitly typed Command Tokens
will break most existing deployments,
as existing OPs and RPs are already commonly using untyped Command Tokens.
However, requiring explicit typing would be a good idea
for new deployment profiles where compatibility with existing deployments
is not a consideration.


# Privacy Considerations

*To be completed.*

# IANA Considerations

*Not all entries have been added.*


## OAuth Authorization Server Metadata Registry

This specification registers the following metadata name in the
IANA "OAuth Authorization Server Metadata" registry [IANA OAuth Parameters]{{IANA.OAuth.Parameters}}
established by [RFC8414]{{RFC8414}}.

### Registration

- **Metadata Name:** `commands_supported`

  **Metadata Description:**
  JSON array containing the OpenID Provider Commands that the OP supports.

  **Change Controller:** OpenID Foundation

  **Specification Document(s):** This document

## OAuth Dynamic Client Registration Metadata Registration

This specification registers the following client metadata definitions
in the IANA "OAuth Dynamic Client Registration Metadata" registry
[IANA OAuth Parameters](#IANA.OAuth.Parameters)
established by [RFC7591](#RFC7591).

### Registry Contents

- **Client Metadata Name:** `commands_uri`
  
**Client Metadata Description:**
  RP URL that will receive OpenID Provider Commands from the OP

**Change Controller:** OpenID Foundation

**Specification Document(s):** This document


## Media Type Registration

This specification registers the `application/command+jwt` media type as per {{!RFC6838}}.

*To be completed.*


# References

## Normative References

*To be completed.*

## Informative References

*To be completed.*

# Acknowledgements

The authors would like to thank early feedback provided by George Fletcher, Dean Saxe, and Rifaat Shekh-Yusef.

*To be updated.*

# Notices

*To be completed.*

# Document History

   [[ To be removed from the final specification ]]

   -00

   initial draft