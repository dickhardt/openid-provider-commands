%%%
title = "OpenID Lifecycle Commands - draft 00"
abbrev = "openid-lifecycle-commands"
ipr = "none"
workgroup = "OpenID Connect"
keyword = ["security", "openid", "lifecycle"]

[seriesInfo]
name = "Internet-Draft"
value = "openid-lifecycle-commands-1_0-10"
status = "standard"

[[author]]
initials="D."
surname="Hardt"
fullname="Dick Hardt"
organization="Hellō"
    [author.address]
    email = "dick.hardt@gmail.com"

%%%

.# Abstract


OpenID Connect defines a protocol for an end-user to use an OpenID Provider (OP) to log in to a Relying Party (RP) and assert claims about the end-user using an ID Token.

OpenID Lifecycle Commands complements OpenID Connect by introducing a set of commands for an OP to directly manage the lifecycle of an end-user account at an RP. These commands enable an OP to create, update, delete, activate, restore, archive, suspend, and unauthorize an end-user account. Command tokens re-use OpenID Connect ID Token mechanisms, lowering the barrier to adoption by an OpenID Connect RP.

{mainmatter}

# Introduction

OpenID Connect 1.0 (OIDC) is a widely adopted identity protocol that enables client applications, known as relying parties (RPs), to verify the identity of end-users based on authentication performed by a trusted service, the OpenID Provider (OP). OIDC also provides mechanisms for securely obtaining identity attributes, or claims, about the end-user, which helps RPs tailor experiences and manage access with confidence.

OIDC not only allows an end-user to log in and authorize access to an RP but also facilitates creating an account with the RP. However, account creation is only the beginning of an account’s lifecycle. Throughout the lifecycle, various actions may be required to ensure data integrity, security, and regulatory compliance.

For example, many jurisdictions grant end-users the “right to be forgotten,” enabling them to request the deletion of their accounts and associated data. When such requests arise, OPs may need to notify RPs to fully delete the end-user’s account and remove all related data, respecting both regulatory obligations and end-user privacy preferences.

In scenarios where malicious activity is detected or suspected, OPs play a vital role in protecting end-users. They may need to instruct RPs to revoke access or delete accounts created by malicious actors. This helps contain the impact of unauthorized actions and prevent further misuse of compromised accounts.

In enterprise environments, where organizations centrally manage workforce access, OPs handle essential account operations across various stages of the lifecycle. These operations include creating, updating, deleting, activating, restoring, archiving, suspending, and unauthorizing accounts to maintain security and compliance.

OpenID Lifecycle Commands enable OPs to manage these account lifecycle stages directly with RPs, extending the functionality of OIDC to cover the full spectrum of account management needs.

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

- **Command Token**: A JSON Web Token (JWT) similar to an ID Token that contains Claims about the command being issued.

# Lifecycle Commands

## Commands

This document defines the following lifecycle commands:

- **create**: Create an account with the included claims. If the account already exists, return the `account_exists` error.

- **update**: Update an existing account with the included claims.

- **unauthorize**: Revoke all active sessions including `offline_access` granted to the account.

- **suspend**: Mark the account as being temporarily unavailable, and perform the `unauthorize` command on the account.

- **activate**: Mark a suspended account as being available.

- **archive**: Remove the account (details TBD). Perform the `unauthorize` command on the account.

- **restore**: Restore an archived account.

- **delete**: Delete all data associated with an account, and perform the `unauthorize` command on the account.

For all commands other than `create`, if the account does not exist, return the `unknown_account` error.

## Indicating OP Support for Lifecycle Commands

If the OpenID Provider supports
{{OpenID.Discovery}},
it uses this metadata value to advertise its support for lifecycle commands:

- **lifecycle_commands_supported**  
  OPTIONAL.  
  A JSON array containing the lifecycle commands that the OP supports.

## Indicating RP Support for Lifecycle Commands

Relying Parties supporting lifecycle commands register a lifecycle commands URI with the OP as part of their client registration.

The lifecycle commands URI MUST be an absolute URI as defined by
Section 4.3 of {{RFC3986}}.
The lifecycle commands URI MAY include an
`application/x-www-form-urlencoded` formatted
query component, per Section 3.4 of {{RFC3986}},
which MUST be retained when adding additional query parameters.
The lifecycle commands URI MUST NOT include a fragment component.

## Command Token

OPs send a JWT similar to an ID Token to RPs called a Command Token
to issue lifecycle commands.
ID Tokens are defined in Section 2 of {{OpenID.Core}}.

The following Claims are used within the Command Token:

- **iss**  
  REQUIRED.  
  Issuer Identifier, as specified in Section 2 of {{OpenID.Core}}.

- **sub**  
  REQUIRED.  
  Subject Identifier, as specified in Section 2 of {{OpenID.Core}}. The `sub` claim identifies the account for the command.

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
  The command for the RP to execute. See [Lifecycle Commands](#lifecycle-commands) for the list of supported commands.

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
  “iss”: “https://server.example.com”,
  “sub”: “248289761001”,
  “aud”: “s6BhdRkqt3”,
  “iat”: 1711718400,
  “exp”: 1711722000,
  “jti”: “bWJq”,
  “http://schemas.openid.net/command”: “unauthorize”
}
```

## Lifecycle Command Request

The OP uses an HTTP POST to the registered lifecycle commands URI
to send lifecycle commands to the RP. The POST body uses the
`application/x-www-form-urlencoded` encoding
and must include a `command_token` parameter
containing a Command Token from the OP for the RP identifying the
command to be executed.

The POST body MAY contain other values in addition to
`command_token`.
Values that are not understood by the implementation MUST be ignored.

The following is a non-normative example of such a command request
(with most of the Command Token contents omitted for brevity):

```
Cache-Control: no-store
```
# Implementation Considerations

This specification defines features used by both Relying Parties and
OpenID Providers that choose to implement OpenID Lifecycle Commands.
All of these Relying Parties and OpenID Providers
MUST implement the features that are listed
in this specification as being "REQUIRED" or are described with a "MUST".
No other implementation considerations for implementations of
Lifecycle Commands are defined by this specification.

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


# IANA Considerations

## OAuth Authorization Server Metadata Registry

This specification registers the following metadata name in the
IANA "OAuth Authorization Server Metadata" registry {{IANA.OAuth.Parameters}}
established by {{RFC8414}}.

### Registration

- **Metadata Name:** `lifecycle_commands_supported`

  **Metadata Description:**
  JSON array containing the lifecycle commands that the OP supports.

  **Change Controller:** OpenID Foundation

  **Specification Document(s):** This document

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