%%%
title = "OpenID Account Commands - draft 01"
abbrev = "openid-account-commands"
ipr = "none"
workgroup = "OpenID Connect"
keyword = ["security", "openid", "lifecycle", "accounts"]

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

OpenID Account Commands complements OpenID Connect by introducing a set of commands for an OP to directly manage an end-user account at an RP. These commands enable an OP to activate, maintain, suspend, reactivate, archive, restore, delete, and unauthorize an end-user account. Command tokens simplify RP adoption by re-use of the OpenID Connect ID Token schema, security, and verification mechanisms.

{mainmatter}

# Introduction

OpenID Connect 1.0 (OIDC) is a widely adopted identity protocol that enables client applications, known as relying parties (RPs), to verify the identity of end-users based on authentication performed by a trusted service, the OpenID Provider (OP). OIDC also provides mechanisms for securely obtaining identity attributes, or claims, about the end-user, which helps RPs tailor experiences and manage access with confidence.

OIDC not only allows an end-user to log in and authorize access to an RP but also facilitates creating an account with the RP. However, account creation is only the beginning of an account's lifecycle. Throughout the lifecycle, various actions may be required to ensure data integrity, security, and regulatory compliance.

For example, many jurisdictions grant end-users the “right to be forgotten,” enabling them to request the deletion of their accounts and associated data. When such requests arise, OPs may need to notify RPs to fully delete the end-user's account and remove all related data, respecting both regulatory obligations and end-user privacy preferences.

In scenarios where malicious activity is detected or suspected, OPs play a vital role in protecting end-users. They may need to instruct RPs to revoke authorization or delete accounts created by malicious actors. This helps contain the impact of unauthorized actions and prevent further misuse of compromised accounts.

In enterprise environments, where organizations centrally manage workforce access, OPs handle essential account operations across various stages of the lifecycle. These operations include activating, maintaining, suspending, reactivating, archiving, restoring, and deleting accounts to maintain security and compliance.

OpenID Account Commands enable OPs to manage these account lifecycle stages directly with RPs, extending the functionality of OIDC to cover the full spectrum of account management needs.

> NOTE
>
> A design objective is to allow the OP to perform key account management with a mechanism that is easy to adopt by the RP. This specification builds upon the ID Token signing mechanism that an RP already supports. No additional credentials are required to be configured or managed by with either the OP or RP. An RP can support only a subset of commands, and the OP can discovery which ones a given RP supports. OpenID Account Commands do not require any changes to deployed protocol endpoints.

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

This specification defines the following term:

- **Account**: *To be completed.*

- **Command**: *To be completed.*

- **Command Token**: A JSON Web Token (JWT) similar to an ID Token that contains Claims about the command being issued.

- **Commands URI**: *To be completed.*

## Overview

This specification defines a Command Request sent from the OP to the RP, and a Command Response returned from the RP to the OP. The Command Request is an HTTP POST with a `application/x-www-form-urlencoded` encoded body containing a signed, and optionally encrypted JWT, the Command Token. The Command Response is an HTTP status code and potentially an `application/json` encoded response.

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
Host: rp.example.org
Content-Type: application/x-www-form-urlencoded

command_token=eyJhbGci ... .eyJpc3Mi ... .T3BlbklE ...
```

# Command Response

If the command succeeded, the RP MUST respond with HTTP 200 OK.
However, note that some Web frameworks will substitute an HTTP 204 No Content response
for an HTTP 200 OK when the HTTP body is empty.
Therefore, OPs should be prepared to also process an HTTP 204 No Content response as a successful response.

If the command request was invalid or the command failed,
the RP MUST respond with HTTP 400 Bad Request.
The response MAY include an HTTP body consisting of a JSON object with
`error` and `error_description` parameters
conveying the nature of the error that occurred, which can assist with debugging.
These error response parameters are used as specified in Section 5.2 of {{!RFC6749}}.
Like in OAuth 2.0, the parameters are included in the entity-body of the HTTP response
using the `application/json` media type.
Also like in OAuth 2.0, the `error` parameter is REQUIRED when a response body is present
and the `error_description` parameter is OPTIONAL.
An `error` value of `invalid_request`
MAY be used to indicate that there was a problem with the syntax of the command request.
Note that the information conveyed in the response body is intended to help debug deployments;
it is not intended that implementations use different `error` values
to trigger different runtime behaviors.

The RP's response SHOULD include the Cache-Control HTTP response header field with a no-store value, keeping the response from being cached to prevent cached responses from interfering with future command requests. An example of this is:

```
Cache-Control: no-store
```


# Command Token

OPs send a JWT similar to an ID Token to RPs called a Command Token
to issue commands. ID Tokens are defined in Section 2 of {{OpenID.Core}}.

The following Claims are used within the Command Token:

- **iss**  
  REQUIRED.  
  Issuer Identifier, as specified in Section 2 of {{OpenID.Core}}.

- **sub**  
  REQUIRED except for **describe** command. 
  PROHIBITED for the **describe** command. 
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

- **http://schemas.openid.net/command**  
  REQUIRED.  
  The command for the RP to execute. See [Commands](#commands) for standard values defined in this document. Other specifications may define additional values.

> NOTE
>
> Use "cmd" rather than a URI? 


- **tenant identifier**
  OPTIONAL.
  An OP specific claim that identifies the OP tenant. Examples include the **hd** claim from the Google OP, and the **tid** claim from the Microsoft OP.

> NOTE
>
> Aligning OPs on a standard tenant identifier does not seem feasible given current deployments.


- **groups**
  OPTIONAL for the **activate** and **maintain** lifecycle commands.
  The groups claim as defined in {{SCIM}}

> NOTE
>
> the **$ref** claim in SCIM **group** claim does not make sense for commands as groups are not an independent resource. Do we specific groups in this context to just be the **value** and **display** 



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

A non-normative example JWT Claims Set for a Command Token follows:

```
{
  “iss”: “https://op.example.org,
  “sub”: “248289761001”,
  “aud”: “s6BhdRkqt3”,
  “iat”: 1711718400,
  “exp”: 1711722000,
  “jti”: “bWJq”,
  “http://schemas.openid.net/command”: “unauthorize”
}
```

# General Commands

The RP MUST support this command. Support for other commands is optional. 

## **describe**

The OP sends this command to the RP to learn what commands the RP supports, and other metadata about the RP. 

### Describe Response


The RP responds with an `application/json` media type that MUST include:

- **commands_supported**: a JSON array of commands the RP supports.
- **commands_uri**: the RP URL that will receive OpenID Account Commands from the OP.

The response MAY also include any OAuth Dynamic Client Registration Metadata (IANA reference)

The RP MAY provide a different response for different OPs, and for different tenants in a multi-tenant OP.

> NOTE
>
> Do we want to include the `iss` and tenant claim if the response is specific to them?
> How do we separate this metadata of context from RP metadata?
>
> Do we want to enable to RP to send back a signed JWT to indicate it is a contract and allow non-repudiation?
>


Following is a non-normative example of a describe response:

```json
{
  "commands_uri":"https://rp.example.org/commands",
  "commands_supported":[
    "describe",
    "unauthorize",
    "suspend",
    "reactivate",
    "delete"
  ],
  "redirect_uris": [
    "https://rp.example.org/response"
  ]
}
```


## **unauthorize** 
This OP sends this command when it suspects a previous OpenID Connect ID Token issued by the OP was granted to a malicious actor. The RP MUST revoke all active sessions and MUST reverse all authorization that may have been granted to applications, including `offline_access`, for account resources identified by the **sub**.


# Lifecycle Commands

Lifecycle commands are defined for each transition defined in ISO 24760-1 (2019-05 edition), Section 7.2 for accounts in an RP's identity register as defined in {{ISO}} 3.4.5.

## **activate** 
Create an account with the included claims in the identity register. 

## **maintain** 
Update an existing account in the identity register with the included claims.

## **suspend** 
Perform the `unauthorize` command on the account and then mark the account as being temporarily unavailable in the identity register.

## **reactivate**
Mark a suspended account as being active in the identity register.

## **archive** 
Perform the `unauthorize` command on the account and remove the account from the identity register. 

## **restore** 
Restore an archived account to the identity register and mark it as being active.

## **delete** 
Perform the `unauthorize` command on the account, and then delete all data associated with an account.


# Error Responses

## **account_exists**

If the account already exists, return 400 status code, and the `account_exists` error.

Applies to:

*activate

command.

## **unknown_account**

If the account does not exist, return the `unknown_account` error.

Applies to:

*unauthorize
*maintain
*suspend
*reactivate
*archive
*restore
*delete

commands.

## **unsupported_command**

The RP does not support the command requested. The RP may support commands for some OPs, and not others, and for some OP tenants, and not others.

# OpenID Account Command Support

## Indicating OP Support

If the OpenID Provider supports
{{OpenID.Discovery}},
it uses this metadata value to advertise its support for lifecycle commands:

- **commands_supported**  
  OPTIONAL.  
  A JSON array containing the OpenID Account Commands that the OP supports.

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
  RP URL that will receive OpenID Account Commands from the OP.
  This URL MUST use the `https` scheme
  and MAY contain path, and query parameter components.

  And it uses this metadata value to indicate the commands supported:

- **commands_supported**

# Implementation Considerations

This specification defines features used by both Relying Parties and
OpenID Providers that choose to implement OpenID Account Commands.
All of these Relying Parties and OpenID Providers
MUST implement the features that are listed
in this specification as being "REQUIRED" or are described with a "MUST".
No other implementation considerations for implementations of
Account Commands are defined by this specification.

# Security Considerations

The signed Command Token is required in the command request to prevent
denial of service attacks by enabling the RP to verify that
the command request is coming from a legitimate party.

OPs are encouraged to use short expiration times in Command Tokens,
preferably at most two minutes in the future,
to prevent captured Command Tokens from being replayable.


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

## OAuth Authorization Server Metadata Registry

This specification registers the following metadata name in the
IANA "OAuth Authorization Server Metadata" registry [IANA OAuth Parameters]{{IANA.OAuth.Parameters}}
established by [RFC8414]{{RFC8414}}.

### Registration

- **Metadata Name:** `commands_supported`

  **Metadata Description:**
  JSON array containing the OpenID Account Commands that the OP supports.

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
  RP URL that will receive OpenID Account Commands from the OP

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