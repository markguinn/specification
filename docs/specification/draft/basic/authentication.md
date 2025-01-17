---
title: Authentication
type: docs
weight: 15
---

{{< callout type="info" >}} **Protocol Revision**: draft {{< /callout >}}

## 1. Introduction

### 1.1 Purpose and Scope

The Model Context Protocol provides authentication capabilities at the transport level,
enabling MCP clients to make requests to restricted resources on behalf of resource
owners. This specification defines the authentication flow for HTTP+SSE transport.

### 1.2 Protocol Requirements

MCP clients and servers supporting HTTP+SSE transport **SHOULD** implement this
specification, though they **MAY** support additional authentication mechanisms.
Implementations using alternative transports **MAY** implement authentication according
to the best practices for their specific transport.

For deployments supporting third-party clients or servers via HTTP+SSE, implementations
**SHOULD** expect authentication as defined in this document. However, controlled
environment deployments like corporate networks **MAY** implement alternative methods
such as certificate-based authentication.

### 1.3 Standards Compliance

This authentication mechanism implements following specifications but recommends a
specific subsets:

- [OAuth 2.1 IETF DRAFT](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12)
- OAuth 2.0 Authentication Server Metadata
  ([RFC8414](https://datatracker.ietf.org/doc/html/rfc8414))
- OAuth 2.0 Dynamic Client Registration Protocol
  ([RFC7591](https://datatracker.ietf.org/doc/html/rfc7591))

## 2. Authentication Flow

### 2.1 Overview

MCP is designed to support interactive, user-facing developer tools that connect to
arbitrary servers. This leads to two key design decisions:

1. All MCP clients **MUST** be treated as public OAuth 2.1 clients, as they cannot
   securely store client secrets when distributed to end users

2. PKCE **MUST** be implemented by all clients to prevent authorization code interception
   attacks, which is especially important for locally-running tools

3. All MCP implementations **MUST** support the OAuth 2.0 Dynamic Client Registration
   Protocol ([RFC7591](https://datatracker.ietf.org/doc/html/rfc7591)), with the
   exception of servers that only listen on loopback interfaces (e.g. localhost for
   native application authentication).

4. All MCP implementations **SHOULD** implement OAuth 2.0 Authentication Server Metadata
   ([RFC8414](https://datatracker.ietf.org/doc/html/rfc8414)). Servers that do not
   support Authentication Server Metadata **MUST** follow the default URI schema.

### 2.2 Basic OAuth 2.1 Authentication

When authentication is required, servers **MUST** respond with HTTP 401 Unauthorized. The
basic
[OAuth 2.1 IETF DRAFT](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12)
authentication flow with PKCE comprises the following sequence:

```mermaid
sequenceDiagram
    participant B as User-Agent (Browser)
    participant C as Client
    participant M as MCP Server

    C->>M: MCP Request
    M->>C: HTTP 401 Unauthorized
    Note over C: Generate code_verifier and code_challenge
    C->>B: Open browser with authorization URL + code_challenge
    B->>M: GET /authorize
    Note over M: User logs in and authorizes
    M->>B: Redirect to callback URL with auth code
    B->>C: Callback with authorization code
    C->>M: Token Request with code + code_verifier
    M->>C: Access Token + Refresh Token
    C->>M: MCP Request with Access Token
    Note over C,M: Begin standard MCP message exchange
```

### 2.3 Server Metadata Discovery

Since MCP clients need to connect to previously unknown servers, automated server
capability discovery is essential. The metadata discovery mechanism allows clients to:

- Locate the OAuth endpoints without manual configuration
- Determine supported authentication features
- Adapt to server-specific requirements
- Enable zero-configuration connections to new servers

The discovery flow is illustrated below:

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: GET /.well-known/oauth-authorization-server
    alt Discovery Success
        S->>C: 200 OK + Metadata Document
        Note over C: Use endpoints from metadata
    else Discovery Failed
        S->>C: 404 Not Found
        Note over C: Fall back to default endpoints
    end
    Note over C: Continue with authentication flow
```

#### 2.3.1 Discovery Document Location

MCP servers **SHOULD** implement OAuth 2.0 Authentication Server Metadata
[RFC8414](https://datatracker.ietf.org/doc/html/rfc8414) by exposing a discovery document
at:

```
/.well-known/oauth-authorization-server
```

#### 2.3.2 Required Fields

The discovery document **MUST** include the following fields:

| Field                            | Description                        |
| -------------------------------- | ---------------------------------- |
| authorization_endpoint           | Authorization request endpoint URL |
| token_endpoint                   | Token exchange endpoint URL        |
| registration_endpoint            | Client registration endpoint URL   |
| code_challenge_methods_supported | Supported PKCE methods             |
| response_types_supported         | Supported OAuth response types     |
| grant_types_supported            | Supported OAuth grant types        |

Example discovery document:

```json
{
  "issuer": "https://mcp.example.com",
  "authorization_endpoint": "https://mcp.example.com/authorize",
  "token_endpoint": "https://mcp.example.com/token",
  "registration_endpoint": "https://mcp.example.com/register",
  "code_challenge_methods_supported": ["S256"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "scopes_supported": ["mcp"]
}
```

#### 2.3.3 Fallback for Servers without Server Metadata Discovery

For servers that do not implement OAuth 2.0 Authentication Server Metadata, clients
**MUST** use the following default endpoint paths relative to the server's base URL:

| Endpoint               | Default Path | Description                          |
| ---------------------- | ------------ | ------------------------------------ |
| Authorization Endpoint | /authorize   | Used for authorization requests      |
| Token Endpoint         | /token       | Used for token exchange & refresh    |
| Registration Endpoint  | /register    | Used for dynamic client registration |

These default paths **MUST** be used when:

1. The /.well-known/oauth-authorization-server endpoint returns a 404 status code
2. The metadata document cannot be retrieved or parsed
3. Required endpoints are missing from the metadata document

Clients **SHOULD** first attempt to discover endpoints via the metadata document before
falling back to default paths. When using default paths, all other protocol requirements
remain unchanged.

Example default endpoints for https://mcp.example.com:

```
https://mcp.example.com/authorize
https://mcp.example.com/token
https://mcp.example.com/register
```

### 2.3 Client Registration Requirements

MCP defines two distinct registration scenarios based on the redirect URI:

#### 2.3.1 Localhost Redirect URIs

When using localhost redirect URIs (http://localhost:{port} or http://127.0.0.1:{port}),
clients:

- Do **NOT** require dynamic registration
- **MUST** use PKCE
- **MAY** proceed directly to authorization
- **MUST NOT** require client secrets

This exception for localhost is explicitly supported by OAuth 2.1 for public clients and
provides a secure flow through the combination of PKCE and localhost-only redirects.

#### 2.3.2 Non-Localhost Redirect URIs

For all other redirect URIs, dynamic client registration is **REQUIRED**. This provides a
standardized way for clients to automatically register with new servers, which is crucial
for MCP because:

- Clients cannot know all possible servers in advance
- Manual registration would create friction for users
- It enables seamless connection to new servers
- Servers can implement their own registration policies

#### 2.3.3 Registration Process

For non-localhost scenarios, clients register with the server by making a POST request to
the registration endpoint with their client metadata:

```
POST /register HTTP/1.1
Content-Type: application/json

{
  "client_name": "Example MCP Client",
  "redirect_uris": ["https://example.com/callback"],
  "token_endpoint_auth_method": "none",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "scope": "mcp"
}
```

#### 2.3.4 Server Response

The server responds with client credentials:

```json
{
  "client_id": "s6BhdRkqt3",
  "client_secret": null,
  "registration_access_token": "reg-23410913-abewfq",
  "registration_client_uri": "https://mcp.example.com/register/s6BhdRkqt3",
  "client_id_issued_at": 2893256800
}
```

### 2.4 Authentication Flow Steps

The complete authentication flow proceeds as follows:

1. Client discovers server metadata (if supported)
2. For non-localhost redirect URIs, client **MUST** register dynamically with server
3. Client generates PKCE code_verifier and code_challenge
4. Client requests authorization with code_challenge
5. User authenticates and authorizes the client
6. Server returns authorization code
7. Client exchanges code and code_verifier for tokens
8. Client uses access token for API requests

**NOTE**: Dynamic registration (step 2) is only required when using non-localhost
redirect URIs. Clients using localhost redirect URIs may skip this step.

Flow diagram:

```mermaid
sequenceDiagram
    participant B as User-Agent (Browser)
    participant C as Client
    participant M as MCP Server

    C->>M: GET /.well-known/oauth-authorization-server
    alt Server Supports Discovery
        M->>C: Authorization Server Metadata
    else No Discovery
        M->>C: 404 (Use default endpoints)
    end

    alt Non-Localhost Redirect URI
        C->>M: POST /register
        M->>C: Client Credentials
    end

    Note over C: Generate PKCE Parameters
    C->>B: Open browser with authorization URL + code_challenge
    B->>M: Authorization Request
    Note over M: User authenticates/authorizes
    M->>B: Redirect to callback with authorization code
    B->>C: Authorization code callback
    C->>M: Token Request + code_verifier
    M->>C: Access Token + Refresh Token
    C->>M: API Requests with Access Token
```

### 2.5 Access Token Usage

#### 2.5.1 Token Requirements

Access token handling **MUST** conform to
[OAuth 2.1 Section 5](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5)
requirements for resource requests. Specifically:

1. Access tokens **MUST** be included using one of the two methods defined in
   [Section 5.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.1)

2. The preferred method is using the Authorization header with Bearer scheme per
   [Section 5.1.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.1.1):

```
Authorization: Bearer <access-token>
```

3. Form-encoded body parameters per
   [Section 5.1.2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.1.2)
   **MAY** be supported as a fallback

4. Access tokens **MUST NOT** be included in the URI query string

Example request:

```http
GET /v1/contexts HTTP/1.1
Host: mcp.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

#### 2.5.2 Token Handling

Resource servers **MUST** validate access tokens as described in
[Section 5.2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.2).
If validation fails, servers **MUST** respond according to
[Section 5.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.3)
error handling requirements. Invalid or expired tokens **MUST** receive a HTTP 401
response with a WWW-Authenticate header indicating the error.

### 2.6 Security Considerations

The following security requirements **MUST** be implemented:

1. Clients **MUST** securely store tokens following OAuth 2.0 best practices
2. Servers **SHOULD** enforce token expiration and rotation
3. All authorization endpoints **MUST** be served over HTTPS
4. Servers **MUST** validate redirect URIs to prevent open redirect vulnerabilities
5. Redirect URIs **MUST** be either localhost URLs or HTTPS URLs

### 2.7 Error Handling

Servers **MUST** return appropriate HTTP status codes for authentication errors:

| Status Code | Description  | Usage                                      |
| ----------- | ------------ | ------------------------------------------ |
| 401         | Unauthorized | Authentication required or token invalid   |
| 403         | Forbidden    | Invalid scopes or insufficient permissions |
| 400         | Bad Request  | Malformed authentication request           |

### 2.8 Implementation Requirements

1. Implementations **MUST** follow OAuth 2.1 security best practices
2. PKCE is **REQUIRED** for all clients
3. Refresh token rotation **SHOULD** be implemented for enhanced security
4. Token lifetimes **SHOULD** be limited based on security requirements

### 2.9 Third-Party Authentication Flow

#### 2.9.1 Overview

MCP servers **MAY** support delegated authentication through third-party authorization servers. In this flow, the MCP server acts as both an OAuth client (to the third-party auth server) and an OAuth authorization server (to the MCP client).

#### 2.9.2 Flow Description

The third-party authentication flow comprises these steps:

1. MCP client initiates standard OAuth flow with MCP server
2. MCP server redirects user to third-party authorization server
3. User authenticates with third-party server
4. Third-party server redirects back to MCP server with authorization code
5. MCP server exchanges code for third-party access token
6. MCP server generates its own access token bound to the third-party session
7. MCP server completes original OAuth flow with MCP client

```mermaid
sequenceDiagram
    participant B as User-Agent (Browser)
    participant C as MCP Client
    participant M as MCP Server
    participant T as Third-Party Auth Server

    C->>M: Initial OAuth Request
    M->>B: Redirect to Third-Party /authorize
    B->>T: Authorization Request
    Note over T: User authenticates
    T->>B: Redirect to MCP Server callback
    B->>M: Authorization code
    M->>T: Exchange code for token
    T->>M: Third-party access token
    Note over M: Generate bound MCP token
    M->>B: Redirect to MCP Client callback
    B->>C: MCP authorization code
    C->>M: Exchange code for token
    M->>C: MCP access token
```

#### 2.9.3 Session Binding Requirements

MCP servers implementing third-party authentication **MUST**:

1. Maintain secure mapping between third-party tokens and issued MCP tokens
2. Validate third-party token status before honoring MCP tokens
3. Implement appropriate token lifecycle management
4. Handle third-party token expiration and renewal

#### 2.9.4 Security Considerations

When implementing third-party authentication, servers **MUST**:

1. Validate all redirect URIs
2. Securely store third-party credentials
3. Implement appropriate session timeout handling
4. Consider security implications of token chaining
5. Implement proper error handling for third-party auth failures
