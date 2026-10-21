# 🛰️ ORBIT FIELD ACADEMY
## MISSION 05 · IDENTITY LOCKDOWN
### Cognito, OAuth, JWT, IAM, Secrets Manager, and a JWT-protected AgentCore agent

> **LEVEL:** Security Engineer · **TARGET TIME:** 100–130 minutes · **ENVIRONMENT:** AWS Management Console only
> **THE STORY:** ORBIT's AI assistant can answer requests, but *who* is allowed to invoke it? Can a stranger pass any token? Can an authenticated app claim permission to change an order? You will build a real Cognito-backed perimeter around a second agent and prove it with successful **and** rejected requests sent from **AWS Lambda in the AWS Console**.

**Nothing runs on your laptop.** This mission uses Amazon Cognito, AgentCore Harness, AWS Secrets Manager, IAM, the Lambda browser editor and Test tab, and CloudWatch. There is no CLI, terminal, browser plug-in, or external HTTP tool.

> [!NOTE]
> **Verified against the live AWS Console on October 20, 2026** in US West (Oregon) with the example ID `s07x42`. Two things changed since earlier versions of this mission: Cognito **no longer charges per machine-to-machine app client** (only per token request, since November 2025), and the protected agent is an AgentCore **Harness** rather than a code-deployed runtime, so no code packaging is needed.

---

## 📖 Before you start: the concepts in this mission

### Authentication vs. authorization

- **Authentication** answers *who are you?* A caller proves its identity, for example by presenting a signed token.
- **Authorization** answers *what may you do?* Even an authenticated app must still be allowed to call *this* agent, with *this* scope, and the agent's tools must still decide whether *this* order may be touched.

### OAuth 2.0 in two minutes

OAuth 2.0 is the standard that lets an application obtain an **access token** from an **authorization server** and present it to a **resource server** (an API). Four roles matter today:

| OAuth role | Today it is… |
|---|---|
| **Client** | Your Lambda test function, identified by an **app client ID** and a **client secret** |
| **Authorization server** | Your Amazon Cognito **user pool** and its **token endpoint** |
| **Resource server** | A name you register in Cognito (`orbit-MY_ID-api`) that owns one or more **scopes** |
| **Scope** | A permission string in the token, such as `orbit-MY_ID-api/invoke`, that the API checks |

The **client credentials grant** is the OAuth flow for **machine-to-machine (M2M)** calls: no human signs in; the client sends its ID and secret and gets a token with custom scopes. A human user would use the **authorization code** grant (with PKCE) instead. The two flows are not interchangeable, but the token validation you configure today is the same for both.

### JSON Web Tokens (JWT)

An access token from Cognito is a **JWT**: three Base64URL strings separated by dots (`header.payload.signature`). The payload carries **claims**:

| Claim | Meaning | Where you will see it |
|---|---|---|
| `iss` | Issuer: the exact user pool URL | Must match the pool behind your **Discovery URL** |
| `client_id` | Which app client requested the token | **Allowed clients** on the harness |
| `scope` | Space-separated scopes granted | **Allowed scopes** on the harness |
| `exp` | Expiry time (Cognito default 60 minutes) | Expired tokens are rejected |
| `aud` | Audience: which API the token is for | Cognito **access tokens do not carry `aud`**; do not configure *Allowed audiences* |

The **signature** proves Cognito issued the token and nobody altered it. AgentCore verifies it by fetching Cognito's public keys through the pool's **OpenID Connect discovery URL** (`…/.well-known/openid-configuration`), which is why that URL is the first thing you configure.

### The AWS services

- **Amazon Cognito user pool:** an identity store plus an OAuth/OIDC authorization server. Today it will hold no human users at all, just one M2M **app client**, one **resource server**, and a **domain** (the hostname of its token endpoint).
- **AWS Secrets Manager:** stores secret values (your client secret) encrypted, with its own IAM access control. Code refers to a secret by **ARN** and reads the value at run time, so the secret never appears in source code, environment variables, screenshots, or logs.
- **Lambda environment variables:** plain configuration (ARNs, Region names) that your function reads with `os.environ`. Configuration is not secret; the client secret is.
- **AgentCore inbound auth:** every harness and gateway has a gatekeeper. *Use IAM permissions* (Mission 04) accepts AWS-signed requests. *Use JSON Web Tokens* accepts a `Bearer` token that matches the discovery URL, allowed clients, and allowed scopes you configure.

### HTTP status codes you will read today

`200` the request was accepted and answered · `400` the token request itself was malformed (wrong scope, wrong client) · `401`/`403` the agent refused the caller (missing, forged, expired, or wrong-client token) · `5xx` something failed *after* authorization, inside the agent.

```mermaid
sequenceDiagram
    participant Tester as AWS Lambda test client
    participant Cognito as Amazon Cognito
    participant Gate as AgentCore JWT authorizer
    participant Harness as AgentCore Harness
    participant Bedrock as Bedrock foundation model
    Tester->>Cognito: client_id + client_secret + scope
    Cognito-->>Tester: Signed access JWT
    Tester->>Gate: POST /harnesses/invoke + Bearer JWT
    Gate->>Gate: Verify signature, issuer, client_id, scope, expiry
    alt permitted
        Gate->>Harness: Authorized invocation
        Harness->>Bedrock: Model inference (using the harness execution role)
        Bedrock-->>Harness: Response
        Harness-->>Tester: Streamed answer (HTTP 200)
    else missing, forged or wrong token
        Gate-->>Tester: HTTP 403 with a reason
    end
```

### Know the difference

| Control | Protects | Does **not** automatically protect |
|---|---|---|
| **IAM for you in the console** | Who may create or change resources | App end users |
| **Harness execution role (IAM)** | Which AWS APIs the agent process may call | Whether an external JWT is valid |
| **Cognito user pool and M2M app client** | Token issuance, client identity, scopes | IAM access to Bedrock or Secrets Manager |
| **AgentCore inbound JWT authorizer** | Which tokens may invoke the harness | Business rules inside each tool |
| **Gateway and target roles** | Which tools may be invoked | Whether *this* customer may touch *this* order |

---

## 🪪 Phase 0 — Shared-account and pricing preflight (5 minutes)

1. Use the **same `MY_ID` and Region** as Missions 03–04.
2. **Cost facts (verified October 2026):** Cognito bills **$0.00225 per successful M2M token request** with no free tier and **no charge per app client**. You will make roughly 3–10 token requests. Secrets Manager bills **$0.40 per secret per month**, prorated hourly. The protected harness bills like Mission 04. The whole mission is a few cents per participant; the only thing worth deleting promptly afterwards is the secret.
3. Create a **new** harness for this mission. Do not switch the Mission 04 harness to JWT; keeping both lets you compare them and avoids breaking the Mission 04 gateway test.

| Resource | Name you will type | Record ID / ARN |
|---|---|---|
| Cognito app client (created first, by the wizard) | `orbit-MY_ID-m2m` | app client ID |
| Cognito user pool (renamed after the wizard) | `orbit-MY_ID-pool` | user pool ID, OIDC configuration URL |
| Cognito resource server and scope | `orbit-MY_ID-api` with scope `invoke` | full scope `orbit-MY_ID-api/invoke` |
| Cognito domain | created automatically, unique per pool | `https://….auth.us-west-2.amazoncognito.com` |
| Secrets Manager secret | `orbit/MY_ID/cognito-m2m` | secret ARN |
| JWT-protected harness | `orbit_MY_ID_secure` | harness ARN |
| Harness execution role | `orbit-MY_ID-secure-role` | |
| Lambda test client | `orbit-MY_ID-jwt-test` | |
| Inline policy on the Lambda role | `orbit-MY_ID-read-secret` | |

---

## 🧩 Phase 1 — Sketch the authentication rules (5 minutes)

Before clicking anything, write the rule set you intend to enforce:

- `iss` must be **my** user pool.
- `client_id` must be **my** app client.
- `scope` must include `orbit-MY_ID-api/invoke`.
- `exp` must be in the future.
- `aud` is **not** checked (Cognito access tokens do not carry it).

Decide which rejection you will prove first: **missing token** and **forged token** are the two you will run; wrong client and expired token are analysed in Phase 6.

**Checkpoint:** If a valid app sends a Cognito JWT to AgentCore, does the caller gain `secretsmanager:*` or `bedrock:*` privileges? **No.** The agent's AWS permissions come from its execution role and are enforced separately.

---

## 🏷️ Phase 2 — Set up Cognito with a unique app identity (25 minutes)

### Step A: Create the user pool and M2M app client in one wizard

1. Console search → **Cognito** → **User pools** → **Create user pool**. The wizard is titled *Set up resources for your application*.
2. Under **Application type** choose **Machine-to-machine application**. The other options (web, SPA, mobile) are for human sign-in and add fields you do not need.
3. **Name your application:** `orbit-MY_ID-m2m`.

   ![Cognito wizard with Machine-to-machine application selected and the app named](images/m05-01-cognito-wizard-m2m.png)

4. Choose **Create user directory**. The confirmation page shows that **both** an app client and a user pool were created. The pool gets an automatic name such as *User pool - edwkah*; you will rename it next. Read the page: Cognito also created a **default resource server** (`default-m2m-resource-server-xxxxxx/read`) and shows a sample `curl` command with your **client ID** in it.

   ![Confirmation page listing the created app, pool, default resource server and sample token request](images/m05-02-cognito-created.png)

### Step B: Rename the pool and record the issuer

5. Choose **Go to overview**. Choose **Rename**, enter `orbit-MY_ID-pool`, and **Save changes**.

   ![Rename dialog for the user pool](images/m05-03-cognito-rename.png)

6. On the **Overview** page record three values: the **User pool ID** (such as `us-west-2_7vX2h98jw`), the **OpenID Connect configuration URL** (this is your **discovery URL**, shaped like `https://cognito-idp.us-west-2.amazonaws.com/USER_POOL_ID/.well-known/openid-configuration`), and the **Token signing key URL** (the JWKS endpoint AgentCore will fetch keys from).

   ![User pool overview with the pool ID and the OIDC configuration URL](images/m05-04-cognito-overview.png)

   Open the OIDC configuration URL in a new browser tab. It is public JSON metadata; confirm its `issuer` is `https://cognito-idp.us-west-2.amazonaws.com/USER_POOL_ID`. The issuer is the pool itself, **not** the domain you will see in Step D.

### Step C: Create your resource server and scope

7. In the left navigation choose **Applications → Resource servers → Create resource server**.
8. **Resource server name** `orbit-MY_ID-api` · **Resource server identifier** `orbit-MY_ID-api` · choose **Add custom scope** → **Scope name** `invoke`, **Description** `Invoke my ORBIT training agent` → **Create resource server**.

   ![Create resource server form with the invoke scope](images/m05-05-resource-server-form.png)

   The list now shows two resource servers: the wizard's default one and yours. Your fully qualified scope is **`orbit-MY_ID-api/invoke`**: a bare `invoke` is not the same string.

   ![Resource servers list with the default and the orbit-s07x42-api servers](images/m05-06-resource-servers.png)

### Step D: Authorize the scope on your app client, and find the token endpoint and secret

9. **Applications → App clients → `orbit-MY_ID-m2m`**. The *App client information* panel shows the **Client ID** and a masked **Client secret** with a *Show client secret* link. Look at the secret only when you copy it into Secrets Manager in Phase 3; never paste it into chat, a screenshot, a Lambda test event, or an environment variable.

   ![App client page with client ID and masked client secret](images/m05-07-app-client.png)

10. Open the **Login pages** tab. Under *Managed login pages configuration* note **OAuth grant types: Client credentials grant** and **Custom scopes** (only the default scope so far). Choose **Edit**.
11. In **Custom scopes** open *Select custom scopes* and add **`orbit-MY_ID-api/invoke`**. Leave the client credentials grant as the only grant. **Save changes**.

    ![Edit managed login pages: client credentials grant with both custom scopes selected](images/m05-08-app-client-scopes.png)

    ![Login pages tab now listing orbit-s07x42-api/invoke](images/m05-09-app-client-login-pages.png)

12. **Branding → Domain**. The wizard already created a **Cognito domain** with a unique prefix (for example `https://us-west-27vx2h98jw.auth.us-west-2.amazoncognito.com`). Record it. Your **token endpoint** is that domain plus `/oauth2/token`. You do not need to create a custom domain.

    ![Domain page showing the automatically created Cognito domain](images/m05-10-cognito-domain.png)

> [!NOTE]
> Because the wizard generates the domain prefix, 30 participants cannot collide on it. Everything else you typed carries `MY_ID`. Cognito's per-account quotas (1,000 pools, 150 token requests per second) are far above what the class needs.

---

## 🔒 Phase 3 — Store the secret in AWS, not in code (10 minutes)

1. Console search → **Secrets Manager** → **Store a new secret**.
2. **Secret type:** **Other type of secret**. Under **Key/value pairs** add four rows (use **+ Add row**). The keys must match the test code exactly:

   | Key | Value |
   |---|---|
   | `client_id` | your app client ID |
   | `client_secret` | your client secret (from *Show client secret* in Phase 2) |
   | `token_url` | your domain + `/oauth2/token` |
   | `scope` | `orbit-MY_ID-api/invoke` |

   ![Key/value pairs for the secret (the secret value is a placeholder in this screenshot)](images/m05-11-secret-key-value.png)

3. Keep the default encryption key and choose **Next**. **Secret name:** `orbit/MY_ID/cognito-m2m`. Add a description. **Next**.
4. Leave **Configure rotation** off (a training secret you will delete today). **Next**, review, then **Store**.
5. Open the secret and copy its **Secret ARN**. Note that Secrets Manager appends six random characters to the ARN (`…:secret:orbit/MY_ID/cognito-m2m-AbCdEf`); you will need the exact ARN for the IAM policy in Phase 5.

**Why this matters:** the ARN is a pointer; the value is credential material. IAM controls who can read the value. Do not give any role `secretsmanager:*` on `*` because it is easier.

---

## 🛡️ Phase 4 — Create the JWT-protected harness (15 minutes)

**Concept:** Unlike the Mission 04 harness, this one will require a validated OAuth bearer token at its inbound boundary. It still needs an AWS execution role for model inference; the two are unrelated.

1. **AgentCore → Build → Harness → Quick create Harness ▾ → Advanced create Harness**.
2. **Harness name:** `orbit_MY_ID_secure`.
3. **Model and system prompt:** change the model to **Anthropic → Claude Haiku 4.5** (inference profile *US Anthropic Claude Haiku 4.5*), and set the system prompt:

   > You are the ORBIT Outfitters customer support assistant. Answer in a warm, supportive voice. Never invent a returns or shipping policy, and never claim to have issued a refund or changed an order.

4. **Memory:** expand it and **untick Enable Memory**. A security test does not need memory, and this avoids the *Memory ARN is required* validation error.

   ![Memory section with Enable Memory unticked](images/m05-14-harness-secure-memory-off.png)

5. Skip **Tools**, **Skills**, **Lifecycle hooks**, and **Advanced configurations**.
6. **Inbound Auth:** choose **Use JSON Web Tokens (JWT)**, then **Use existing Identity provider configurations** (not *Quick create with Cognito*, which would create a second pool you do not control). Fill in:
   - **Discovery URL:** your OIDC configuration URL from Phase 2 (the full `…/.well-known/openid-configuration` URL, not the domain, not the JWKS URL).
   - Leave **Allowed audiences** unticked.
   - **Allowed clients:** tick it and enter your **app client ID**.
   - **Allowed scopes:** tick it and enter `orbit-MY_ID-api/invoke`.
   - Leave **Custom claims** unticked.

   ![Inbound Auth section configured for JWT with discovery URL, allowed client and allowed scope](images/m05-13-harness-secure-jwt.png)

7. **Permissions:** **Create default role**, **Role name** `orbit-MY_ID-secure-role`.

   ![Advanced create Harness form for the secure harness](images/m05-12-harness-secure-create.png)

8. **Create Harness**. Wait for **Ready** (about a minute). On the detail page expand **Inbound Auth** and confirm it shows **JSON Web Tokens (JWT)**, your discovery URL, your client ID, and your scope. Record the **Harness ARN**.

   ![Secure harness detail page with the Inbound Auth panel showing the JWT settings](images/m05-15-harness-secure-inbound.png)

> [!IMPORTANT]
> The console **Harness playground signs requests with your own IAM identity**, so it cannot exercise the JWT path and may simply refuse. **Do not downgrade the harness to IAM to "make the playground work."** The Lambda client in the next phase is the valid test.

---

## 🧪 Phase 5 — Build an AWS-only authenticated test client (25 minutes)

**Concept:** A credible security test sends both permitted and forbidden requests to the **same** secured endpoint. Your Lambda obtains a token from Cognito over HTTPS, calls the harness's public invoke endpoint with a `Bearer` header, and returns only sanitized status information.

### Step A: Create the function and its configuration

1. **Lambda → Create function → Author from scratch**: **Function name** `orbit-MY_ID-jwt-test`, **Runtime** Python 3.13, default execution role. **Create Event Function**.

   ![Create function form for the JWT test client](images/m05-16-lambda-create.png)

2. **Configuration tab → Environment variables → Edit → Add environment variable** three times, then **Save**:

   | Key | Value |
   |---|---|
   | `SECRET_ARN` | the secret ARN from Phase 3 |
   | `HARNESS_ARN` | the secure harness ARN from Phase 4 |
   | `AWS_REGION_FOR_LAB` | `us-west-2` |

   ![Edit environment variables with the three keys](images/m05-17-lambda-env-vars.png)

   These are **ARNs and a Region name**, not credentials. The secret value is read at run time.

3. **Configuration → General configuration → Edit**: set **Timeout** to **1 min 0 sec** (the default 3 seconds is too short for a token request plus a model call). **Save**.

   ![General configuration edit with the timeout set to 1 minute](images/m05-18-lambda-timeout.png)

### Step B: Give the function least-privilege access to the secret

4. **Configuration → Permissions** → click the **Role name** link (`orbit-MY_ID-jwt-test-role-…`) to open IAM.

   ![IAM role page for the Lambda test client](images/m05-19-iam-lambda-role.png)

5. **Add permissions ▾ → Create inline policy → JSON**. Paste the policy below with **your** secret ARN (keep the trailing `-*` or use the exact six-character suffix):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "ReadMyOrbitTestSecretOnly",
         "Effect": "Allow",
         "Action": "secretsmanager:GetSecretValue",
         "Resource": "arn:aws:secretsmanager:us-west-2:111122223333:secret:orbit/MY_ID/cognito-m2m-*"
       }
     ]
   }
   ```

   ![Inline policy JSON allowing GetSecretValue on one secret](images/m05-20-iam-inline-policy-json.png)

6. **Next** → **Policy name** `orbit-MY_ID-read-secret` → **Create policy**.

**Why no `bedrock-agentcore:InvokeHarness` permission for this Lambda?** It calls the harness over HTTPS with a **JWT bearer token**, not with AWS signatures. The JWT authorizer, not IAM, decides. (The harness execution role separately needs permission to call the model, and the default role already has it.)

### Step C: Paste the test code

7. **Code tab**: select everything in `lambda_function.py`, paste the code below, and choose **Deploy**. It uses only Python's standard library plus the AWS SDK that Lambda already includes.

```python
import base64
import json
import os
import re
import urllib.error
import urllib.parse
import urllib.request
import uuid

import boto3


def http_post(url, body, headers):
    request = urllib.request.Request(url=url, data=body, headers=headers, method="POST")
    try:
        with urllib.request.urlopen(request, timeout=55) as resp:
            return resp.status, resp.read(65536).decode("utf-8", "replace")
    except urllib.error.HTTPError as exc:
        # Expected for missing, forged, or expired credentials.
        return exc.code, exc.read(65536).decode("utf-8", "replace")


def lambda_handler(event, context):
    # event mode: good, missing, forged
    mode = (event or {}).get("mode", "good")
    if mode not in ("good", "missing", "forged"):
        return {"error": "Choose good, missing, or forged"}

    # Read the M2M client credentials from Secrets Manager (never from code).
    secret = json.loads(
        boto3.client("secretsmanager")
        .get_secret_value(SecretId=os.environ["SECRET_ARN"])["SecretString"]
    )

    region = os.environ["AWS_REGION_FOR_LAB"]
    harness_arn = os.environ["HARNESS_ARN"]
    runtime_url = (
        f"https://bedrock-agentcore.{region}.amazonaws.com/harnesses/invoke"
        f"?harnessArn={urllib.parse.quote(harness_arn, safe='')}"
    )

    headers = {
        "Content-Type": "application/json",
        # Every harness conversation needs a session id of 33+ characters.
        "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id": f"orbit-test-{uuid.uuid4()}",
    }
    token_status = None
    if mode == "good":
        pair = f"{secret['client_id']}:{secret['client_secret']}"
        basic_auth = base64.b64encode(pair.encode()).decode()
        payload = urllib.parse.urlencode({
            "grant_type": "client_credentials",
            "scope": secret["scope"],
        }).encode()
        token_status, response_text = http_post(
            secret["token_url"], payload,
            {"Content-Type": "application/x-www-form-urlencoded",
             "Authorization": f"Basic {basic_auth}"},
        )
        if token_status != 200:
            return {"mode": mode, "token_status": token_status,
                    "phase": "Cognito token issuance failed",
                    "body_preview": response_text[:300]}
        access_token = json.loads(response_text)["access_token"]
        headers["Authorization"] = f"Bearer {access_token}"
    elif mode == "forged":
        headers["Authorization"] = "Bearer this.is.not.a.valid.jwt"

    body = json.dumps({"messages": [{"role": "user", "content": [
        {"text": "In one sentence, explain least privilege."}]}]}).encode()
    status, response_text = http_post(runtime_url, body, headers)

    # A successful harness call streams events; pull out just the model's text.
    if status == 200:
        pieces = re.findall(r'"text":"((?:[^"\\\\]|\\\\.)*)"', response_text)
        response_text = "".join(pieces).encode().decode("unicode_escape", "replace") or response_text

    # Keep credentials and JWTs out of logs and returned data.
    return {
        "mode": mode,
        "token_status": token_status,
        "runtime_http_status": status,
        "body_preview": response_text[:300],
    }
```

   ![Lambda code editor with the deployed JWT test client](images/m05-21-lambda-code.png)

   Read the code once before running it:
   - the **invoke URL** is `https://bedrock-agentcore.<region>.amazonaws.com/harnesses/invoke?harnessArn=<URL-encoded ARN>`;
   - every call carries a **session ID header** of at least 33 characters, because a harness conversation is a session;
   - a successful answer arrives as a **stream of events**, so the code pulls out just the model's text for the preview;
   - the function **never returns or logs** the token or the secret.

### Step D: Run good, missing, and forged

8. **Test tab**: with **Create new event** selected, set the Event JSON to `{"mode": "good"}` and choose **Test** (you can also **Save** it as an event named `good`). Expand **Details** and read the **Response**. Repeat with `{"mode": "missing"}` and `{"mode": "forged"}`.

   ![good: token_status 200 and runtime_http_status 200 with the model's one-sentence answer](images/m05-22-test-good.png)

   ![missing: runtime_http_status 403, Missing Authentication Token](images/m05-23-test-missing.png)

   ![forged: runtime_http_status 403, OAuth authorization failed: Failed to parse token](images/m05-24-test-forged.png)

   Verified results:

   | Test mode | Cognito token exchange | AgentCore inbound decision | Observed |
   |---|---|---|---|
   | `good` | `token_status: 200` | accepted (issuer, client, scope all valid) | `runtime_http_status: 200`, preview: *"Least privilege is the security principle of granting…"* |
   | `missing` | not attempted | rejected, no bearer token | `403`, `{"message":"Missing Authentication Token"}` |
   | `forged` | not attempted | rejected, unparseable JWT | `403`, `{"message":"OAuth authorization failed: Failed to parse token"}` |

> [!CAUTION]
> **Do not paste token values** into screenshots, chats, logs, or this guide. The function deliberately does not print or return them. Restrict who may run or edit your test Lambda; the ability to invoke a client that holds a secret is itself sensitive.

### Step E: Troubleshoot the right side of the boundary

| Symptom | Likely layer | Correct console check |
|---|---|---|
| Lambda error `AccessDeniedException` on `GetSecretValue` | Lambda execution role | Inline policy resource matches the **exact** secret ARN (or `-*`); same Region |
| `token_status` 400 `invalid_client` | Cognito app client | Client ID and secret in the secret value; the client must have **client credentials** enabled |
| `token_status` 400 `invalid_scope` | Cognito scope assignment | The scope is listed under the app client's **Custom scopes** (Phase 2, Step D) and spelled `orbit-MY_ID-api/invoke` |
| `token_status` 404 or a DNS error | token URL | Must be the **domain** + `/oauth2/token`, not the issuer URL |
| `runtime_http_status` 401/403 **with** a 200 token | AgentCore inbound JWT | Discovery URL points at the **same** pool; **Allowed clients** has this client ID; **Allowed scopes** matches the scope in the token |
| `runtime_http_status` 5xx with a 200 token | Harness process or model | Harness status Ready; execution role; CloudWatch log group `/aws/bedrock-agentcore/runtimes/harness_orbit_MY_ID_secure-…` |
| `Task timed out after 3.00 seconds` | Lambda timeout | Phase 5, Step A, item 3 |
| `missing` or `forged` returns **200** | Security misconfiguration | Stop. The harness is not using JWT inbound auth; check Phase 4 |
| *The user pool cannot be deleted because deletion protection is activated* | Cognito cleanup | Deactivate deletion protection on the pool before deleting it (cleanup step 4) |

**Boss challenge — audience confusion:** a teammate sets **Allowed audiences** to the app client ID but leaves **Allowed clients** empty. Why is a perfectly good Cognito access token rejected? Because the authorizer compares *Allowed audiences* with the `aud` claim, and Cognito access tokens carry `client_id` instead. The two checks validate **different claims**.

**Boss challenge — issuer mismatch (analysis only):** an app's token is signed by Pool A but the harness's discovery URL points at Pool B. What should AgentCore do? Reject it: the signature cannot be verified with Pool B's keys and `iss` does not match. Do not modify your working authorizer to prove this; reason it through from the Inbound Auth panel.

---

## 🔎 Phase 6 — What you have secured, and what you have not (10 minutes)

Complete the threat map in your notes:

| Threat | Which control helps? | Remaining question |
|---|---|---|
| Random internet caller tries the invoke URL | JWT inbound authorizer (`missing` → 403) | Are issuer, client and scope checks all configured? |
| Attacker fabricates a token | Signature verification via the discovery URL (`forged` → 403) | Are the signing keys fetched from the right pool? |
| A valid app tries an unauthorized refund | Tool-level authorization + business policy | Does the refund tool check the customer's identity and order? |
| The agent process reads secrets unrelated to it | Execution role scoped by ARN | Did anyone attach `secretsmanager:*`? |
| "Ignore your rules" text hidden in a retrieved document | Treat retrieval as untrusted data | Does the agent separate instructions from source text? |
| One student's token used against another's harness | Different app clients, scopes and authorizers | Is **Allowed clients** specific to your `MY_ID`? |

**Key insight:** *Authenticated* is not *authorized for every action*. A token proves an approved app is calling; a production refund tool would still have to decide whether that app may touch **this order** and run **this refund**.

### Optional inspection — outbound identity

Open **AgentCore → Identity** and look at the workload identities and credential providers. Do not create providers for outside services. Answer: how is **inbound JWT validation** (who may call me) different from **outbound OAuth credentials** (what I may call downstream)?

---

## 🏁 Final scorecard — Earn the Identity Badge

- [ ] I used **only the AWS Console** and AWS-hosted execution.
- [ ] I created my **own Cognito user pool**, M2M app client, resource server and scope, all carrying `MY_ID`.
- [ ] I stored the client secret in **Secrets Manager**, never in code or environment variables.
- [ ] I created an AgentCore **Harness with a JWT inbound authorizer** pointing at my pool, my client, and my scope.
- [ ] My Lambda test obtained an M2M token and got **200** from the secured harness.
- [ ] I captured **403** for both the missing and the forged token.
- [ ] I can explain `iss`, `client_id`, `scope`, `exp`, why `aud` is absent, and the separate IAM roles.
- [ ] I can explain why M2M authentication is **not** interactive user sign-in.

---

## 🧯 Quotas, costs, and cleanup — mandatory in a shared account

**What this mission costs** (US West (Oregon) list prices, October 2026):

| Item | Price | Typical use | Cost per participant |
|---|---|---|---|
| Cognito M2M token requests | $0.00225 per successful request, no free tier | 3–10 requests | $0.01–0.02 |
| Cognito user pool (Essentials tier) | free up to 10,000 monthly active users | 0 users | $0 |
| Secrets Manager | $0.40 per secret per month, prorated hourly | one secret for a few hours | < $0.01 (**$0.40 per month if left behind**) |
| Secure harness: Claude Haiku 4.5 + Runtime sessions | as in Mission 04 | 3–6 short calls, each in its own session | about $0.03 |
| Lambda and CloudWatch | per request and per GB | a handful of invocations | < $0.01 |
| **Total** | | | **about $0.05** (budget $0.10) |

A class of 25 spends roughly **$2** on this mission. The one recurring item is the secret: 25 forgotten secrets cost about **$10 per month**, so delete yours at the end.

**Shared-account checklist:**

- Cognito token requests are rate-limited per account (150 per second). You will not hit that, but do not loop the test.
- 15 secure harnesses plus 15 Mission 04 harnesses are well within the account's limits; stagger creation in waves of five if the instructor asks.
- Never enable payload or token logging to debug authentication.

**Delete only your own resources, in this order:**

1. **Lambda:** delete `orbit-MY_ID-jwt-test`.
2. **Harness:** delete `orbit_MY_ID_secure` (its runtime and endpoint go with it).
3. **Secrets Manager:** open `orbit/MY_ID/cognito-m2m` → **Actions → Delete secret**. The default is a 7-day recovery window, which is fine; billing stops when the deletion is scheduled.
4. **Cognito:** open `orbit-MY_ID-pool`. The wizard switched on **deletion protection**, so first go to **Settings → Deletion protection** (on the pool overview) and **Deactivate** it. Then delete the app client `orbit-MY_ID-m2m`, delete the domain under **Branding → Domain**, and finally **Delete user pool** (type its name to confirm). Resource servers are removed with the pool.
5. **IAM:** delete `orbit-MY_ID-secure-role` and the auto-created `orbit-MY_ID-jwt-test-role-…` only after their resources are gone.
6. Return to the Mission 04 and Mission 03 cleanup lists for the gateway, Lambda tool, first harness, and knowledge base.

**AWS docs to consult:** [Cognito resource servers and scopes](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html) · [Cognito app clients](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html) · [Cognito token endpoint](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html) · [AgentCore inbound JWT authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html) · [Harness security and access controls](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-security.html) · [Cognito pricing](https://aws.amazon.com/cognito/pricing/) · [Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/).

*End of Mission 05 · ORBIT's agent perimeter is now something you can prove with real AWS requests, not merely describe.*
