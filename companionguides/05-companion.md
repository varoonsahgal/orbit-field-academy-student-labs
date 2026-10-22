# 🛰️ ORBIT FIELD ACADEMY
## MISSION 05 — IDENTITY LOCKDOWN
### 🔐 Companion Field Guide: Cognito, OAuth 2.0, JWTs, Secrets Manager, IAM, and Securing AgentCore

> **Read alongside:** Mission 05, *Identity Lockdown*  
> **Environment:** AWS Management Console only · **Region:** `us-west-2` (US West / Oregon)  
> **Mission:** Prove—not merely assume—that an approved application can invoke the ORBIT agent while missing, forged, and otherwise unauthorized credentials are rejected.

---

## 🎬 The big reveal: you already have an agent. Now who gets through the door?

**Mission 03:** You gave ORBIT reliable policy evidence.  
**Mission 04:** You created a real AI agent with a managed loop, a Gateway, a Lambda order-lookup tool, and a knowledge base.  
**Mission 05:** You protect a **second, separate Harness** with an OAuth/JWT identity boundary and then try to break that boundary.

The big lesson is that **there are multiple, independent kinds of permission**:

- **Inbound authentication:** Is this caller allowed to invoke *this* agent?
- **Agent execution authorization:** What can the running agent access in AWS?
- **Tool authorization:** If the agent calls a tool, is that operation permitted?
- **Business authorization:** Can this caller actually view or change *this customer's* order?

One successful login or valid JWT does **not** answer all four questions.

### 🗺️ Mission 05 architecture — the complete picture

```mermaid
flowchart TB
    T[Lambda test client / Runs inside AWS Lambda] -->|GetSecretValue using Lambda IAM role| S[Secrets Manager / Client ID, secret, token URL, scope]
    T -->|POST client_credentials| C[Cognito OAuth token endpoint]
    C -->|Signed access JWT| T
    T -->|HTTPS: Bearer JWT + message + session ID| A[AgentCore inbound JWT authorizer]
    A -->|Valid issuer, signature, client, scope, expiry| H[Protected Harness / ORBIT support assistant]
    A -->|Missing, forged, or unauthorized| R[Reject request / 401 or 403]
    H --> RT[AgentCore Runtime / Isolated execution session]
    RT --> M[Amazon Bedrock / Claude Haiku model]
    RT -. Logs and traces .-> CW[CloudWatch]
    classDef secure fill:#e4f5ef,stroke:#268362,color:#143b30
    classDef identity fill:#fff0dc,stroke:#c98827,color:#48331d
    classDef blocked fill:#ffe7e7,stroke:#c74e4e,color:#5b2222
    class C,S,A identity
    class H,RT secure
    class R blocked
```

> **Important architectural note:** The Harness is a managed agent built *on* AgentCore Runtime, not a separate agent that calls an unrelated Runtime. The diagram separates them to show their responsibilities. **Mission 05 deliberately does not attach the Gateway or tools from Mission 04.** Its purpose is to isolate and test inbound authentication.

### 🧭 Fast map to the actual lab

| Lab phase | What you create/test | What it proves |
|---|---|---|
| **0–1** | Shared-account names and intended JWT claims | Define the security requirements before configuration |
| **2** | Cognito user pool, M2M app client, resource server/scope, domain | Issue access tokens to an approved application |
| **3** | Secrets Manager secret | Keep the client credential out of source code |
| **4** | Second Harness with JWT inbound auth | Require a signed, correctly scoped bearer token |
| **5A–C** | Lambda test client, IAM role, and code | Simulate a real backend application calling the agent |
| **5D** | `good`, `missing`, `forged` requests | Prove both successful and rejected invocations |
| **5E–6** | Debugging and threat map | Identify exactly where each defense applies |

---

# PART I — THE IDENTITY MYSTERY

## ❓ 01. Why do we need authentication when the agent already has an IAM execution role?

Because the execution role and the caller answer **different questions**.

- **Caller authentication:** Who is trying to invoke the agent?
- **Harness execution role:** After a request has been accepted, what AWS operations may the agent process perform?

Think of a secure warehouse: the front-door badge identifies approved visitors; the warehouse worker's job permissions define which equipment the worker may operate. A visitor's badge does **not** become the worker's permissions.

**In our lab:** Cognito JWT controls the *front door*. The Harness IAM role lets the already-running agent call its Bedrock model. These checks are separate.

## ❓ 02. Authentication vs. authorization — aren't those the same?

No. They are related but distinct:

| Question | Authentication | Authorization |
|---|---|---|
| Meaning | Who or what is calling? | What is this caller allowed to do? |
| Mission 05 example | Verify a token genuinely came from the expected Cognito user pool | Require the allowed client ID and the `orbit-MY_ID-api/invoke` scope |
| Production order example | Customer Alice has signed in | Alice may access **Alice's** order, not Bob's |

A **valid token is not a refund authorization**. Even after the Harness accepts a request, business tools must authorize the specific operation and object.

## ❓ 03. What's OAuth 2.0, in plain English?

OAuth 2.0 is a framework for obtaining and using **access tokens** that allow a client to request access to a protected API.

Instead of sending a password to every API, the application asks a trusted authorization server for an access token. It then presents that token to the API.

```mermaid
sequenceDiagram
    participant App as Approved application
    participant Auth as Cognito authorization server
    participant API as Protected AgentCore endpoint
    App->>Auth: Prove client identity; request invoke scope
    Auth-->>App: Signed access token
    App->>API: Authorization: Bearer ACCESS_TOKEN
    API->>API: Validate token and required permissions
    API-->>App: Answer or rejection
```

This is an *application-to-service security protocol*. It does not give the model freedom to execute arbitrary tools or AWS APIs.

## ❓ 04. What's **machine-to-machine (M2M)** authentication?

A **machine** here means a software service or application backend, not necessarily a physical computer. M2M authentication allows one application to call another service **without a human logging in during that exchange**.

Examples:

- An airline reservation backend requests a token to call a customer-support agent.
- A scheduled workflow calls an order-status API.
- A backend Lambda calls the protected ORBIT Harness.

In Mission 05, the **Lambda test function is the client**. It authenticates as a registered Cognito *app client*, not as Sarah or another airline customer.

## ❓ 05. What exactly is the OAuth **client** in this lab?

A **client** is the application requesting access to a protected service.

The word can be confusing because it does **not** mean "customer."

| Term | In Mission 05 |
|---|---|
| Client application | Lambda test function `orbit-MY_ID-jwt-test` |
| Client ID | Public identifier Cognito assigns to `orbit-MY_ID-m2m` |
| Client secret | Private credential that proves the client knows the secret |
| Authorization server | Cognito token service |
| Protected API | AgentCore Harness invocation endpoint |

The Lambda function contains the code that uses the client credentials; the **Cognito app client** is the registered identity and configuration for that software.

## ❓ 06. Why does the lab create a **Cognito user pool** if no users sign in?

Because a Cognito **user pool is also an OAuth/OIDC authorization server**. It can issue tokens to *applications*, not just human users.

The pool holds configuration for:

- The registered M2M app client.
- The app client's secret and permitted grant type.
- The custom scope that can be issued.
- The signing keys and issuer metadata needed for verification.

**Mission 05 has no human end-user accounts.** Creating a user pool does not mean someone must sign in through a login page.

## ❓ 07. What's the difference between a user pool and an identity pool?

| Cognito feature | Purpose | Used in this mission? |
|---|---|---|
| **User pool** | Authenticate users/applications and issue JWTs | **Yes** |
| **Identity pool** | Exchange supported identities for temporary AWS credentials | **No** |

The lab's Lambda is not exchanging a Cognito token for AWS credentials to call AgentCore. It sends the **JWT directly as a bearer token** to a JWT-protected endpoint.

## ❓ 08. What is the **client credentials grant**?

It is the OAuth grant type for a confidential client acting on its **own** behalf rather than on behalf of a signed-in human user.

The important HTTP request looks like this:

```http
POST /oauth2/token HTTP/1.1
Host: YOUR_DOMAIN.auth.us-west-2.amazoncognito.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic BASE64_OF_CLIENT_ID_COLON_CLIENT_SECRET

grant_type=client_credentials&scope=orbit-MY_ID-api%2Finvoke
```

Cognito verifies the app client and the requested scope, then returns something conceptually like:

```json
{
  "access_token": "eyJ...REDACTED...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Never copy a real client secret or live access token into notes, a screenshot, a chat, or a repository.** The examples above are placeholders, not usable credentials.

## ❓ 09. Where does Lambda get the client ID and password? Are they hardcoded?

In this lab, the Lambda test function reads a JSON object from **AWS Secrets Manager** at runtime.

It uses the `SECRET_ARN` environment variable to find the secret, then `boto3` to retrieve it:

```python
secret = json.loads(
    boto3.client("secretsmanager")
    .get_secret_value(SecretId=os.environ["SECRET_ARN"])["SecretString"]
)
```

The secret stores `client_id`, `client_secret`, `token_url`, and `scope`. The IAM role attached to **the test Lambda** needs permission to read *that specific secret*.

**Important:** The client ID is an identifier, not a password. The client secret is sensitive. The token URL and scope are configuration. For simplicity, the lab stores them together.

## ❓ 10. Does the app authenticate with Cognito **on every chatbot message**?

**In this lab, the Lambda requests a new token for each `good` test invocation.** That makes each test independent and easy to understand.

**In a production application, usually not.** A backend often caches a valid short-lived access token securely and reuses it until near expiration, then requests a new one. It must prevent accidental token sharing across security principals and handle refresh/rotation correctly.

Also distinguish **three events**:

1. Requesting a token from Cognito (not needed for every message if a usable token is cached).
2. Sending that bearer token with an agent request (required for **each** protected HTTP invocation).
3. AgentCore validating the incoming token (performed at the inbound boundary).

A user might send 10 chat messages while the backend uses the *same unexpired app token* for all 10 calls.

---

# PART II — READ A TOKEN LIKE A SECURITY ENGINEER

## ❓ 11. What's a JWT, and why is it three pieces separated by dots?

**JWT = JSON Web Token.** A typical *signed* JWT has this structure:

```text
HEADER.PAYLOAD.SIGNATURE

HEADER:    algorithm and key identifier (kid)
PAYLOAD:   claims: issuer, client, scope, expiry...
SIGNATURE: cryptographic proof that the token wasn't modified
```

The header and payload are Base64URL-encoded, **not encrypted**. Anyone who holds the token may be able to decode and read those claims. The signature prevents unauthorized alteration; it does not hide the contents.

### 🔎 JWT anatomy (fake claims, not a live token)

```json
{
  "iss": "https://cognito-idp.us-west-2.amazonaws.com/us-west-2_EXAMPLE",
  "client_id": "example-client-id",
  "scope": "orbit-s07x42-api/invoke",
  "token_use": "access",
  "exp": 1790000000
}
```

The values are illustrative. Do **not** create a token by typing JSON; AgentCore requires a valid signature from the approved issuer.

## ❓ 12. What does each important JWT claim mean?

| Claim | Meaning | Mission 05 question |
|---|---|---|
| `iss` | Issuer | Did the token come from **our** Cognito pool? |
| `client_id` | OAuth app that obtained the token | Is this our approved `orbit-MY_ID-m2m` client? |
| `scope` | Granted permission strings | Does it include `orbit-MY_ID-api/invoke`? |
| `exp` | Expiry time (Unix timestamp) | Is the token still valid? |
| `iat` | Issued-at time | When was it issued? |
| `token_use` | Kind of Cognito token | Is it an access token? |
| `aud` | Intended resource/API, when present | Is the token meant for this resource? |

**Important lab-specific nuance:** Cognito's **client-credentials M2M access tokens in this exercise do not use `aud`**, so leave the Harness's **Allowed audiences** setting unchecked. This is **not** a universal statement about all Cognito access tokens—Cognito supports an `aud` claim in certain user-based resource-binding flows. In this specific lab, `client_id` and `scope` are the relevant configured checks.

## ❓ 13. How does the agent know the token wasn't forged?

Cognito signs its token with a **private signing key**. AgentCore obtains the corresponding **public verification keys** through metadata published by the Cognito user pool.

An attacker can write their own payload claiming to have the `invoke` scope. But without the correct private key, they cannot produce a valid matching signature.

```mermaid
flowchart LR
    C[Cognito holds private signing key] -->|Signs JWT| Token[JWT]
    Token -->|Bearer token| A[AgentCore authorizer]
    Meta[Cognito public JWKS keys] -->|Verify signature| A
    A --> D{Signature and claims valid?}
    D -->|Yes| H[Invoke Harness]
    D -->|No| R[Reject]
    classDef good fill:#e4f5ef,stroke:#268362,color:#143b30
    classDef bad fill:#ffe7e7,stroke:#c74e4e,color:#5b2222
    class H good
    class R bad
```

AgentCore should verify **signature, issuer, expiry, and configured client/scope restrictions**. Decoding the JWT text yourself without signature verification proves nothing.

## ❓ 14. Why does the Harness need a **discovery URL**?

Because AgentCore needs to learn which issuer it should trust and where to fetch that issuer's public signing keys.

The lab's three URLs are **not interchangeable**:

| URL | Example pattern | Used for |
|---|---|---|
| **Token endpoint** | `https://YOUR_DOMAIN.auth.us-west-2.amazoncognito.com/oauth2/token` | Lambda **requests** a token |
| **Issuer URL** | `https://cognito-idp.us-west-2.amazonaws.com/USER_POOL_ID` | JWT `iss` identifies the issuing pool |
| **OIDC discovery URL** | `https://cognito-idp.us-west-2.amazonaws.com/USER_POOL_ID/.well-known/openid-configuration` | Harness **discovers** issuer/JWKS information |

The discovery document provides a `jwks_uri` that identifies the public signing-key endpoint. This is not the client secret and does not let anyone impersonate Cognito.

### 🧩 Spot the mistake

An engineer pastes the Cognito hosted-domain `/oauth2/token` URL into the **Harness Discovery URL** field. **Wrong:** that endpoint issues tokens; it does not serve OIDC discovery metadata.

## ❓ 15. What's a resource server in Cognito? Did we deploy another server?

**No additional compute server was deployed.** In the Cognito console, a *resource server registration* defines a namespace for API permissions called **scopes**.

You create:

```text
Resource server identifier: orbit-MY_ID-api
Scope name:                invoke
Fully qualified scope:     orbit-MY_ID-api/invoke
```

The *actual API being protected* is the AgentCore Harness endpoint.

The Cognito configuration says which access scopes can be issued; the AgentCore authorizer says which scopes an incoming token must have.

## ❓ 16. What is a **scope**, and is it an IAM permission?

A scope is an OAuth permission **string** agreed upon by the authorization server and the API.

Example:

```text
orbit-s07x42-api/invoke
```

It means, "This access token was granted permission to invoke the ORBIT training API," **provided the API validates it**.

It is **not** an AWS IAM action. It does not automatically grant `bedrock:*`, `lambda:InvokeFunction`, or `secretsmanager:GetSecretValue`.

| Kind of permission | Example | Checked by |
|---|---|---|
| OAuth scope | `orbit-s07x42-api/invoke` | JWT authorizer / application API |
| IAM action | `secretsmanager:GetSecretValue` | AWS IAM |
| Business rule | "Order belongs to authenticated customer" | Application / order tool |

## ❓ 17. If I define a scope, can any app client request it?

No. The scope must be **allowed on that app client** in Cognito. In Mission 05, you edit `orbit-MY_ID-m2m` and explicitly add the custom `invoke` scope.

Two permissions work together:

1. **Cognito issuance:** Is this app client allowed to *receive* the scope?
2. **AgentCore inbound check:** Does this token *contain* the required scope?

If the client requests a scope it cannot use, Cognito can reject the token request before AgentCore receives anything. The exact OAuth error (`invalid_scope`, `invalid_grant`, etc.) depends on the cause.

## ❓ 18. Why check **Allowed clients** and **Allowed scopes**? Isn't one enough?

They answer different questions:

- **Allowed clients:** Is the calling application one of our approved apps?
- **Allowed scopes:** Does the issued token include the specific permission needed for this endpoint?

A token could come from the right Cognito pool but belong to the wrong app client. Or it could come from the right app client but lack the necessary scope.

**Defense in depth:** configure both restrictions, exactly as the lab instructs.

## ❓ 19. What's the difference between an **access token**, **ID token**, and **refresh token**?

| Token | Main purpose | Mission 05 |
|---|---|---|
| **Access token** | Authorize calls to protected APIs | **Used**: bearer token sent to AgentCore |
| **ID token** | Convey information about a signed-in human identity in an OIDC login | Not used |
| **Refresh token** | Obtain new tokens after an interactive authorization flow | Not issued by this M2M client-credentials flow |

Sending the wrong token type is a common API-security mistake. For this lab, use the **`access_token`** returned by `/oauth2/token`.

---

# PART III — FOLLOW THE LAB'S REAL NETWORK CALLS

## ❓ 20. Why are we creating a **second Harness** instead of changing the Mission 04 one?

The first Harness is your IAM-authenticated agent with Gateway tools and memory. The second is a **separate, minimal JWT-protected agent** used to isolate the authentication experiment.

| | Mission 04 Harness | Mission 05 Harness |
|---|---|---|
| Inbound authentication | AWS IAM / signed AWS requests | OAuth bearer JWT |
| Gateway/order lookup/KB | Connected | **Not connected in this lab** |
| AgentCore Memory | Enabled | Disabled for this test |
| Main question | "Can the agent use tools correctly?" | "Who may invoke the agent?" |

**Do not change the Mission 04 Harness to JWT** just to run Mission 05. You would break the earlier exercise and blur the security comparison.

## ❓ 21. Why does the secure Harness still need an IAM role if we use JWTs?

Because **JWT authenticates the incoming caller**. Once a valid request reaches the agent, the agent still needs AWS permissions to call Bedrock and write applicable telemetry.

```text
External Lambda test client
        |
        | Bearer JWT
        v
AgentCore inbound authorization   <-- Cognito JWT identity check
        |
        v
Protected Harness / Runtime
        |
        | AWS execution role
        v
Bedrock model inference           <-- AWS IAM permission check
```

A successful JWT does **not** give the caller the Harness execution role's AWS permissions.

## ❓ 22. How many IAM identities are involved in Mission 05?

At least these important ones:

| Principal | What it does | Typical permission needed |
|---|---|---|
| **Your AWS console identity** | Creates Cognito, secrets, Harness, Lambda, and policies | Administrative permissions required for setup |
| **Lambda test execution role** | Reads its secret and writes logs | `secretsmanager:GetSecretValue` on one ARN; CloudWatch logs |
| **Harness execution role** | Calls Bedrock model and supports runtime execution | Model/runtime permissions scoped to agent needs |
| **Cognito M2M app client** | Obtains JWT via OAuth | Client secret and permitted OAuth scopes; **not an IAM role** |

**Not present in the active Mission 05 request path:** the Gateway execution role and the order-lookup Lambda's execution role from Mission 04.

## ❓ 23. Why doesn't the test Lambda need `bedrock-agentcore:InvokeHarness`?

Because the code in Mission 05 calls the **JWT-protected HTTPS endpoint** with:

```http
Authorization: Bearer ACCESS_TOKEN
```

For this inbound mode, the JWT authorizer decides whether the request may enter. By contrast, an **IAM-protected** Harness is invoked using AWS-signed requests and requires the appropriate AWS IAM invocation permissions.

**Same API concept; different inbound authentication mechanism.** The AWS execution role of the *running agent* still applies after entry.

## ❓ 24. What happens, step by step, when we run `{"mode":"good"}`?

```mermaid
sequenceDiagram
    autonumber
    participant Test as Lambda test function
    participant SM as Secrets Manager
    participant Cog as Cognito token endpoint
    participant Gate as AgentCore JWT authorizer
    participant Harness as Protected Harness / Runtime
    participant FM as Bedrock model
    Test->>SM: GetSecretValue (IAM-signed AWS API call)
    SM-->>Test: M2M client ID, secret, scope, token URL
    Test->>Cog: POST /oauth2/token, client_credentials, scope
    Cog-->>Test: HTTP 200 + access JWT
    Test->>Gate: POST /harnesses/invoke, Bearer JWT
    Gate->>Gate: Verify signature, issuer, client, scope, exp
    Gate->>Harness: Allow request
    Harness->>FM: Generate answer using Harness IAM role
    FM-->>Harness: Model output
    Harness-->>Test: Streamed HTTP 200 response
    Test-->>Test: Return sanitized status and answer preview
```

Expected lab evidence:

```json
{
  "mode": "good",
  "token_status": 200,
  "runtime_http_status": 200,
  "body_preview": "Least privilege is ..."
}
```

These values describe the **successful path observed in the supplied lab**. They are illustrative until you run the test in your own account.

## ❓ 25. Why do we use Lambda as a **test client**? Isn't Lambda usually a backend tool?

Lambda is simply **code that AWS executes on demand**. What the code does determines its role.

- In Mission 04, Lambda **was the business tool backend**: `lookup_order`.
- In Mission 05, Lambda **is the API client**: it requests a token and invokes the agent.

The same AWS service can be on completely different sides of an architecture depending on the application.

**Think:** *What HTTP request or event does this function send or receive?* That tells you whether it acts as a client, a server-side handler, or a backend tool.

## ❓ 26. Why does the test Lambda need an `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` header?

The Harness invocation is associated with a **Runtime session**. The lab supplies a unique session ID for each test invocation and uses a UUID so test runs do not accidentally share conversation state.

A simplified excerpt from the lab:

```python
headers = {
    "Content-Type": "application/json",
    "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id":
        f"orbit-test-{uuid.uuid4()}",
}
```

The lab notes a **minimum session ID length of 33 characters** for this endpoint. The prefix plus generated UUID meets that requirement.

**Session ID ≠ authentication token.** Knowing a session ID is not a substitute for presenting an authorized JWT. In production, carefully control who can choose or reuse session identifiers so one customer's conversation is never exposed to another.

## ❓ 27. Why is the successful response a **stream** instead of one JSON answer?

Agent frameworks often stream response events so the caller can consume generated output incrementally and may observe structured execution events.

The sample Lambda uses a small demonstration parser to extract text from the response body and return a preview. In a production client, use an appropriate SDK or a robust stream/event parser rather than assuming a regular expression can parse every possible stream frame.

An HTTP `200` says the request was accepted and responded to; it does **not** by itself prove the *answer* is correct, authorized for a business operation, or supported by policy evidence.

## ❓ 28. What happens with `{"mode":"missing"}`?

The Lambda skips token issuance and sends the invocation **without an Authorization header**.

```mermaid
sequenceDiagram
    participant L as Test Lambda
    participant A as AgentCore inbound authorizer
    participant H as Protected Harness
    L->>A: Invoke with no Bearer token
    A-->>L: Reject (403 in verified lab)
    Note over H: Not invoked
```

In the supplied lab, the result was `runtime_http_status: 403` with `Missing Authentication Token`.

**Be precise:** depending on the front-end service and failure type, an authentication failure may appear as `401` or `403`. Don't memorize one numeric code as the universal rule. The key evidence is **rejection before model inference**.

## ❓ 29. What happens with `{"mode":"forged"}`?

The Lambda sends this intentionally invalid token:

```http
Authorization: Bearer this.is.not.a.valid.jwt
```

It doesn't have a valid token structure/signature from your Cognito pool. The AgentCore authorizer rejects it **before the model runs**.

The lab observed `403` and a message explaining that the token could not be parsed. Other invalid-token formats may fail at different verification stages.

## ❓ 30. What if the JWT is valid, but from the **wrong Cognito pool**?

**Reject.** The token's issuer (`iss`) and signing keys must match the configured issuer reached through the Harness's discovery URL.

A correctly signed token from **Pool B** isn't valid for a Harness that trusts only **Pool A**.

This is why validating *only* the shape of a JWT, or only its scope string, is dangerously insufficient.

## ❓ 31. What if the token is valid but belongs to the **wrong app client**?

**Reject**, because Mission 05 configures a specific value under **Allowed clients**.

For example, an unrelated app client in the same Cognito pool should not gain access merely because its token was issued by the correct pool.

## ❓ 32. What if the client is correct but the **scope is missing**?

**Reject** under the lab's Harness configuration, because `orbit-MY_ID-api/invoke` is required.

There are two possible failure stages:

- Cognito might reject the **token request** if the app requests a scope it has not been granted.
- AgentCore should reject an **otherwise valid token** if its `scope` claim lacks the required `invoke` permission.

These are different errors in different services.

## ❓ 33. What if the JWT has expired? Can the agent still use it?

No. Its `exp` claim tells the authorizer when the token stops being valid. A valid signature **does not override expiration**.

When the token expires, the client must obtain a new valid one. In this client-credentials M2M flow, the application can request another token using its client credentials; there is no M2M refresh token in this lab.

---

# PART IV — THE SECRET, THE ROLES, AND THE REAL SECURITY BOUNDARIES

## ❓ 34. Why Secrets Manager instead of putting the client secret in a Lambda environment variable?

Secrets Manager centralizes secret storage, encryption at rest, IAM-protected retrieval, auditing, and rotation workflows. It reduces the temptation to copy sensitive credentials into deployment settings, code, screenshots, or test events.

The lab keeps **configuration identifiers** in environment variables:

```text
SECRET_ARN
HARNESS_ARN
AWS_REGION_FOR_LAB
```

The secret **value** remains in Secrets Manager and is read at runtime.

Secrets Manager does **not** magically protect a secret once a permitted process reads it. Anyone who can edit or execute sufficiently privileged code in the test Lambda may be able to misuse that application's credential. Protect access to the Lambda and its IAM role.

## ❓ 35. Why does the Secrets Manager ARN end in six random characters?

AWS appends a random suffix to distinguish the secret resource reliably. Its full ARN might look like:

```text
arn:aws:secretsmanager:us-west-2:111122223333:secret:orbit/s07x42/cognito-m2m-AbCdEf
```

The example is synthetic. When you write an IAM policy, you can use the **exact ARN** you copied, which is preferable to a broad wildcard.

The lab also shows a narrowly scoped pattern ending in `-*` to match that suffix. Neither choice should grant access to unrelated secrets.

## ❓ 36. What's a role's **trust policy** vs. its **permissions policy**?

- **Trust policy:** Who may assume this IAM role?
- **Permissions policy:** Once the role is assumed, what may it do?

For the test Lambda:

```text
Lambda service assumes the Lambda execution role  <-- Trust relationship
                    |
                    v
Role may call GetSecretValue on its own secret      <-- Permissions policy
```

Giving the Harness execution role permission to read the Lambda's secret would be the **wrong fix**. The Harness doesn't request the OAuth token in this lab—the test Lambda does.

## ❓ 37. Why is `secretsmanager:GetSecretValue` limited to one ARN?

Because of the **principle of least privilege**: give each component only the access necessary for its task.

A policy for the test Lambda can be limited to one secret:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:us-west-2:111122223333:secret:orbit/s07x42/cognito-m2m-AbCdEf"
    }
  ]
}
```

Use **your own exact ARN**, not the example account ID or example secret name. Your Lambda's AWS role shouldn't be able to retrieve classmates' secrets.

## ❓ 38. Could someone who steals a valid **bearer token** impersonate the app?

**Yes, until that token expires or is otherwise rejected.** A bearer token grants access to whoever possesses it; the token is not a password, but it is still a **sensitive credential**.

Keep it out of logs, browser console output, error messages, URLs, and screenshots. Use HTTPS. Prefer short-lived tokens and narrow scopes. For higher-risk systems, consider stronger sender-constrained token mechanisms where supported, and monitor suspicious use.

**An M2M client secret is even more consequential:** someone holding it may be able to request *fresh* tokens until you rotate or disable it.

## ❓ 39. Does a valid token make an agent safe from prompt injection?

No. Token verification answers **who may enter**. Prompt injection attacks try to manipulate **what the agent does after entering**.

A valid client might send a malicious prompt, or a malicious instruction could be hidden in retrieved policy content. Protect the system separately through input validation, least-privilege tools, tool-level authorization, confirmation for sensitive actions, and secure handling of untrusted retrieved content.

**Authentication is a gate, not a guarantee that the request is safe.**

## ❓ 40. Does this lab authorize a real customer to view or refund an order?

**No.** The M2M token identifies an **application**, not the human currently using it. Also, the Mission 05 Harness has **no order lookup or refund tool attached**.

Even in a production system with a valid app token, the order tool must independently check customer identity, order ownership, permitted actions, fraud controls, and any required approval flow.

---

# PART V — WHAT WOULD A REAL AIRLINE CHATBOT LOOK LIKE?

## ❓ 41. If anybody can open an airline website, how can a chatbot be authenticated?

Distinguish **guest chat** from **account-specific actions**.

- A guest can ask public questions (e.g., baggage policies) without being signed in, subject to abuse controls.
- A customer asking about their personal booking should sign in or complete an appropriate account/booking verification flow.
- The **website backend** can authenticate to internal agent services with its own M2M credentials, while separately carrying verified context about the signed-in customer.

```mermaid
flowchart TB
    Browser[Customer browser] --> Web[Airline backend / API layer]
    Web --> Guest{Public or personal request?}
    Guest -->|Public policy| Public[Guest-safe knowledge responses / With rate limits and abuse protection]
    Guest -->|Personal booking| Login[Human sign-in / verified identity]
    Login --> UserAuth[User token and booking authorization checks]
    UserAuth --> App[Backend calls protected AgentCore agent]
    App --> API[Read-only order API or authorized workflow]
    API --> Check[Verify user may access this specific booking]
    Check --> Result[Return permitted information only]
    classDef security fill:#e4f5ef,stroke:#268362,color:#143b30
    class Login,UserAuth,Check security
```

**Critical caveat:** An application-level M2M token alone cannot establish **which person** the application is serving. For customer-specific actions, use verified user identity and enforce per-user authorization in your backend/tools. Never blindly trust a customer ID typed into the prompt.

## ❓ 42. Why not put the Cognito M2M client ID and secret directly in the website's JavaScript?

Because **browser JavaScript is public to whoever visits the website**. Users can inspect its code and network requests. A client secret placed there is no longer secret.

Use a confidential backend for M2M secrets. For interactive human login, use a browser-appropriate authorization-code flow with **PKCE** (and proper session handling) rather than embedding an M2M client secret in a SPA.

## ❓ 43. What about spam, bot attacks, and someone sending a million chat messages?

A valid application token is not rate limiting, and a guest web page is often publicly reachable. Production systems may need:

- AWS WAF/bot controls and throttling at the public API/edge.
- Per-user, per-IP, per-app, and per-session quotas where appropriate.
- Request size limits, concurrency limits, and token/cost budgets.
- Sensible timeout, circuit-breaking, and abuse detection.
- Authentication gates before exposing personal data or sensitive actions.
- Monitoring for spikes, repeated failures, and unusual tool use.

The lab specifically tests **identity acceptance/rejection**, not a complete production anti-abuse perimeter.

## ❓ 44. Is Cognito required for *every* AgentCore deployment?

No. AgentCore supports multiple inbound identity patterns. In these labs:

- **Mission 04:** IAM-based invocation is appropriate for an AWS-authenticated exercise.
- **Mission 05:** Cognito-issued JWT is a practical way to learn an OAuth-based inbound boundary.

A production system may use a compatible enterprise identity provider, JWT or IAM inbound patterns, and other security layers depending on who the callers are. Cognito is **one identity provider**, not the definition of OAuth.

## ❓ 45. What if the agent later needs to call an external service requiring OAuth?

That's **outbound authorization**, not the inbound JWT check.

```text
INBOUND: External application --> AgentCore Harness
         Who may invoke my agent?

OUTBOUND: AgentCore Harness --> Downstream API
          What credentials may my agent use to call that service?
```

AgentCore Identity provides mechanisms for managing outbound credentials. But you must still design scopes, user delegation, token storage, and authorization for the downstream service.

An inbound JWT does not automatically authorize the agent to call every other API.

---

# PART VI — DEBUG LIKE A SECURITY ENGINEER

## 🧰 Failure triage: which service said "No"?

```mermaid
flowchart TD
    Start[Run Lambda test] --> Secret{Read secret succeeded?}
    Secret -->|No| IAM[Check Lambda role's GetSecretValue policy / and the exact secret ARN]
    Secret -->|Yes| Tok{Requested Cognito token?}
    Tok -->|No: missing/forged test| Inv[Call AgentCore endpoint]
    Tok -->|Yes| TokenOK{Cognito returned 200?}
    TokenOK -->|No| Cog[Check client ID/secret, grant type, / token URL, and assigned scope]
    TokenOK -->|Yes| Inv
    Inv --> Auth{AgentCore accepted?}
    Auth -->|401 or 403| JWT[Check discovery URL, issuer, / client ID, scope, expiry, JWT validity]
    Auth -->|200| Success[Success: inspect model answer]
    Auth -->|5xx| Runtime[Check Harness Ready state, / model permissions, and CloudWatch logs]
    classDef error fill:#ffe7e7,stroke:#c74e4e,color:#5b2222
    class IAM,Cog,JWT,Runtime error
```

## ❓ 46. What's the difference between `token_status` and `runtime_http_status`?

- **`token_status`:** The result of contacting **Cognito** to obtain an access token (only attempted in the `good` mode).
- **`runtime_http_status`:** The result of contacting **AgentCore** with or without a token.

| Observed combination | Likely meaning |
|---|---|
| `token_status: 200`, `runtime_http_status: 200` | Cognito issued token; AgentCore accepted request; agent responded |
| `token_status: 400`, no runtime status | Cognito rejected token request; invocation wasn't attempted |
| `token_status: 200`, runtime `401/403` | Token issued, but AgentCore's inbound validation rejected it |
| `token_status: 200`, runtime `5xx` | Authentication may have succeeded; downstream runtime/model failure needs investigation |
| `token_status: null`, runtime `403` for `missing` | Expected missing-token rejection in the supplied lab |

Don't try to fix a **Cognito token issuance** failure by adding more IAM permissions to the Harness role.

## ❓ 47. Why does Cognito say `invalid_client`?

It usually means the **client couldn't authenticate successfully** to the token endpoint. Check that you copied the correct client ID and client secret from the **same** user pool and that the app client has the client-credentials grant enabled.

Also confirm you used the **hosted Cognito domain `/oauth2/token`**, not the OIDC discovery URL.

## ❓ 48. Why does Cognito reject my scope?

Check these items:

1. The resource server identifier is `orbit-MY_ID-api`.
2. The custom scope is named `invoke`.
3. The full requested scope is `orbit-MY_ID-api/invoke`.
4. The **app client** has that custom scope enabled.

Different invalid configurations can generate different OAuth errors. The lab describes both `invalid_scope` and scope-assignment failures; do not assume one error string always identifies the exact cause.

## ❓ 49. What if token issuance works, but the Harness still returns `403`?

Inspect **Harness → Inbound Auth** and compare the settings with the token you intended to issue:

- Correct Cognito **discovery URL** (pool, not hosted-domain token URL).
- **Allowed clients** matches the registered app client ID.
- **Allowed scopes** includes the fully qualified invoke scope.
- **Allowed audiences** is *not* configured for this lab's M2M flow.
- Token hasn't expired and is a real signed access JWT.

This is where inspecting **non-sensitive metadata** and CloudWatch logs helps. Avoid logging raw bearer tokens or client secrets.

## ❓ 50. Why does the Harness Playground sometimes refuse to test this JWT agent?

The supplied lab notes that the **Harness Playground uses your console IAM identity** for its requests. A Harness configured to require JWT bearer authentication may therefore be incompatible with the Playground's usual invocation path.

**Do not disable JWT authentication to make the Playground work.** The lab provides an AWS Lambda test client precisely so you can send requests with the correct bearer token.

## ❓ 51. What do CloudWatch and OpenTelemetry show here?

CloudWatch logs and AgentCore observability help you troubleshoot events *after* or around the invocation boundary: agent startup, model calls, latency, errors, and traces (depending on the telemetry configured).

Recall **OTEL/OpenTelemetry** from Mission 04: it standardizes instrumentation for traces, metrics, and logs. CloudWatch is an AWS destination to inspect that telemetry.

**Security rule:** Useful log entry = request ID, timestamp, failure category, status. Dangerous log entry = unredacted JWT, client secret, full sensitive prompts, or customer records.

---

# PART VII — DEFENSE-IN-DEPTH CHALLENGES 🧪

> **How to use:** Answer each challenge before reading its explanation. Unless explicitly instructed otherwise, these are **reasoning exercises**, not permission to weaken your shared account or change classmates' resources.

### 🧩 Challenge 1 — The confused role

**Scenario:** Your test Lambda fails at `GetSecretValue`. A teammate suggests attaching `secretsmanager:*` to the **Harness execution role**.

**Answer:** Wrong principal. The **test Lambda execution role** reads the secret, not the Harness. Grant `secretsmanager:GetSecretValue` for the exact secret ARN to the Lambda role. Do not grant broad wildcard permissions.

### 🧩 Challenge 2 — The impostor from another pool

**Scenario:** Pool B signs a perfectly valid JWT containing `orbit-s07x42-api/invoke`. Your Harness trusts Pool A.

**Answer:** Reject it. The issuer and signing key are wrong for the configured trust boundary; matching a scope string is not enough.

### 🧩 Challenge 3 — A scope is not an AWS role

**Scenario:** An app receives a token with `orbit-s07x42-api/invoke` and then tries to call `secretsmanager:GetSecretValue` using that token.

**Answer:** The OAuth scope is not an IAM credential. The app does not gain that AWS permission from the JWT.

### 🧩 Challenge 4 — Signed, but expired

**Scenario:** A correctly signed access token expired five minutes ago.

**Answer:** Reject it. Signature validity does not cancel the `exp` requirement. Obtain a new token using an authorized flow.

### 🧩 Challenge 5 — The dangerous customer mix-up

**Scenario:** The airline backend holds a valid M2M token. A chat request says, "I am Alice. Show me booking 1234," but the booking belongs to Bob.

**Answer:** Reject the account-specific operation unless the caller's **verified user identity and booking authorization** permit it. An app-level M2M token cannot prove the prompt's customer identity.

### 🧩 Challenge 6 — Public browser secret

**Scenario:** A developer copies the M2M client secret into a React app so the browser can ask Cognito for tokens directly.

**Answer:** Insecure. Browser code cannot protect a confidential client secret. Move M2M credential handling to a trusted backend; use an appropriate human-login flow for end users.

### 🧩 Challenge 7 — The misleading HTTP 200

**Scenario:** `runtime_http_status` is `200`, and the model says, "Your refund was processed."

**Answer:** That is **not proof of a refund**. An HTTP success means the agent invocation was accepted and produced a response. No refund tool exists in Mission 05, and Mission 04's order tool was read-only.

### 🧩 Challenge 8 — The `aud` trap

**Scenario:** A teammate pastes the app client ID into **Allowed audiences** and removes **Allowed clients**.

**Answer:** Wrong claim for the supplied M2M setup. The client ID appears in `client_id`, not in an `aud` claim in this flow. Leave `aud` unchecked and configure **Allowed clients**.

### 🧩 Challenge 9 — Why not reuse a JWT forever?

**Scenario:** An app caches a token and ignores its expiry time because token requests cost money.

**Answer:** Unsafe and unreliable. Cache only within its valid lifetime, keep it protected, and obtain a new token before or when it expires. Also consider revocation response and credential rotation policies.

### 🧩 Challenge 10 — One role to rule them all?

**Scenario:** A developer puts AdministratorAccess on the Lambda and Harness roles so all lab calls succeed.

**Answer:** It hides the very boundaries Mission 05 is designed to teach. The Lambda needs narrow secret access; the Harness needs narrowly scoped model/runtime access; downstream tools, if added later, require their own boundaries.

---

# PART VIII — THREE MISSIONS, ONE SECURITY ARCHITECTURE

## 🎓 How Missions 03, 04, and 05 fit together

```mermaid
flowchart TB
    M3[Mission 03 / Knowledge Base / Verified policy evidence]
    M4[Mission 04 / Harness + Runtime / Agent loop and tool calls]
    GW[AgentCore Gateway / MCP tools]
    L[Order lookup Lambda / Read-only]
    M5[Mission 05 / Cognito + JWT / Protected invocation]
    M4 --> GW
    GW --> M3
    GW --> L
    M5 -. Demonstrates how to secure inbound access to a separate Harness .-> M4
    classDef data fill:#e4f5fb,stroke:#348ab1,color:#153746
    classDef agent fill:#fff0dc,stroke:#c98827,color:#48331d
    classDef security fill:#e4f5ef,stroke:#268362,color:#143b30
    class M3,L data
    class M4,GW agent
    class M5 security
```

**The dotted line is conceptual.** Mission 05 does **not** modify or secure the already-created Mission 04 Harness. It creates a new protected Harness to demonstrate a different inbound mode. In a later application design, you could combine authentication and tools, but every permission boundary would still need to be checked.

### 🧠 One-page memory map

| Component | Mental model | What it is **not** |
|---|---|---|
| Cognito user pool | Trusted issuer / OAuth server | Your agent or knowledge base |
| Cognito app client | Registered software identity | A human customer |
| Client ID | Application identifier | A secret |
| Client secret | Confidential credential | A JWT |
| OAuth scope | Permission string carried in token | An IAM policy |
| JWT access token | Signed, time-limited API credential | Proof of authorization for every business action |
| OIDC discovery URL | Metadata to find issuer and public keys | Token-issuing endpoint |
| Secrets Manager | Protected credential store | The token-issuing service |
| Lambda test client | Backend application making calls | The agent's tool backend from Mission 04 |
| Harness JWT authorizer | Inbound gatekeeper | Business-order authorization logic |
| Harness execution role | AWS permissions for running agent | The identity of the external caller |
| CloudWatch | Logs, metrics, traces | Automatic proof that all actions were safe |

## 🏁 The 60-second explanation

> "In Mission 05, our Lambda application reads its Cognito client credentials from Secrets Manager, then uses the OAuth client-credentials flow to obtain a signed access JWT. It sends that token to a separate AgentCore Harness configured with Cognito's discovery URL and checks for the correct client and scope. The authorizer rejects missing or forged tokens before the model runs. Once admitted, the agent uses its own IAM execution role for model calls. The app token is not a customer's identity and doesn't grant broad tool or AWS permissions."

### ✅ Mission 05 readiness check

- [ ] I can explain **authentication vs. authorization** with an ORBIT example.
- [ ] I can identify the **client, authorization server, protected API, and scope** in this lab.
- [ ] I can explain why the **client-credentials grant has no human sign-in**.
- [ ] I know the difference between the **token endpoint, issuer URL, discovery URL, and JWKS URL**.
- [ ] I can explain `iss`, `client_id`, `scope`, `exp`, and why we leave **Allowed audiences** off **for this M2M lab**.
- [ ] I know why **JWT claims can be decoded but must not be trusted without signature verification**.
- [ ] I can trace the three test results: `good`, `missing`, and `forged`.
- [ ] I know which **IAM role** needs access to Secrets Manager.
- [ ] I can distinguish **JWT inbound authentication** from **Harness outbound IAM permissions**.
- [ ] I know why an app M2M identity **does not prove who the human customer is**.
- [ ] I can explain why **scopes, tool authorization, and business authorization are separate**.
- [ ] I never copy live tokens, secrets, or sensitive customer data into examples or logs.

---

## 📚 Reference shelf — official AWS documentation

These links are for understanding the services and checking current behavior. Console names and pricing can change; the *hands-on steps* remain in the accompanying Mission 05 lab guide.

- [Amazon Cognito: Token endpoint](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html)
- [Amazon Cognito: Scopes, M2M, and resource servers](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html)
- [Amazon Cognito: Understanding access tokens](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-access-token.html)
- [Amazon Cognito: App clients](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html)
- [AgentCore: Harness security and access controls](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-security.html)
- [AgentCore: Inbound JWT authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html)
- [AWS Secrets Manager: User Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [AWS IAM: Roles and temporary credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [OpenTelemetry: Observability fundamentals](https://opentelemetry.io/docs/concepts/observability-primer/)

**Source fidelity note:** The specific ORBIT names, three Lambda test modes, selected model, configuration decisions, observed HTTP statuses, and exact console lab sequence above come from *Mission 05 — Identity Lockdown* (verified by its author October 20, 2026). Broader production architecture advice, token-handling cautions, and the nuance about optional `aud` claims in other Cognito flows are additional context informed by the AWS documentation above. All URLs and ARNs labeled "example" or "illustrative" must be replaced with values from your own AWS account when needed.

---

*End of Mission 05 Companion Field Guide · 🔐 Identity verified. Least privilege still matters.*
