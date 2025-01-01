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

OpenID Connect defines a protocol for an end-user to use an OpenID Provider (OP) to log in to a Relying Party (RP), assert claims about the end-user using an ID Token, and create an account at the RP with those claims.

OpenID Provider Commands complements OpenID Connect by introducing a set of commands for an OP to directly manage an end-user account at an RP. These commands enable an OP to activate, maintain, suspend, reactivate, archive, restore, delete, and unauthorize an end-user account. Command tokens simplify RP adoption by re-use of the OpenID Connect ID Token schema, security, and verification mechanisms.

{mainmatter}

# Introduction

OpenID Connect 1.0 (OIDC) is a widely adopted identity protocol that enables client applications, known as relying parties (RPs), to verify the identity of end-users based on authentication performed by a trusted service, the OpenID Provider (OP). OIDC also provides mechanisms for securely obtaining identity attributes, or claims, about the end-user, which helps RPs tailor experiences and manage access with confidence.

OIDC not only allows an end-user to log in and authorize access to an RP but also facilitates creating an account with the RP. However, account creation is only the beginning of an account's lifecycle. Throughout the lifecycle, various actions may be required to ensure data integrity, security, and regulatory compliance.

For example, many jurisdictions grant end-users the "right to be forgotten," enabling them to request the deletion of their accounts and associated data. When such requests arise, OPs may need to notify RPs to fully delete the end-user's account and remove all related data, respecting both regulatory obligations and end-user privacy preferences.

In scenarios where malicious activity is detected or suspected, OPs play a vital role in protecting end-users. They may need to instruct RPs to revoke authorization or delete accounts created by malicious actors. This helps contain the impact of unauthorized actions and prevent further misuse of compromised accounts.

In enterprise environments, where organizations centrally manage workforce access, OPs handle essential account operations across various stages of the lifecycle. These operations include activating, maintaining, suspending, reactivating, archiving, restoring, and deleting accounts to maintain security and compliance.

OpenID Provider Commands enable OPs to manage these account lifecycle stages directly with RPs, extending the functionality of OIDC to cover the full spectrum of account management needs.

> Author NOTE
> 
> The target implementors of this specification are RPs that have implemented, or will implement OpenID Connect. Reusing the OpenID Connect ID Token 
> syntax, semantics, and verification simplifies implementation of this specification. Reusing token verification removes any requirement to manage or configure additional credentials by either the OP or RP.
>
> The schema of a Command Token is defined by this specification, and any extensions are defined by the OP, aligning with how an ID Token schema is defined. This is in contrast to SCIM where the group schema and user schema extensions are defined by the service provider, AKA RP. 


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

- **Command**: An instruction from an OP to an RP.

- **Command Token**: A JSON Web Token (JWT) signed by the OP that contains Claims about the Command being issued.

- **Commands URI**: The URL at the RP where OPs post Command Tokens.

## Overview

This specification defines a Command Request containing a Command Token sent from the OP to the RP, and a Command Response returned from the RP to the OP.

```
+------+  Command Request       +------+
|      |---- Command Token ---->|      |
|  OP  |                        |  RP  | 
|      |<-----------------------|      |
+------+      Command Response  +------+
```


# Command Request

The OP uses an HTTP POST to the registered Commands URI
to send account commands to the RP. The POST body uses the
`application/x-www-form-urlencoded` encoding
and must include a `command_token` parameter
containing a Command Token from the OP for the RP.

The POST body MAY contain other values in addition to
`command_token`.
Values that are not understood by the implementation MUST be ignored.

The following is a non-normative example of such a command request
(with most of the Command Token contents omitted for brevity):

```
POST /commands HTTP/1.1
Host: rp.example.net
Content-Type: application/x-www-form-urlencoded

command_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```

# Command Response

If the command succeeded, the RP MUST respond with HTTP 200 OK.
However, note that some Web frameworks will substitute an HTTP 204 No Content response
for an HTTP 200 OK when the HTTP body is empty.
Therefore, OPs should be prepared to also process an HTTP 204 No Content response as a successful response.


The RP's response SHOULD include the Cache-Control HTTP response header field with a no-store value, keeping the response from being cached to prevent cached responses from interfering with future command requests. An example of this is:

```
Cache-Control: no-store
```

If there is a response body, it MUST by JSON and use the `application/json` media type.

If the request is not valid, the RP MUST return an `error` parameter, and may include a `error_description` parameter. 
Note that the information conveyed in an error response is intended to help debug deployments;
it is not intended that implementations use different `error` values
to trigger different runtime behaviors.


## Invalid Request Error

If there was a problem with the syntax of the command request, or the Command Token was invalid, the RP MUST return an HTTP 400 Bad Request and include the `error` parameter with a value of `invalid_request`. 

## Unsupported Command Error

If the RP does not support the command requested, the RP MUST return an HTTP 400 Bad Request and include the `error` parameter with the value of `unsupported_command`.

Note that the RP may support commands for some OPs, and not others, and for some organizations, and not others.

## Server Error

If the RP is unable to process a valid request, the RP MUST respond with a Server Error status code as defined in RFC 9110 section 15.6.

# Command Token

OPs send a JWT similar to an ID Token to RPs called a Command Token
to issue commands. ID Tokens are defined in Section 2 of {{OpenID.Core}}.

The following Claims are used within the Command Token:

- **iss**  
  REQUIRED.  
  Issuer Identifier, as specified in Section 2 of {{OpenID.Core}}.

- **sub**  
  REQUIRED except for **describe** and **groups** command. 

  PROHIBITED for the **describe** and **groups** command. 

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
  OPTIONAL.
  The organization the user belongs to, typically a tenant of the OP. It is a JSON object containing:
    - **id** 
    REQUIRED.
    A JSON string that is an OP specific, globally unique identifier for the organization.

    - **domain**
    OPTIONAL
    A JSON string for a domain name that the OP has verified the organization controls. The **domain** MUST not be used as a persistent identifier for the organization. The **domain** MAY be used to link organization data.

- **groups**
  OPTIONAL for the **activate** and **maintain** lifecycle commands.
  The **groups** claim is a JSON array of zero or more JSON objects that each contain:
    - **id** 
    REQUIRED. 
    A JSON string that is an OP specific, globally unique identifier for the group.

    - **display** 
    REQUIRES.
    A JSON string that is an OP unique human readable string representing the group. The **display** MUST not be used as a persistent identifier for the group.

> Author NOTE
>
> The **groups** claim is registered in [IANA JSON Web Token Claims](https://www.iana.org/assignments/jwt/jwt.xhtml), which refers to [RFC 7643 4.1.2](https://www.rfc-editor.org/rfc/rfc7643.html#section-4.1.2) and [RFC 9068 2.2.3.1](https://www.rfc-editor.org/rfc/rfc9068.html#name-claims-for-authorization-ou)
> Besides defining that it is a list, none of the specifications define the properties of a group. To enable interop, this document defines the contents of the groups claim.
>
> Both **org.id** and **groups[].id** are defined as being a globally unique identifier rather than being OP unique to allow an RP to use them as an index. They are not defined as UUIDs, but that is an acceptable format. Is this a step too far? Should it just be OP unique?


The following Claim MUST NOT be used within the Command Token:

- **nonce**  
  PROHIBITED.  
  A `nonce` Claim MUST NOT be present.
  Its use is prohibited to prevent misuse of the Command Token.

Command Tokens MAY contain other Claims.
Any Claims used that are not understood MUST be ignored.

A Command Token MUST be signed and MAY also be encrypted.
The same keys are used to sign and encrypt Command Tokens
as are used for ID Tokens.
If the Command Token is encrypted, it SHOULD replicate the
`iss` (issuer) claim
in the JWT Header Parameters,
as specified in Section 5.3 of {{JWT}}.

It is RECOMMENDED that Command Tokens be explicitly typed.
This is accomplished by including a `typ` (type) Header Parameter
with a value of `command+jwt` in the Command Token.
See [Security Considerations](#security-considerations) for a discussion of the security and interoperability considerations
of using explicit typing.

A non-normative example JWT Claims Set for Command Token for an **activate** command follows:

```json
{
  "iss": "https://op.example.org",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "iat": 1734003000,
  "exp": 1734003060,
  "jti": "bWJq",
  "command": "activate",
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

A non-normative example JWT Claims Set for Command Token for an **unauthorize** command follows:

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

The OP sends this command to the RP to learn what commands the RP supports, and other metadata about the RP. 

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

- **iss**: the **iss* value from the Command Token.
- **commands_supported**: a JSON array of commands the RP supports. The **describe** value MUST be included.
- **commands_uri**: the RPs Commands URI. This is the URL the Command Token was sent to.

If the Command Token included the **org** claim, then the **org** JSON object with the **id** claim MUST be included in the response.

The response MAY also include any OAuth Dynamic Client Registration Metadata *TBD [IANA reference]https://www.iana.org/assignments/oauth-parameters/oauth-parameters.xhtml#client-metadata)*

The RP MAY provide a different response for different **iss** claim values, and for different **org** claim values.

Following is a non-normative example of a **describe** response to the example request payload in {{describe-command}}:

```json
{
  "iss": "https://op.example.org",
  "org": {
    "id": "73849284748493"
  },
  "commands_uri": "https://rp.example.net/commands",
  "commands_supported":[
    "describe",
    "unauthorize",
    "suspend",
    "reactivate",
    "delete"
  ],
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
The OP sends this command to inform the OP of the **groups** claims the OP may include in ID and Command Tokens. 

The **groups** claim is REQUIRED.
The **org** claim is OPTIONAL and indicates the **groups** are specific to the **org**.

A **groups** command overrides any previous **groups** command.

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


## Lifecycle Command Responses

When an RP successful processes a lifecycle command, the RP returns an HTTP 200 Ok and a JSON object containing **current_state** set the state of the account after processing. 

Following is a non-normative response to a successful **activate** command:

```json
{
  "current_state": "active"
}
```

If the account is in an incompatible state in the identity register for the lifecycle command, the RP returns an HTTP 409 Conflict and a JSON object containing:

- **current_state** where the value is the current state of the account in the identity register
- **error** set to the value "incompatible_state"

Following is a non-normative response to a unsuccessful **restore** command where the account was in the **suspended** state:

```json
{
  "current_state": "suspended",
  "error": "incompatible_state"
}
```

Note that if an **activate** command is sent for an account that exists, or one of the other commands are sent for an account that does not exist, 
the account is incompatible state. 

Following is a non-normative response to an unsuccessful **activate** for an existing account in the **activated** state:

```json
{
  "current_state": "activated",
  "error": "incompatible_state"
}
```

Following is a non-normative response to an unsuccessful **maintain** for an non-existing account:

```json
{
  "current_state": "unknown",
  "error": "incompatible_state"
}
```


## **activate** Command
Create an account with the included claims in the identity register. The account MUST be in the **unknown** state. The account is in the **active** state after successful processing. If the account is already in the register, 

## **maintain** Command
Update an existing account in the identity register with the included claims. The account MUST be in the **active** state. The account remains in the **active** state after successful processing.

## **suspend** Command
Perform the `unauthorize` command on the account and then mark the account as being temporarily unavailable in the identity register. The account MUST be in the **active** state. The account is in the **suspended** state after successful processing.

## **reactivate** Command
Mark a suspended account as being active in the identity register. The account MUST be in the **suspended** state. The account is in the **active** state after successful processing.

## **archive** Command
Perform the `unauthorize` command on the account and remove the account from the identity register. The account MUST be in either the **active** or **suspended** state. The account is in the **archived** state after successful processing.

## **restore** Command
Restore an archived account to the identity register and mark it as being active. The account MUST be in the **archived** state. The account is in the **active** state after successful processing.

## **delete** Command
Perform the `unauthorize` command on the account, and then delete all data associated with an account. The account can be in any state except **unknown**. The account is in the **unknown** state after successful processing.


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

*To be completed.*

# Notices

*To be completed.*

# Document History

   [[ To be removed from the final specification ]]

   -00

   initial draft