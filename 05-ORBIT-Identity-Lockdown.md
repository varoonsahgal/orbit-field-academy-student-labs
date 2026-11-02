# 🛰️ ORBIT FIELD ACADEMY
## MISSION 05 · IDENTITY LOCKDOWN
### Cognito, OAuth, JWT, IAM, Secrets Manager, and a JWT-protected AgentCore agent

> **LEVEL:** Security Engineer · **TARGET TIME:** 110–140 minutes · **ENVIRONMENT:** AWS Management Console only
> **THE STORY:** ORBIT's AI assistant can answer requests, but *who* is allowed to invoke it? Can a stranger pass any token? Can an authenticated app claim permission to change an order? You will build a real Cognito-backed perimeter around a second agent and prove it with successful **and** rejected requests sent from **AWS Lambda in the AWS Console**.

**Nothing runs on your laptop.** This mission uses Amazon Cognito, AgentCore Harness, AWS Secrets Manager, IAM, the Lambda browser editor and Test tab, and CloudWatch. There is no CLI, terminal, browser plug-in, or external HTTP tool.

> [!NOTE]
> **Re-verified against the live AWS Console on October 25, 2026** in US West (Oregon). Screenshots show the example IDs `s07x42` and `i25k10`; yours will show your own `MY_ID`. What changed in this edition: the test client now proves **five** outcomes instead of three (a tampered token and a wrong-scope token were added), the HTTP-status guide matches what AgentCore really returns, and the Harness form's new defaults (managed Memory switched on, *Allowed clients* pre-ticked) are called out where you will meet them.

---

## 🗺️ Where this mission fits

In Mission 03 you gave ORBIT a **memory of facts** (a knowledge base). In Mission 04 you gave it **hands** (a harness that can call a Lambda tool through a gateway). Both of those agents could only be called by someone **already signed in to this AWS account**, because their front door used **IAM**. That is fine for engineers in the console, but ORBIT's real customers, partner apps and mobile apps will **never** have AWS credentials.

Mission 05 builds the front door that the outside world will use. Look back at the Mission 04 migration map: the row *Inbound caller authentication* said **"IAM today, JWT in Mission 05"**. Today is that row.

```mermaid
flowchart LR
    subgraph inside["Inside your AWS account"]
        direction TB
        H4["🤖 Mission 04 harness<br/>orbit_MY_ID<br/>front door: IAM"]
        GW["🌐 Gateway → Lambda tool<br/>+ Mission 03 knowledge base"]
        H4 --> GW
        H5["🤖 Mission 05 harness<br/>orbit_MY_ID_secure<br/>front door: JWT"]
    end
    ENG["👩‍💻 Engineer signed in<br/>to the AWS console"] -->|"AWS-signed request (IAM)"| H4
    APP["📱 Outside app<br/>(no AWS credentials)"] -->|"① ask for a token"| COG["🪪 Amazon Cognito"]
    COG -->|"② signed JWT"| APP
    APP -->|"③ Bearer JWT"| H5
    classDef today fill:#FF9900,color:#111,stroke:#c46f00
    class H5,COG,APP today
```

> 🏢 **An analogy to carry through the whole mission.** Think of ORBIT HQ as an office building.
> - **Cognito** is the **badge office**. An app proves who it is (its ID and a password) and the office prints a **visitor badge**.
> - The badge is a **JWT**. It says *who* it was issued to, *which floors* it opens (**scopes**), *when it expires*, and it carries a **hologram** (the **signature**) that only the badge office can produce.
> - The **AgentCore JWT authorizer** is the **guard at the elevator**. The guard doesn't phone the badge office for every visitor. They check the hologram against the office's published sample, read the name and floor, check the date, and either wave you through or turn you away.
> - The **harness execution role** is the **staff keycard the agent itself carries** inside the building to open the model room. A visitor's badge never turns into a staff keycard.
> - **Secrets Manager** is the **locked safe** where the app's password is kept, so it never sits on a sticky note (in code).

---

## 📖 Before you start: the concepts in this mission

You will meet a lot of new words today. Rather than memorizing definitions, keep asking one question: **"who is proving what, to whom?"** Every piece below answers that question for a different hop.

### Authentication vs. authorization

- **Authentication** answers *who are you?* A caller proves its identity, for example by presenting a token that only a trusted issuer could have signed.
- **Authorization** answers *what may you do?* Knowing who you are is only the start. This app must still be allowed to call *this* agent, with *this* scope, and the agent's tools must still decide whether *this* order may be touched.

**Why the difference matters today:** you will see a request that is perfectly **authenticated** (a genuine Cognito token from your own pool) and is still **rejected**, because it asked for the wrong permission. If you remember only one thing from this mission, make it this: *a real identity is not the same as permission.*

### Why not just use IAM like Mission 04?

IAM is AWS's own identity system. To pass an IAM check, the caller must sign every request with AWS access keys. That works for code running *inside* AWS and for people in the console. It fails for:

- a mobile app on a customer's phone (you would have to ship AWS keys inside the app, where anyone can extract them);
- a partner company's server (you would have to create IAM users for another company);
- a web front-end whose users sign in with email, Google or a corporate login.

The industry's answer is **OAuth 2.0 tokens**. A dedicated identity service hands out short-lived, signed tokens, and the API checks the token. AgentCore supports both styles, which is why every harness and gateway has an **Inbound Auth** setting with two choices: **IAM** or **JSON Web Tokens**.

### OAuth 2.0 in two minutes

OAuth 2.0 is the standard that lets an application obtain an **access token** from an **authorization server** and present it to a **resource server** (an API). Four roles matter today:

| OAuth role | Today it is… | In the building analogy |
|---|---|---|
| **Client** | Your Lambda test function, identified by an **app client ID** and a **client secret** | The visitor asking for a badge |
| **Authorization server** | Your Amazon Cognito **user pool** and its **token endpoint** | The badge office |
| **Resource server** | A name you register in Cognito (`orbit-MY_ID-api`) that owns one or more **scopes** | The department whose floors the badge can open |
| **Scope** | A permission string in the token, such as `orbit-MY_ID-api/invoke`, that the API checks | A floor printed on the badge |

The **client credentials grant** is the OAuth flow for **machine-to-machine (M2M)** calls. No human signs in: the client sends its ID and secret and gets back a token with custom scopes. A human user would use the **authorization code** grant with PKCE instead, typing a password into a Cognito-hosted sign-in page.

**Why we use M2M today:** it is the simplest flow that produces a real, signed token without needing a web page or a test user. The *checking* half, which is the part you configure on the harness, is identical for both flows. Once you have built today's front door, a human sign-in flow is just a different way of getting the badge.

### JSON Web Tokens (JWT)

An access token from Cognito is a **JWT** (say "jot"): three Base64URL strings separated by dots, `header.payload.signature`. Base64URL is just an encoding, **not encryption**, so anyone holding the token can read the payload. That is fine, because the payload holds **claims** about the caller, not secrets:

| Claim | Meaning | Where you will see it |
|---|---|---|
| `iss` | Issuer: the exact user pool URL | Must match the pool behind your **Discovery URL** |
| `client_id` | Which app client requested the token | **Allowed clients** on the harness |
| `scope` | Space-separated scopes granted | **Allowed scopes** on the harness |
| `exp` | Expiry time (Cognito default 60 minutes) | Expired tokens are rejected |
| `aud` | Audience: which API the token is for | Cognito **access tokens do not carry `aud`**; do not configure *Allowed audiences* |

The **signature** is what makes a readable token trustworthy. Cognito signs the header and payload with a **private key** that never leaves Cognito. It publishes the matching **public keys** at a well-known address. Anyone can use a public key to *check* a signature, but nobody can use it to *create* one. Change a single character of the payload or the signature, and the check fails.

**How AgentCore finds those public keys:** you give it one URL, the pool's **OpenID Connect discovery URL** (`…/.well-known/openid-configuration`). That small public JSON document tells AgentCore the pool's official **issuer** name and where its **public keys** (the *JWKS*) live. This is why the discovery URL is the single most important thing you type into the harness. It decides *whose* badges the guard trusts.

### The AWS services, and why each one is here

| Service | What it is | Why this mission needs it |
|---|---|---|
| **Amazon Cognito user pool** | An identity store plus an OAuth/OIDC authorization server | It is the badge office. Today it holds **no human users at all**, just one M2M **app client**, one **resource server** and a **domain** (the hostname of its token endpoint). |
| **AWS Secrets Manager** | Encrypted storage for credentials, with its own IAM access control | The client secret is a password. Code refers to the secret by **ARN** and reads the value at run time, so it never appears in source code, environment variables, screenshots or logs. |
| **Lambda environment variables** | Plain configuration that your function reads with `os.environ` | ARNs and Region names are *addresses*, not secrets. Anyone who can view the function can read them. |
| **AgentCore inbound auth** | The gatekeeper in front of every harness and gateway | *Use IAM permissions* (Mission 04) accepts AWS-signed requests. *Use JSON Web Tokens* accepts a `Bearer` token that matches the discovery URL, allowed clients and allowed scopes you configure. |
| **AWS Lambda** | Code that runs on demand without a server | We need *something* to play the outside app. A Lambda in the console lets you send real HTTPS requests without installing anything. |

### Who is proving what, to whom: the three hops

The same request crosses **three different security checks**, each run by a different service with different credentials. Most debugging confusion today comes from fixing the wrong hop.

```mermaid
flowchart LR
    L["🧪 Lambda test client"] -->|"Hop 1 · IAM<br/>Lambda role may read ONE secret"| SM["🔒 Secrets Manager"]
    L -->|"Hop 2 · OAuth<br/>client ID + secret → token"| C["🪪 Cognito"]
    L -->|"Hop 3 · JWT<br/>Bearer token → allowed?"| A["🛡️ AgentCore JWT authorizer"]
    A --> H["🤖 Secure harness"]
    H -->|"Hop 4 · IAM<br/>harness role may call the model"| B["🧠 Bedrock model"]
```

| Hop | Who proves | To whom | With what | Configured in |
|---|---|---|---|---|
| 1 | Lambda's execution role | Secrets Manager | AWS signature (IAM) | Phase 5, Step B |
| 2 | Your app client | Cognito | Client ID + client secret | Phase 2 |
| 3 | The token holder | AgentCore JWT authorizer | Signed JWT | Phase 4 |
| 4 | The harness's execution role | Amazon Bedrock | AWS signature (IAM) | Phase 4 (default role) |

Notice that IAM appears **twice**, on hops 1 and 4, and that neither IAM check knows anything about the JWT. That is by design.

### Know the difference

| Control | Protects | Does **not** automatically protect |
|---|---|---|
| **IAM for you in the console** | Who may create or change resources | App end users |
| **Harness execution role (IAM)** | Which AWS APIs the agent process may call | Whether an external JWT is valid |
| **Cognito user pool and M2M app client** | Token issuance, client identity, scopes | IAM access to Bedrock or Secrets Manager |
| **AgentCore inbound JWT authorizer** | Which tokens may invoke the harness | Business rules inside each tool |
| **Gateway and target roles** (Mission 04) | Which tools may be invoked | Whether *this* customer may touch *this* order |

---

## 🪪 Phase 0 — Shared-account and pricing preflight (5 minutes)

1. Use the **same `MY_ID` and Region** as Missions 03–04.
2. **Cost facts (verified October 2026):** Cognito bills **$0.00225 per successful M2M token request**, with no free tier and **no charge per app client**. You will make roughly 5–15 token requests. Secrets Manager bills **$0.40 per secret per month**, prorated hourly. The protected harness bills like Mission 04. The whole mission costs a few cents per participant; the only thing worth deleting promptly afterwards is the secret.
3. Create a **new** harness for this mission. Do not switch the Mission 04 harness to JWT. Keeping both lets you compare an IAM front door with a JWT front door side by side, and it avoids breaking the Mission 04 gateway test.

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

**Why plan first?** Security engineers write down the rule *before* building the lock. If you can't say what a valid caller looks like, you can't tell whether a rejection is a bug or the system working.

Write the rule set you intend to enforce:

- `iss` must be **my** user pool (only badges from *my* badge office).
- `client_id` must be **my** app client (only badges issued to *my* app).
- `scope` must include `orbit-MY_ID-api/invoke` (the badge must open *this* floor).
- `exp` must be in the future (no expired badges).
- The signature must verify against my pool's public keys (no photocopied or altered badges).
- `aud` is **not** checked (Cognito access tokens do not carry it).

Now write the **evidence plan**: the five requests your test client will send in Phase 5, and what each one should prove.

| Test | What it sends | Proves the authorizer checks… | Expected |
|---|---|---|---|
| `good` | A genuine token with the right scope | the happy path works | accepted |
| `missing` | No token | that a token is required at all | rejected |
| `forged` | A string that isn't a JWT | the token's structure | rejected |
| `tampered` | A genuine token with an altered signature | the signature | rejected |
| `wrong-scope` | A genuine token for a different scope | the **scope**: authentication ≠ authorization | rejected |

**Checkpoint:** if a valid app sends a Cognito JWT to AgentCore, does the caller gain `secretsmanager:*` or `bedrock:*` privileges? **No.** The token only opens the harness's front door. What the agent may do inside AWS comes from its execution role, which is enforced separately (hop 4).

---

## 🏷️ Phase 2 — Set up Cognito with a unique app identity (25 minutes)

```mermaid
flowchart LR
    S["🔒 Secrets Manager"] -.-> T["🧪 Lambda test client"]
    T <-->|"① get token"| C["🪪 Cognito<br/>◀ YOU ARE HERE"]
    T -->|"② Bearer JWT"| G["🛡️ JWT authorizer"]
    C -.->|"public keys"| G
    G --> H["🤖 Secure harness"] --> M["🧠 Model"]
    classDef now fill:#FF9900,color:#111,stroke:#c46f00,stroke-width:3px
    classDef done fill:#d9f2e3,color:#14532d,stroke:#3a9d63
    classDef todo fill:#eef1f4,color:#555,stroke:#c3cad3
    class C now
    class S,T,G,H,M todo
```

**What you are building and why:** the badge office. By the end of this phase Cognito will know about one app (`orbit-MY_ID-m2m`), one API it may get badges for (`orbit-MY_ID-api`) with one permission (`invoke`), and it will have a public web address where apps go to collect badges (the **domain** + `/oauth2/token`).

### Step A: Create the user pool and M2M app client in one wizard

> [!CAUTION]
> **You create a brand-new pool here. Do not open an existing one.** The **User pools** list shows every pool in the shared account: the instructor's read-only `orbit-academy-pool` from Mission 02, and your classmates' pools. In an earlier class several people opened a pool that was not theirs and added their app client to it; their tokens then came from someone else's badge office and every test failed in confusing ways. Choose **Create user pool**, and only ever open a pool whose name contains *your* `MY_ID`.
>
> | | Mission 02 | Mission 05 (today) |
> |---|---|---|
> | Pool | `orbit-academy-pool` (shared, instructor's) | `orbit-MY_ID-pool` (yours, you create it) |
> | Who signs in | People (recruits) on a hosted login page | A program, with a client ID and secret |
> | OAuth flow | Authorization code | Client credentials (machine-to-machine) |
> | You may change it? | **No**, read-only | **Yes**, and you delete it at the end |

1. Console search → **Cognito** → **User pools** → **Create user pool**. The wizard is titled *Set up resources for your application*.
2. Under **Application type** choose **Machine-to-machine application**. The other options (web, SPA, mobile) are for human sign-in and add fields you do not need.
3. **Name your application:** `orbit-MY_ID-m2m`.

   ![Cognito wizard with Machine-to-machine application selected and the app named](images/m05-01-cognito-wizard-m2m.png)

4. Choose **Create user directory**. The confirmation page shows that **both** an app client and a user pool were created. The pool gets an automatic name such as *User pool - edwkah*; you will rename it next. Read the page: Cognito also created a **default resource server** (`default-m2m-resource-server-xxxxxx/read`) and shows a sample `curl` command with your **client ID** in it. **Write down that default scope name**; you will use it as the "wrong" scope in Phase 5.

   ![Confirmation page listing the created app, pool, default resource server and sample token request](images/m05-02-cognito-created.png)

> 💡 **Why is it called a "user pool" if there are no users?** Cognito began as a sign-in service for people, so its container is a *user pool*. An app client is also an identity, a "user" that happens to be a program. One pool can serve human sign-in and M2M apps at the same time, and both get tokens signed by the same keys.

### Step B: Rename the pool and record the issuer

5. Choose **Go to overview**. Choose **Rename**, enter `orbit-MY_ID-pool`, and **Save changes**. (In a shared account, a name that carries `MY_ID` is how you, your classmates and your instructor tell 30 pools apart. **Do not skip this.** After the last class the account held eleven pools still called *User pool - xxxxxx* that nobody could identify, and pools nobody can identify are pools nobody dares to delete.)

   ![Rename dialog for the user pool](images/m05-03-cognito-rename.png)

6. On the **Overview** page record three values:
   - the **User pool ID** (such as `us-west-2_7vX2h98jw`);
   - the **OpenID Connect configuration URL**, which is your **discovery URL**, shaped like `https://cognito-idp.us-west-2.amazonaws.com/USER_POOL_ID/.well-known/openid-configuration`;
   - the **Token signing key URL**, the JWKS endpoint holding the public keys AgentCore will fetch.

   ![User pool overview with the pool ID and the OIDC configuration URL](images/m05-04-cognito-overview.png)

   Open the OIDC configuration URL in a new browser tab. It is public JSON metadata: no sign-in is needed, because public keys are *meant* to be public. Find two lines:
   - `"issuer"` is `https://cognito-idp.us-west-2.amazonaws.com/USER_POOL_ID`. Every token from this pool carries exactly this string in `iss`.
   - `"token_endpoint"` points at a **different** hostname, `https://<prefix>.auth.us-west-2.amazoncognito.com/oauth2/token`.

   **That difference is today's most common mistake.** The **issuer** (the pool's identity, used for *checking* tokens) is not the **domain** (the web address where apps *collect* tokens). AgentCore needs the first; your Lambda needs the second.

### Step C: Create your resource server and scope

7. In the left navigation choose **Applications → Resource servers → Create resource server**.
8. **Resource server name** `orbit-MY_ID-api` · **Resource server identifier** `orbit-MY_ID-api` · choose **Add custom scope** → **Scope name** `invoke`, **Description** `Invoke my ORBIT training agent` → **Create resource server**.

   ![Create resource server form with the invoke scope](images/m05-05-resource-server-form.png)

   The list now shows two resource servers: the wizard's default one and yours. Your fully qualified scope is **`orbit-MY_ID-api/invoke`**: a bare `invoke` is not the same string.

   ![Resource servers list with the default and the orbit-s07x42-api servers](images/m05-06-resource-servers.png)

> 💡 **"Resource server" sounds like you just deployed a server. You didn't.** It is only a **name** in Cognito that groups permissions for one API. The real API is your harness. Why bother? Because in a real company one pool issues tokens for many APIs: `orders-api/read`, `orders-api/refund`, `agent-api/invoke`. The prefix keeps their permissions from colliding, and each API checks only its own. Today you are inventing a permission called `invoke` and, in Phase 4, teaching the harness to demand it.

### Step D: Authorize the scope on your app client, and find the token endpoint and secret

**Why this step?** Defining a scope (Step C) only says *the permission exists*. You must also say *which apps may ask for it*. Defining a floor in the building isn't the same as putting it on someone's badge.

9. **Applications → App clients → `orbit-MY_ID-m2m`**. The *App client information* panel shows the **Client ID** and a masked **Client secret** with a *Show client secret* link. The client ID is like a username and is not secret. The client secret is the password. Look at the secret only when you copy it into Secrets Manager in Phase 3; never paste it into chat, a screenshot, a Lambda test event or an environment variable.

   ![App client page with client ID and masked client secret](images/m05-07-app-client.png)

10. Open the **Login pages** tab. Under *Managed login pages configuration*, note **OAuth grant types: Client credentials grant** and **Custom scopes** (only the default scope so far). Choose **Edit**.
11. In **Custom scopes** open *Select custom scopes* and add **`orbit-MY_ID-api/invoke`**. Leave the client credentials grant as the only grant. **Save changes**.

    ![Edit managed login pages: client credentials grant with both custom scopes selected](images/m05-08-app-client-scopes.png)

    ![Login pages tab now listing orbit-s07x42-api/invoke](images/m05-09-app-client-login-pages.png)

    Your app may now ask for **two** scopes: the wizard's default `…/read` and your `orbit-MY_ID-api/invoke`. Only one of them will open the harness. That is exactly what the `wrong-scope` test proves later.

12. **Branding → Domain**. The wizard already created a **Cognito domain** with a unique prefix (for example `https://us-west-27vx2h98jw.auth.us-west-2.amazoncognito.com`). Record it. Your **token endpoint** is that domain plus `/oauth2/token`. You do not need to create a custom domain.

    ![Domain page showing the automatically created Cognito domain](images/m05-10-cognito-domain.png)

> [!NOTE]
> Because the wizard generates the domain prefix, 30 participants cannot collide on it. Everything else you typed carries `MY_ID`. Cognito's per-account quotas (1,000 pools, 150 token requests per second) are far above what the class needs.

**🔍 What you have now:** a badge office that will issue a signed token to exactly one app, for the scopes you allowed it, at `https://<your-domain>/oauth2/token`. Nothing can *use* those tokens yet: no API is checking them. That comes in Phase 4.

---

## 🔒 Phase 3 — Store the secret in AWS, not in code (10 minutes)

```mermaid
flowchart LR
    S["🔒 Secrets Manager<br/>◀ YOU ARE HERE"] -.-> T["🧪 Lambda test client"]
    T <-->|"① get token"| C["🪪 Cognito ✔"]
    T -->|"② Bearer JWT"| G["🛡️ JWT authorizer"]
    C -.->|"public keys"| G
    G --> H["🤖 Secure harness"] --> M["🧠 Model"]
    classDef now fill:#FF9900,color:#111,stroke:#c46f00,stroke-width:3px
    classDef done fill:#d9f2e3,color:#14532d,stroke:#3a9d63
    classDef todo fill:#eef1f4,color:#555,stroke:#c3cad3
    class S now
    class C done
    class T,G,H,M todo
```

**Why not just paste the secret into the code?** Code gets copied into Git repositories, shared in chat, pasted into tickets and shown on screen in classes like this one. A secret in code is a secret you can't take back. Secrets Manager gives you three things code can't:

- **Encryption at rest** with a KMS key.
- **A separate permission** (`secretsmanager:GetSecretValue`), so being able to *edit* a function doesn't automatically mean being able to *read* its password.
- **Rotation**: you can change the password in one place without redeploying code.

The code holds only the secret's **ARN**, which is an address. An address is useless without permission to open the door.

1. Console search → **Secrets Manager** → **Store a new secret**.
2. **Secret type:** **Other type of secret**. Under **Key/value pairs** add four rows (use **+ Add row**). The keys must match the test code exactly:

   | Key | Value | Is it secret? |
   |---|---|---|
   | `client_id` | your app client ID | no (like a username) |
   | `client_secret` | your client secret (from *Show client secret* in Phase 2) | **yes** |
   | `token_url` | your domain + `/oauth2/token` | no |
   | `scope` | `orbit-MY_ID-api/invoke` | no |

   Only one value is truly secret, but keeping all four together means the test client needs exactly **one** permission to get everything it needs to log in.

   ![Key/value pairs for the secret (the secret value is a placeholder in this screenshot)](images/m05-11-secret-key-value.png)

3. Keep the default encryption key and choose **Next**. **Secret name:** `orbit/MY_ID/cognito-m2m`. Add a description. **Next**.
4. Leave **Configure rotation** off (a training secret you will delete today). **Next**, review, then **Store**.
5. Open the secret and copy its **Secret ARN**. Note that Secrets Manager appends six random characters to the ARN (`…:secret:orbit/MY_ID/cognito-m2m-AbCdEf`). You will need the exact ARN, or the name followed by `-*`, for the IAM policy in Phase 5.

> 💡 **Why the random suffix?** If you delete a secret and later create a new one with the same name, it gets a **different** ARN. An old IAM policy that named the exact ARN won't silently grant access to the new secret. It's a small safety feature that trips up almost everyone the first time.

**Why this matters:** the ARN is a pointer; the value is credential material. IAM controls who can read the value. Do not give any role `secretsmanager:*` on `*` because it is easier.

---

## 🛡️ Phase 4 — Create the JWT-protected harness (15 minutes)

```mermaid
flowchart LR
    S["🔒 Secrets Manager ✔"] -.-> T["🧪 Lambda test client"]
    T <-->|"① get token"| C["🪪 Cognito ✔"]
    T -->|"② Bearer JWT"| G["🛡️ JWT authorizer<br/>◀ YOU ARE HERE"]
    C -.->|"public keys"| G
    G --> H["🤖 Secure harness<br/>◀ YOU ARE HERE"] --> M["🧠 Model"]
    classDef now fill:#FF9900,color:#111,stroke:#c46f00,stroke-width:3px
    classDef done fill:#d9f2e3,color:#14532d,stroke:#3a9d63
    classDef todo fill:#eef1f4,color:#555,stroke:#c3cad3
    class G,H now
    class C,S done
    class T,M todo
```

**What you are building and why:** the guard at the elevator, and the agent behind it. This harness is a near copy of your Mission 04 agent, but with a **different front door**. Instead of trusting AWS signatures, it trusts badges from *your* Cognito pool, for *your* app, carrying *your* scope.

Notice the dotted line from Cognito to the authorizer: the guard only reads the badge office's **public** keys. Cognito is **not** called for each request. That is the general trade-off of JWTs: checking them is fast and needs no call home, so the token's short lifetime (`exp`, 60 minutes by default) is what limits the damage if one ever leaks.

1. **AgentCore → Agents → More ▾ → Advanced create** (the form is titled **Create Harness**).
2. **Harness name:** `orbit_MY_ID_secure`.
3. **Model and system prompt:** the form now defaults to **Claude Sonnet 4.6**. Change it to **Anthropic → Claude Haiku 4.5** (inference profile *US Anthropic Claude Haiku 4.5*). Haiku is cheaper and faster, and today's test needs a working agent, not a brilliant one. Set the system prompt:

   > You are the ORBIT Outfitters customer support assistant. Answer in a warm, supportive voice. Never invent a returns or shipping policy, and never claim to have issued a refund or changed an order.

4. **Memory:** expand it. The form now switches memory **on** by default (*Create new Memory (managed by Harness)*, which takes 3–5 minutes to provision and is one more resource to clean up). Switch **Enable Memory** **off**. A security test needs no conversation history.

   ![Memory section switched on by default with Create new Memory selected; turn the Enable Memory toggle off](images/m05-14-harness-secure-memory.png)

5. Skip **Tools**, **Skills**, **Lifecycle hooks** and **Advanced configurations**. This agent deliberately has **no tools**: today's question is *who can talk to it*, not *what it can do*.
6. **Inbound Auth:** this is the heart of the mission. Choose **Use JSON Web Tokens (JWT)**. Under **JWT schema configuration** the form preselects *Quick create configurations with Cognito*; choose **Use existing Identity provider configurations** instead. Quick create would build a second pool that you don't control and haven't just learned. Then fill in:
   - **Discovery URL:** your OIDC configuration URL from Phase 2: the full `…/.well-known/openid-configuration` URL. Not the domain, not the JWKS URL. *(This decides whose badges are trusted.)*
   - Leave **Allowed audiences** unticked. *(Cognito access tokens have no `aud`, so this check would reject every token.)*
   - **Allowed clients:** already ticked in the current form. Enter your **app client ID**. *(Only badges issued to your app.)*
   - **Allowed scopes:** tick it and enter `orbit-MY_ID-api/invoke`. *(The badge must open this floor. The console's hint, "at least one of the required scopes", means a token that carries `invoke` plus other scopes still passes.)*
   - Leave **Custom claims** unticked.

   ![Inbound Auth section configured for JWT with discovery URL, allowed client and allowed scope](images/m05-13-harness-secure-jwt.png)

7. **Permissions:** **Create default role**, **Role name** `orbit-MY_ID-secure-role`. This is hop 4: the role the agent itself uses to call Bedrock. It has nothing to do with the JWT, and the caller never gets it.

   ![Advanced create Harness form for the secure harness](images/m05-12-harness-secure-create.png)

8. **Create Harness**. Wait for **Ready** (a few minutes: about 5 in the October 2026 run; refresh the page). On the detail page expand **Inbound Auth** and confirm it shows **JSON Web Tokens (JWT)**, your discovery URL, your client ID and your scope. Record the **Harness ARN**.

   ![Secure harness detail page with the Inbound Auth panel showing the JWT settings](images/m05-15-harness-secure-inbound.png)

> [!IMPORTANT]
> The console **playground (Test agent) signs requests with your own IAM identity**, so it cannot exercise the JWT path and may simply refuse. **Do not downgrade the harness to IAM to "make the playground work."** The Lambda client in the next phase is the valid test. That's the whole point: this front door no longer accepts AWS console identities.

**🔍 What you have now:** an agent whose only entrance checks five things about every badge: the signature, the issuer, the client, the scope and the expiry. It has its own IAM keycard (the execution role) for the model room. Nobody has knocked yet.

---

## 🧪 Phase 5 — Build an AWS-only authenticated test client (30 minutes)

```mermaid
flowchart LR
    S["🔒 Secrets Manager ✔"] -.-> T["🧪 Lambda test client<br/>◀ YOU ARE HERE"]
    T <-->|"① get token"| C["🪪 Cognito ✔"]
    T -->|"② Bearer JWT"| G["🛡️ JWT authorizer ✔"]
    C -.->|"public keys"| G
    G --> H["🤖 Secure harness ✔"] --> M["🧠 Model"]
    classDef now fill:#FF9900,color:#111,stroke:#c46f00,stroke-width:3px
    classDef done fill:#d9f2e3,color:#14532d,stroke:#3a9d63
    classDef todo fill:#eef1f4,color:#555,stroke:#c3cad3
    class T now
    class C,S,G,H done
    class M todo
```

**Concept:** a credible security test sends both **permitted and forbidden** requests to the **same** secured endpoint. A lock that has only been tested with the right key hasn't been tested. Your Lambda plays the "outside app". It reads its credentials from Secrets Manager, gets a token from Cognito over HTTPS, and calls the harness's public invoke endpoint with an `Authorization: Bearer …` header. Then it returns only sanitized status information: never the token, never the secret.

### Step A: Create the function and its configuration

1. **Lambda → Create function → Author from scratch**: **Function name** `orbit-MY_ID-jwt-test`, **Runtime** Python 3.13, default execution role. **Create Event Function**. (Newer runtimes such as 3.14 also appear and would work; 3.13 is what this guide was verified with.)

   ![Create function form for the JWT test client](images/m05-16-lambda-create.png)

2. **Configuration tab → Environment variables → Edit → Add environment variable** three times, then **Save**:

   | Key | Value |
   |---|---|
   | `SECRET_ARN` | the secret ARN from Phase 3 |
   | `HARNESS_ARN` | the secure harness ARN from Phase 4 |
   | `AWS_REGION_FOR_LAB` | `us-west-2` |

   ![Edit environment variables with the three keys](images/m05-17-lambda-env-vars.png)

   These are **ARNs and a Region name**, not credentials. Anyone who can view the function can read them, and that's fine. The secret value is read at run time, only by code whose role is allowed to.

3. **Configuration → General configuration → Edit**: set **Timeout** to **1 min 0 sec**. The default 3 seconds is too short for a token request plus a model call, which together took about 3 seconds in testing. **Save**.

   ![General configuration edit with the timeout set to 1 minute](images/m05-18-lambda-timeout.png)

### Step B: Give the function least-privilege access to the secret

**Why this step?** This is hop 1. Lambda created a role for your function that can only write logs. Without the next policy, the very first line of the test (reading the secret) fails with `AccessDeniedException`. You will grant **one action on one secret**, nothing more. This is *least privilege*, the same principle the model will explain back to you in the `good` test.

4. **Configuration → Permissions** → click the **Role name** link (`orbit-MY_ID-jwt-test-role-…`) to open IAM.

   ![IAM role page for the Lambda test client](images/m05-19-iam-lambda-role.png)

5. **Add permissions ▾ → Create inline policy → JSON**. Paste the policy below with **your** account ID, `MY_ID` and secret name. Keep the trailing `-*` (it matches the six random characters) or use the exact suffix:

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

**Why no `bedrock-agentcore:InvokeHarness` permission for this Lambda?** Because it doesn't call the harness *as an AWS identity*. It calls it over plain HTTPS with a **JWT bearer token**, exactly as a phone app or partner server would. The JWT authorizer decides, not IAM. Adding `InvokeHarness` to this role would not help: this harness's front door is configured for JWTs, not AWS signatures. (The harness execution role separately needs permission to call the model, and the default role already has it.)

### Step C: Paste the test code

7. **Code tab**: select everything in `lambda_function.py`, paste the code below, and choose **Deploy**. It uses only Python's standard library plus the AWS SDK (`boto3`) that Lambda already includes.

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
            return resp.status, resp.headers, resp.read(65536).decode("utf-8", "replace")
    except urllib.error.HTTPError as exc:
        # Expected for missing, forged, tampered, or expired credentials.
        return exc.code, exc.headers, exc.read(65536).decode("utf-8", "replace")


def get_token(secret, scope):
    # OAuth 2.0 client credentials grant: prove WHO we are (id + secret), ask for WHAT we need (scope).
    pair = f"{secret['client_id']}:{secret['client_secret']}"
    basic_auth = base64.b64encode(pair.encode()).decode()
    payload = urllib.parse.urlencode({
        "grant_type": "client_credentials",
        "scope": scope,
    }).encode()
    return http_post(
        secret["token_url"], payload,
        {"Content-Type": "application/x-www-form-urlencoded",
         "Authorization": f"Basic {basic_auth}"},
    )


def lambda_handler(event, context):
    # event mode: good, missing, forged, tampered, wrong-scope
    mode = (event or {}).get("mode", "good")
    if mode not in ("good", "missing", "forged", "tampered", "wrong-scope"):
        return {"error": "Choose good, missing, forged, tampered, or wrong-scope"}

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
    if mode in ("good", "tampered", "wrong-scope"):
        # wrong-scope: a genuine token for a scope the harness does NOT allow.
        scope = event.get("scope", "") if mode == "wrong-scope" else secret["scope"]
        token_status, _, response_text = get_token(secret, scope)
        if token_status != 200:
            return {"mode": mode, "token_status": token_status,
                    "phase": "Cognito token issuance failed",
                    "body_preview": response_text[:300]}
        access_token = json.loads(response_text)["access_token"]
        if mode == "tampered":
            # A real, well-formed Cognito token whose signature no longer matches its contents.
            access_token = access_token[:-6] + ("AAAAAA" if not access_token.endswith("AAAAAA") else "BBBBBB")
        headers["Authorization"] = f"Bearer {access_token}"
    elif mode == "forged":
        headers["Authorization"] = "Bearer this.is.not.a.valid.jwt"

    body = json.dumps({"messages": [{"role": "user", "content": [
        {"text": "In one sentence, explain least privilege."}]}]}).encode()
    status, resp_headers, response_text = http_post(runtime_url, body, headers)

    # A successful harness call streams events; pull out just the model's text.
    if status == 200:
        pieces = re.findall(r'"text":"((?:[^"\\\\]|\\\\.)*)"', response_text)
        response_text = "".join(pieces).encode().decode("unicode_escape", "replace") or response_text

    # Keep credentials and JWTs out of logs and returned data.
    return {
        "mode": mode,
        "token_status": token_status,
        "runtime_http_status": status,
        "error_type": (resp_headers.get("x-amzn-ErrorType") or "").split(":")[0] or None,
        "body_preview": response_text[:300],
    }
```

   ![Lambda code editor with the deployed JWT test client](images/m05-21-lambda-code.png)

   **Read the code once before running it.** Each piece maps to a box on the diagram:

   | Code | What it does | Hop |
   |---|---|---|
   | `get_secret_value(SecretId=os.environ["SECRET_ARN"])` | Opens the safe using the Lambda role's one permission | 1 · IAM |
   | `get_token()` | The client credentials grant: sends `client_id:client_secret` as HTTP *Basic* auth plus `grant_type` and `scope` to the token endpoint, and receives a signed JWT | 2 · OAuth |
   | `headers["Authorization"] = f"Bearer {access_token}"` | Presents the badge. *Bearer* means "whoever bears this token is the caller", which is why tokens must be guarded like passwords | 3 · JWT |
   | `runtime_url` | The harness's public invoke URL: `https://bedrock-agentcore.<region>.amazonaws.com/harnesses/invoke?harnessArn=<URL-encoded ARN>` | 3 |
   | `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` | A harness conversation is a **session** (an isolated micro-VM, as in Mission 04), and each needs an ID of at least 33 characters | 3 |
   | `re.findall(r'"text":…')` | A successful answer arrives as a **stream of events**, the same way a chat UI shows words as they are generated, so the code stitches the text pieces together | 4 |
   | `error_type` | AgentCore names its rejection in an `x-amzn-ErrorType` response header. Often that header is the *only* clue, because the body is empty | 3 |
   | `return {…}` | Returns statuses and a short preview, and **never** the token or the secret | – |

   And the two "attack" modes:
   - **`tampered`** gets a **genuine** token from Cognito, then changes its last six characters (part of the signature). The header and payload still decode perfectly and still claim to be from your pool, client and scope. Only the cryptographic check can tell it was altered.
   - **`wrong-scope`** asks Cognito for a **genuine** token for a **different** scope that your app client *is* allowed: the wizard's default `…/read`. Cognito happily issues it. The question is whether the harness accepts a real badge for the wrong floor.

### Step D: Run the five tests

8. **Test tab**: with **Create new event** selected, set the Event JSON to `{"mode": "good"}` and choose **Test**. You can also **Save** it as an event named `good` so you can rerun it later. Expand **Details** and read the **Response**. Repeat with `{"mode": "missing"}`, `{"mode": "forged"}` and `{"mode": "tampered"}`.

   ![good: token_status 200 and runtime_http_status 200 with the model's one-sentence answer](images/m05-22-test-good.png)

   ![missing: runtime_http_status 403, AccessDeniedException, Missing Authentication Token](images/m05-23-test-missing.png)

   ![forged: runtime_http_status 403, AccessDeniedException, OAuth authorization failed: Failed to parse token](images/m05-24-test-forged.png)

   ![tampered: token_status 200, runtime_http_status 500, UnrecognizedClientException, empty body](images/m05-25-test-tampered.png)

9. **The authorization test.** Use the default scope you wrote down in Phase 2 (on **Applications → Resource servers** it is the `default-m2m-resource-server-xxxxxx` row, scope `read`):

   ```json
   {"mode": "wrong-scope", "scope": "default-m2m-resource-server-xxxxxx/read"}
   ```

   ![wrong-scope: Cognito issues the token (200), the harness rejects it (500, UnrecognizedClientException)](images/m05-26-test-wrong-scope.png)

   Then try a scope that doesn't exist at all, such as `{"mode": "wrong-scope", "scope": "orbit-MY_ID-api/admin"}`. This time it never reaches the harness: **Cognito** refuses to issue the token (`token_status: 400`, `invalid_scope`). The badge office won't print a floor that doesn't exist.

**Verified results (October 25, 2026):**

| Test mode | Cognito token exchange | Which check said no | Observed |
|---|---|---|---|
| `good` | `200` | none: issuer, client, scope, signature and expiry all valid | `200`, preview: *"Least privilege is the security principle of granting…"* |
| `missing` | not attempted | AgentCore: no bearer token at all | `403`, `AccessDeniedException`, `{"message":"Missing Authentication Token"}` |
| `forged` | not attempted | AgentCore: not even a JWT | `403`, `AccessDeniedException`, `{"message":"OAuth authorization failed: Failed to parse token"}` |
| `tampered` | `200` | AgentCore: **signature** check | `500`, `UnrecognizedClientException`, empty body |
| `wrong-scope` (default `…/read`) | `200` | AgentCore: **scope** check | `500`, `UnrecognizedClientException`, empty body |
| `wrong-scope` (`…/admin`) | `400` `invalid_scope` | **Cognito**: no such scope | the harness was never called |

### 🔬 Read the evidence like a security engineer

Here is where each request was stopped:

```mermaid
sequenceDiagram
    autonumber
    participant L as 🧪 Lambda client
    participant C as 🪪 Cognito
    participant A as 🛡️ JWT authorizer
    participant H as 🤖 Harness + model
    Note over L,H: good
    L->>C: client ID + secret, scope …/invoke
    C-->>L: 200 · signed JWT
    L->>A: Bearer JWT
    A->>H: ✅ signature, iss, client_id, scope, exp all OK
    H-->>L: 200 · streamed answer (~3 s)
    Note over L,H: missing / forged
    L->>A: no token, or "this.is.not.a.valid.jwt"
    A-->>L: 403 AccessDeniedException (~0.3 s)
    Note over L,H: tampered / wrong-scope
    L->>C: client ID + secret
    C-->>L: 200 · genuine JWT
    L->>A: altered signature, or scope …/read
    A-->>L: 500 UnrecognizedClientException, empty body (~0.6 s)
    Note over L,H: scope that doesn't exist
    L->>C: scope …/admin
    C-->>L: 400 invalid_scope (harness never called)
```

Three lessons are hiding in those results.

1. **Two different "No" families.** If the request has *no* token, or something that isn't a JWT, AgentCore answers **403** and tells you why. If it is a **real, well-formed JWT** that fails a check (bad signature, wrong client, wrong scope), AgentCore answers **500 `UnrecognizedClientException` with an empty body**. A 500 *usually* means "server bug", so this surprises people. Here it means *the authorizer looked at your token and refused it*. Read the `error_type`, not just the number. The empty body is also good security practice: the caller isn't told *which* check failed, so an attacker gets no hints.
2. **The model never ran for any rejection.** Compare **Duration** under *Summary*. `good` took about 3 seconds because Claude actually generated an answer. Every rejection finished in under a second, including `tampered` and `wrong-scope`, which still spent time fetching a token from Cognito first. Rejection happens **at the door**, before the agent spends a single model token. That's what protects your bill as well as your data.
3. **Authentication is not authorization.** The `wrong-scope` token is 100% genuine: signed by your pool, issued to your app, not expired. It was still refused, because it carried permission for the wrong thing. Two different services enforced scope: **Cognito** won't *issue* a scope the client isn't allowed or that doesn't exist, and **AgentCore** won't *accept* a token whose scope isn't on its list.

> [!CAUTION]
> **Do not paste token values** into screenshots, chats, logs or this guide. The function deliberately does not print or return them. Restrict who may run or edit your test Lambda: the ability to invoke a client that holds a secret is itself sensitive. Anyone who can run it can get a valid token.

### Step E: Troubleshoot the right side of the boundary

First decide **which hop** failed: `token_status` belongs to Cognito (hop 2), `runtime_http_status` and `error_type` belong to AgentCore (hop 3), and a Lambda error before either belongs to IAM (hop 1).

```mermaid
flowchart TD
    Start["Run a test"] --> E{"Lambda error before<br/>any status?"}
    E -->|"AccessDeniedException<br/>on GetSecretValue"| IAM["Hop 1 · IAM<br/>Check the inline policy's secret ARN"]
    E -->|"Task timed out"| TO["Lambda timeout is still 3 s"]
    E -->|no| T{"token_status?"}
    T -->|"400"| COG["Hop 2 · Cognito<br/>Read the error: invalid_client,<br/>invalid_scope, invalid_grant"]
    T -->|"200 or null"| R{"runtime_http_status<br/>+ error_type"}
    R -->|"200"| OK["✅ Accepted"]
    R -->|"403 AccessDeniedException"| NOTOK["Hop 3 · No token, or not a JWT"]
    R -->|"500 UnrecognizedClientException"| JWT["Hop 3 · Real token refused:<br/>discovery URL, Allowed clients,<br/>Allowed scopes"]
    R -->|"other 5xx"| RUN["Inside the harness:<br/>Ready? model access? CloudWatch"]
    classDef bad fill:#ffe7e7,stroke:#c74e4e,color:#5b2222
    class IAM,TO,COG,NOTOK,JWT,RUN bad
```

| Symptom | Likely layer | Correct console check |
|---|---|---|
| Lambda error `AccessDeniedException` on `GetSecretValue` | Lambda execution role (hop 1) | Inline policy resource matches the **exact** secret ARN (or `-*`); same Region and account |
| `token_status` 400 `invalid_client` | Cognito app client (hop 2) | Client ID and secret in the secret value come from the **same** app client; the client has **client credentials** enabled |
| `token_status` 400 `invalid_scope` | Cognito scope definition (hop 2) | The scope doesn't exist as spelled. Check the resource server identifier and scope name: `orbit-MY_ID-api/invoke`, not `invoke` or `orbit-MY_ID-api/Invoke` |
| `token_status` 400 `invalid_grant` | Cognito scope assignment (hop 2) | The scope exists but isn't enabled on this app client: Phase 2, Step D |
| `token_status` 404 or a DNS error | token URL | Must be the **domain** + `/oauth2/token`, not the issuer URL |
| `good` returns `403` `AccessDeniedException` | AgentCore inbound (hop 3) | The `Authorization` header wasn't sent. Did the code change? |
| `good` returns `500` `UnrecognizedClientException` | AgentCore inbound JWT (hop 3) | A real token was refused. Discovery URL points at the **same** pool; **Allowed clients** has this exact client ID; **Allowed scopes** is `orbit-MY_ID-api/invoke`; **Allowed audiences** is **unticked** |
| `good` returns another 5xx with a body | Harness process or model (hop 4) | Harness status Ready; execution role; CloudWatch log group `/aws/bedrock-agentcore/runtimes/harness_orbit_MY_ID_secure-…` |
| `Task timed out after 3.00 seconds` | Lambda timeout | Phase 5, Step A, item 3 |
| `missing`, `forged`, `tampered` or `wrong-scope` returns **200** | Security misconfiguration | **Stop.** The harness is not enforcing JWT, or not checking scope. Check Phase 4 |
| *The user pool cannot be deleted because deletion protection is activated* | Cognito cleanup | Tick **Deactivate deletion protection** (and **Delete Cognito domain**) in the Delete dialog (cleanup step 4) |

**Boss challenge — audience confusion:** a teammate sets **Allowed audiences** to the app client ID but leaves **Allowed clients** empty. Why is a perfectly good Cognito access token rejected? Because the authorizer compares *Allowed audiences* with the `aud` claim, and Cognito **access** tokens carry `client_id` instead (Cognito *ID* tokens, which are for humans, do carry `aud`). The two checks validate **different claims**. Predict the `runtime_http_status` and `error_type` they would see, using what you learned above.

**Boss challenge — issuer mismatch (analysis only):** an app's token is signed by Pool A but the harness's discovery URL points at Pool B. What should AgentCore do? Reject it: the signature cannot be verified with Pool B's keys, and `iss` does not match. It's the same family of failure as your `tampered` test. Do not modify your working authorizer to prove this; reason it through from the Inbound Auth panel.

---

## 🔎 Phase 6 — What you have secured, and what you have not (10 minutes)

```mermaid
flowchart LR
    X["🌍 Anyone on the internet"] -->|"no token / junk"| G
    Y["🦹 Attacker with an altered token"] -->|"bad signature"| G
    Z["📱 Real app, wrong permission"] -->|"wrong scope"| G
    APP["✅ Your app, right scope"] -->|"valid JWT"| G["🛡️ JWT authorizer"]
    G -->|"only this one passes"| H["🤖 Secure harness"]
    H --> M["🧠 Model"]
    H -.->|"NOT covered today"| TOOL["🔧 Tools: may THIS caller<br/>refund THIS order?"]
    classDef stop fill:#ffe7e7,stroke:#c74e4e,color:#5b2222
    classDef ok fill:#d9f2e3,color:#14532d,stroke:#3a9d63
    classDef gap fill:#fff4d6,stroke:#d9a400,color:#5a4500
    class X,Y,Z stop
    class APP ok
    class TOOL gap
```

Complete the threat map in your notes. The *Your evidence* column is filled in for the threats you actually tested:

| Threat | Which control helps? | Your evidence | Remaining question |
|---|---|---|---|
| Random internet caller tries the invoke URL | JWT inbound authorizer | `missing` → 403 | Are issuer, client and scope checks all configured? |
| Attacker fabricates a token | JWT structure check | `forged` → 403 | – |
| Attacker alters a real token (e.g. edits its scope) | Signature verification via the discovery URL | `tampered` → 500 `UnrecognizedClientException` | Are the signing keys fetched from the right pool? |
| A genuine app asks for more than it should | Allowed scopes, plus Cognito scope assignment | `wrong-scope` → 500; `…/admin` → 400 `invalid_scope` | Does every app client get only the scopes it needs? |
| A valid app tries an unauthorized refund | Tool-level authorization + business policy | not tested: no tools today | Does the refund tool check the customer's identity and order? |
| The agent process reads secrets unrelated to it | Execution role scoped by ARN | | Did anyone attach `secretsmanager:*`? |
| "Ignore your rules" text hidden in a retrieved document | Treat retrieval as untrusted data | | Does the agent separate instructions from source text? |
| One student's token used against another's harness | Different app clients, scopes and authorizers | | Is **Allowed clients** specific to your `MY_ID`? |
| A valid token is stolen and replayed | Short `exp` (60 min), TLS, never logging tokens | | How long would a stolen token work? |

**Key insight:** *Authenticated* is not *authorized for every action*. A token proves an approved app is calling, and scopes narrow *what kind* of call it may make. A production refund tool would still have to decide whether this app, on behalf of **this** customer, may touch **this order** and run **this refund**.

### How this looks in a real ORBIT deployment

Today's M2M flow is how one **service** talks to another (a partner's order system calling ORBIT's agent, say). When **people** use the agent through a website, the front door is identical, but the badge is collected differently:

```mermaid
flowchart LR
    U["🧑 Customer on the<br/>ORBIT website or app"] -->|"① sign in, get token<br/>authorization code + PKCE"| COG["🪪 Cognito user pool"]
    P["🏭 Partner order system"] -->|"① get token<br/>client credentials · today's flow"| COG
    U -->|"② Bearer JWT"| A["🛡️ JWT authorizer<br/>same settings as today"]
    P -->|"② Bearer JWT"| A
    A --> H["🤖 ORBIT agent"] --> GW["🌐 Gateway → order tools<br/>Mission 04"]
    classDef today fill:#FF9900,color:#111,stroke:#c46f00
    class P,A today
```

Three things change in production, and you've already met each one:

- **Human sign-in** uses the *authorization code* grant, so a customer's token carries **their** identity, which tools can use to restrict which orders they can see.
- **The Mission 04 gateway** has the same **Inbound Auth** choice as a harness, so tools can be locked behind JWTs too.
- **Outbound identity** (next section) handles the opposite direction: the agent proving *itself* to someone else's API.

### Optional inspection — outbound identity

Open **AgentCore → Connections → Outbound Auth** (this page was called *Identity* until late October 2026). Its *How it works* panel explains inbound and outbound auth side by side; the list shows the account's outbound credential providers. Do not create providers for outside services. Answer: how is **inbound JWT validation** (who may call me) different from **outbound OAuth credentials** (what I may call downstream)? *Hint: today the badge office was Cognito and the guard was AgentCore. For outbound calls, AgentCore becomes the visitor collecting badges from someone else's office.*

---

## 🔭 Production lens: authentication is the first of several gates

Your JWT authorizer proves *which app* is calling. It says nothing about *what that app may make the agent do*. A real deployment adds, in order:

1. **User identity in the token** (authorization-code sign-in, as in Mission 02), so tools can restrict a customer to *their own* orders.
2. **Policy on the gateway** that reads token claims: a Cedar `permit` can require `principal.getTag("scope")` to contain a refund scope before `issue_refund` runs. Mission 06 builds the gateway policy with IAM callers; the same engine reads JWT claims when the gateway uses JWT inbound auth.
3. **Guardrails, limits and evaluations** on the agent itself (Mission 06).

The question to keep asking is the one from Phase 6: *for each threat, which control helps, and what is the evidence?*

---

## 🏁 Final scorecard — Earn the Identity Badge

- [ ] I used **only the AWS Console** and AWS-hosted execution.
- [ ] I created my **own Cognito user pool**, M2M app client, resource server and scope, all carrying `MY_ID`.
- [ ] I can explain why the **issuer** (discovery URL) and the **domain** (token endpoint) are different addresses, and which service uses each.
- [ ] I stored the client secret in **Secrets Manager**, never in code or environment variables, and gave the Lambda role access to **one** secret.
- [ ] I created an AgentCore **Harness with a JWT inbound authorizer** pointing at my pool, my client and my scope.
- [ ] My Lambda test obtained an M2M token and got **200** from the secured harness.
- [ ] I captured **403** for the missing and forged tokens, and **500 `UnrecognizedClientException`** for the tampered and wrong-scope tokens, and can explain why the two families differ.
- [ ] I can explain why a **genuine** token with the wrong scope is rejected: authentication ≠ authorization.
- [ ] I can explain `iss`, `client_id`, `scope`, `exp`, why `aud` is absent, and the four hops with their separate credentials.
- [ ] I can explain why M2M authentication is **not** interactive user sign-in.

---

## 🧯 Quotas, costs, and cleanup — mandatory in a shared account

**What this mission costs** (US West (Oregon) list prices, October 2026):

| Item | Price | Typical use | Cost per participant |
|---|---|---|---|
| Cognito M2M token requests | $0.00225 per successful request, no free tier | 5–15 requests | $0.01–0.03 |
| Cognito user pool (Essentials tier) | free up to 10,000 monthly active users | 0 users | $0 |
| Secrets Manager | $0.40 per secret per month, prorated hourly | one secret for a few hours | < $0.01 (**$0.40 per month if left behind**) |
| Secure harness: Claude Haiku 4.5 + Runtime sessions | as in Mission 04 | 3–6 short successful calls (rejected calls never reach the model) | about $0.03 |
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
4. **Cognito:** open `orbit-MY_ID-pool` and choose **Delete user pool** (top right of the overview). The wizard switched on **deletion protection** and created a **domain**, and the dialog offers to handle both: tick **Delete Cognito domain … that you assigned** and **Deactivate deletion protection**, type the pool name, and choose **Delete**. The app client and resource servers are removed with the pool.

   ![Delete user pool dialog with both prerequisite actions ticked and the pool name typed](images/m05-27-cognito-delete-pool.png)
5. **IAM:** delete `orbit-MY_ID-secure-role` and the auto-created `orbit-MY_ID-jwt-test-role-…` only after their resources are gone. Before deleting each role, open its **Permissions** tab and note the customer-managed policy the console created alongside it (`AmazonBedrockAgentCoreHarnessExecutionPolicy_…` and `AWSLambdaBasicExecutionRole-…`). Deleting a role does not delete those, so remove them afterwards from **IAM → Policies**.
6. Return to the Mission 04 and Mission 03 cleanup lists for the gateway, Lambda tool, first harness and knowledge base.

**AWS docs to consult:** [Cognito resource servers and scopes](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html) · [Cognito app clients](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html) · [Cognito token endpoint](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html) · [AgentCore inbound JWT authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html) · [Harness security and access controls](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-security.html) · [Cognito pricing](https://aws.amazon.com/cognito/pricing/) · [Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/).

*End of Mission 05 · ORBIT's agent perimeter is now something you can prove with real AWS requests, not merely describe.*
