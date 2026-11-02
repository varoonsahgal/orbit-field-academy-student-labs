# 🛰️ ORBIT FIELD ACADEMY
## MISSION 02 · IGNITION
### AWS Lambda, Amazon CloudWatch, a Bedrock model call from code, and the services an agent depends on

> **LEVEL:** Recruit · **TARGET TIME:** 60–75 minutes (plus optional challenges) · **ENVIRONMENT:** AWS Management Console (CloudShell for two challenges)
> **THE STORY:** Ground Control cleared you for ignition. Today you run your first code in the cloud, read its telemetry, call a foundation model from that code, and hit the permission wall every agent builder hits once. Then you walk the launch complex: Amazon Bedrock, AgentCore, Amazon Cognito, and Amazon OpenSearch Service, the four systems the ORBIT support agent will depend on in the missions ahead.

**You already created the role for this mission.** Mission 01 left you with `orbit-MY_ID-lambda-role`, trusted by Lambda and allowed only to write logs. Keep the ledger from Mission 01 at hand.

> [!NOTE]
> **Verified against the live AWS Console on October 21, 2026** in US West (Oregon) as participant `s07x42`. The Lambda console now opens a **VS Code-based editor** inside the browser and calls functions **Event Functions**; both are reflected below. The OpenSearch domain steps in Phase 8 were written from the console's documented layout; if a label differs, look for the nearest equivalent. **Updated November 2, 2026:** the AgentCore tour (Phase 6) uses the reorganised AgentCore console, and the Cognito phase (Phase 7) was rewritten after a class found it confusing: it is now explicitly read-only, uses the console's **View login page** button instead of a URL from the whiteboard, and explains the expected *This site can't be reached* screen before you meet it.

---

## 🎯 Mission outcomes

By the end, you will be able to:

- Create and test an **AWS Lambda** function in the console, and explain *handler*, *event*, *context*, *execution role*, *timeout* and *cold start*.
- Find a function's **CloudWatch** log group, read a `REPORT` line, and run a **Logs Insights** query.
- Call an **Amazon Bedrock** model from code with the Converse API, read the **AccessDenied** message like a pro, and fix it with a **least-privilege inline policy**.
- Navigate the **Bedrock** model catalog and the **AgentCore** console, and explain what an **execution role for an AgentCore workload** must look like.
- Inspect a **Cognito user pool** and explain how it issues the JWTs that authenticate agent callers.
- Inspect an **OpenSearch** domain and explain its role in retrieval (index, document, vector field).

---

## 📖 Before you start: the concepts in this mission

### The launch complex: what an agent depends on

```mermaid
flowchart LR
    C[Caller app] -->|JWT| COG[Amazon Cognito user pool]
    C --> AC[AgentCore Harness / Runtime]
    AC -->|InvokeModel| B[Amazon Bedrock model]
    AC -->|tool call via Gateway| L[AWS Lambda function]
    AC -->|retrieve| OS[OpenSearch index / knowledge base]
    AC --> CW[CloudWatch logs, metrics, traces]
    L --> CW
    subgraph IAM[IAM execution roles decide every arrow]
    end
    classDef focus fill:#0B243B,color:#FFFFFF,stroke:#24CAD5
    class AC,L,B focus
```

### AWS Lambda in six terms

| Term | Meaning |
|---|---|
| **Function** | Your code plus its configuration (runtime, memory, timeout, role). Billed per request and per millisecond. |
| **Handler** | The function Lambda calls: `lambda_function.lambda_handler`. Receives `event` (the JSON input) and `context` (facts about the run: request ID, time remaining, memory). |
| **Execution role** | The IAM role the running code *becomes*. Lambda assumes it (the trust policy you read in Mission 01) and gives the code temporary credentials. Every AWS call your code makes is authorised against **this** role, not against you. |
| **Timeout** | Default **3 seconds**. A model call can take longer, so you raise it. Maximum 15 minutes. |
| **Cold start / Init** | The first invocation builds the execution environment (`INIT_START` … `Init Duration` in the log). Later calls reuse it. |
| **Deploy** | In the console editor, saving a file is not enough: **Deploy** (⇧⌘U on Mac, Ctrl+Shift+U on Windows) ships the code. |

### Amazon CloudWatch in four terms

**Log group** (one per function, `/aws/lambda/<name>`) → **log streams** (one per execution environment) → **log events** (each line your code prints, plus Lambda's own `START`, `END`, `REPORT`). **Metrics** (Invocations, Duration, Errors, Throttles) are emitted automatically; **alarms** watch a metric; **Logs Insights** is a query language over log events (`fields`, `filter`, `stats`, `sort`).

### Amazon Bedrock in four terms

- **Model catalog:** every foundation model available in the Region, with its **model ID** (`amazon.nova-lite-v1:0`), modalities, pricing and status.
- **On-demand vs. inference profile:** some models accept the plain model ID in this Region; others are reached through a **cross-Region inference profile** (`us.amazon.nova-micro-v1:0`), which routes to several Regions. An IAM policy for a profile must allow the profile ARN *and* the model ARN in every Region it can route to.
- **Converse API:** one request shape for every chat model: `system`, `messages`, `inferenceConfig`. Your Lambda uses it.
- **Tokens:** the unit of billing. The response reports `inputTokens` and `outputTokens`.

### AgentCore execution roles

An AgentCore Harness or Runtime runs *as* an IAM role, exactly like Lambda does. Its trust policy names **`bedrock-agentcore.amazonaws.com`** (the mystery role from Mission 01 is the pattern), and its permissions policy typically allows `bedrock:InvokeModel*` on the models it may use, `bedrock-agentcore:InvokeGateway` on its gateways, `logs:*` on its own log groups, and nothing else. Mission 04 creates these roles for real.

### Amazon Cognito in four terms

**User pool** (a user directory and OAuth 2.0 / OpenID Connect server) → **app client** (an application registered with the pool; public clients have no secret, confidential clients do) → **hosted login pages** on a **domain** (where users sign in) → **tokens** (JWTs: an ID token about the person, an access token about the permission scopes). The pool publishes its **issuer** URL, a **discovery document** and its **JWKS** public keys so that any service, including AgentCore, can verify a token without calling Cognito.

**Why look at Cognito now, before any agent exists?** Because every agent you build later has a front door, and somebody has to hand out the keys. Today you only *watch* a person collect one, on the instructor's shared pool. In Mission 05 you build your *own* pool for a program instead of a person and lock an agent behind it.

| | **Today (Mission 02)** | **Mission 05** |
|---|---|---|
| Pool | `orbit-academy-pool`, shared, created by the instructor | `orbit-MY_ID-pool`, yours, you create it |
| Who signs in | A **person** (your `recruitNN` user) on a hosted login page | A **program** with a client ID and a secret |
| OAuth flow | **Authorization code** (browser redirect) | **Client credentials** (machine to machine) |
| What you do | Read the settings, sign in once, look at what comes back | Build everything, then test accepted and rejected calls |
| May you change it? | **No**, read-only | **Yes**, and you delete it at the end |

### Amazon OpenSearch Service in four terms

A **domain** is a managed OpenSearch cluster (nodes in one or more AZs). It holds **indexes** of JSON **documents**; an index **mapping** declares each field's type, including `knn_vector` for embeddings. Retrieval-augmented agents embed the question, run a **k-NN** search for the nearest vectors, and hand the matching documents to the model. Access is controlled by the domain's **resource-based access policy** plus IAM (and optionally fine-grained access control).

---

## 🪪 Phase 0 — Preflight (5 minutes)

1. Sign in, set the Region to **United States (Oregon)**, and have your `MY_ID` and ledger from Mission 01 ready.
2. Naming for this mission:

| What | Name you will type |
|---|---|
| Lambda function | `orbit-MY_ID-hello` |
| Test event | `ignition-check` |
| Inline policy on your role | `orbit-MY_ID-bedrock-invoke` |
| (Challenge) CloudWatch alarm | `orbit-MY_ID-hello-errors` |

3. **Two values for Phase 7** (you need nothing else from the instructor; the console gives you the rest):

| Item | Value |
|---|---|
| Your recruit user name | `recruitNN`, where **NN is your participant number** (sign-in name `participant-07` → `recruit07`) |
| Temporary password | announced by the instructor (the same for every recruit until first sign-in) |

> [!CAUTION]
> `orbit-academy-pool` (Cognito) and `orbit-academy-search` (OpenSearch) are **shared and read-only**, created by the instructor before class. **In this mission you create nothing in Cognito.** If you find yourself on a *Create user pool* or *Create app client* page, you have wandered into Mission 05; go back. Your student permissions deny changes to the shared resources, so an *Access denied* there means you clicked something you were not meant to.

**Resource ledger**

| Property | Your value |
|---|---|
| Function ARN | |
| Log group name | `/aws/lambda/orbit-MY_ID-hello` |
| Request ID of your first successful Bedrock call | |

---

## 🚀 Phase 1 — Create your function (10 minutes)

**Concept:** A function is *code + configuration + role*. You bring the role; the console builds the rest.

1. Search for `Lambda` and open it. In the left navigation choose **Event Functions**, then **Create function** (the console calls them *Event Functions* now; they are ordinary Lambda functions).

   ![Create Event Function page with Author from scratch selected](images/m02-01-lambda-create-page.png)

2. Keep **Author from scratch** and fill in **Basic information**:
   - **Function name:** `orbit-MY_ID-hello`
   - **Runtime:** open the dropdown, type `Python`, choose **Python 3.13** (any non-Preview Python works; 3.13 is what the verified run used).
3. Expand **Additional settings** at the bottom. Under **General**, switch on **Custom execution role**. A side panel **Configure custom execution role** opens: under **Execution role** choose **`orbit-MY_ID-lambda-role`** from the dropdown and choose **Save**.

   ![Additional settings with Custom execution role switched on and the role chosen](images/m02-02-lambda-create-form.png)

   ![The Configure custom execution role panel](images/m02-02b-lambda-role-panel.png)

   Why not let the console create a role? It would, and it would be fine, but you would learn nothing about trust policies. The role you built by hand is the one the function will assume.

4. Choose **Create Event Function**. After a few seconds the function page opens with the green banner *Successfully created the function*. A **Getting started** dialog offers local development paths; choose **Dismiss**.

   ![The new function's page with the Code tab and the in-browser editor](images/m02-03-lambda-created.png)

5. Record the **Function ARN** in your ledger. Note the **Runtime settings** card at the bottom: Handler `lambda_function.lambda_handler`, architecture `x86_64`.

**🧩 Investigation checkpoint**

- [ ] The function's **Configuration → Permissions** tab shows the execution role `orbit-MY_ID-lambda-role` (open it to check).
- [ ] I can explain why creating a function needed `iam:PassRole` on that role.

---

## 💻 Phase 2 — Write, deploy, test (10 minutes)

**Concept:** The in-browser editor is VS Code. Edits are local until you **Deploy**. **Test** invokes the function with a JSON event you define and shows the response, a billing summary and the log tail.

1. On the **Code** tab the editor shows `lambda_function.py` with the default *Hello from Lambda* code. Click inside the editor, select all (⌘A / Ctrl+A), delete, and paste this:

   ```python
   # ORBIT Field Academy - Mission 02 - Step 1
   # Runs ONLY inside AWS Lambda. No external dependencies.
   import json
   import os
   import time


   def lambda_handler(event, context):
       """Lambda calls this function once per invocation.
       event   = the JSON input (whatever invoked us sent)
       context = facts about this run (function name, request ID, time left)
       """
       recruit = (event or {}).get("recruit", "unknown recruit")
       print(f"ORBIT telemetry: event received for {recruit}")
       print("ORBIT telemetry: raw event =", json.dumps(event))

       return {
           "statusCode": 200,
           "mission": "02-IGNITION",
           "greeting": f"Ground Control to {recruit}: ignition confirmed.",
           "function_name": context.function_name,
           "request_id": context.aws_request_id,
           "region": os.environ.get("AWS_REGION"),
           "memory_mb": context.memory_limit_in_mb,
           "ms_remaining": context.get_remaining_time_in_millis(),
           "unix_time": int(time.time()),
       }
   ```

2. **Deploy:** press ⇧⌘U (Mac) or Ctrl+Shift+U (Windows), or use the **DEPLOY** section in the editor's left panel. The status changes to **✓ Current · Lambda Deployed** and a banner says *Successfully updated the function*.

   ![The editor after deploying the code](images/m02-04-lambda-code-editor.png)

3. Open the **Test** tab (the tab next to *Code*, outside the editor). Under **Test event**:
   - **Test event action:** Create new event
   - **Event name:** `ignition-check`
   - **Event JSON:** replace the template with

     ```json
     {
       "recruit": "MY_ID",
       "question": "In one sentence, what does a Lambda execution role do?"
     }
     ```
   - Choose **Save** (so the event survives page reloads), then the orange **Test**.

   ![The Test tab with the ignition-check event](images/m02-05-lambda-test-event.png)

4. A green panel appears: **Executing function: succeeded**. Expand **Details**:
   - **Response:** your JSON, including the greeting and the request ID. Copy the request ID to your ledger.
   - **Summary:** *Duration* (a few milliseconds of your code), *Billed duration*, *Resources configured* 128 MB, *Max memory used* about 36 MB, and on the first run an **Init duration** (the cold start).
   - **Log output:** `START`, your two `ORBIT telemetry` lines, `END`, `REPORT`.

   ![Test result with Response, Summary and Log output](images/m02-06-lambda-test-result.png)

5. Choose **Test** again. No *Init duration* this time: the execution environment was reused.

**🧩 Investigation checkpoint**

- [ ] I can point to the `event` and the `context` values in the response.
- [ ] I know the difference between *Duration* and *Billed duration*, and what *Init duration* was.

---

## 📈 Phase 3 — Telemetry: CloudWatch logs, metrics, and Logs Insights (10 minutes)

**Concept:** Everything your function prints goes to CloudWatch Logs through the execution role's `logs:*` permissions (that is what `AWSLambdaBasicExecutionRole` was for). Metrics come free. This is where you will debug agents later.

1. Open the **Monitor** tab. The graphs (Invocations, Duration, Error count and success rate, Throttles…) fill within a minute or two of your tests.

   ![Monitor tab with the Lambda metrics graphs](images/m02-07-lambda-monitor.png)

2. Choose **View CloudWatch logs** (top right of the Monitor tab), or search for `CloudWatch` → **Log management** → `/aws/lambda/orbit-MY_ID-hello`. The log group page lists **log streams**, one per execution environment; the stream name contains the date and `[$LATEST]`.

   ![The function's log group with one log stream](images/m02-08-cw-log-group.png)

3. Open the newest stream. Each row is a log event. Read the `REPORT` line: `Duration`, `Billed Duration`, `Memory Size`, `Max Memory Used`, and (first run) `Init Duration`. This line is the raw material for every Lambda cost and performance question.

   ![Log events with the ORBIT telemetry lines and the REPORT line](images/m02-09-cw-log-events.png)

4. In the CloudWatch left navigation choose **Logs → Log Analytics** (the console's new name for Logs Insights). Under **Query scope** keep *1h* and set **Log group =** `/aws/lambda/orbit-MY_ID-hello`, paste the query below into the editor, and choose **Run** (⌘+Enter / Ctrl+Enter). Two `REPORT` rows appear, newest first, with the durations you saw on the Test tab:

   ```
   fields @timestamp, @message
   | filter @message like /REPORT/
   | sort @timestamp desc
   ```

   ![Logs Insights query returning the REPORT lines](images/m02-17-logs-insights.png)

   Try a second one that computes statistics (challenge B does more of this):

   ```
   filter @type = "REPORT"
   | stats count() as invocations, avg(@duration) as avg_ms, max(@maxMemoryUsed)/1024/1024 as max_mb
   ```

**🧩 Investigation checkpoint**

- [ ] I can navigate from a function to its log group to a log stream in under a minute.
- [ ] I can explain why two test invocations landed in the **same** log stream (same execution environment).

---

## 🧠 Phase 4 — Call a foundation model from code, and hit the wall (15 minutes)

**Concept:** Code running under an execution role can call any AWS API **that the role allows**. Your role allows logging and nothing else. Watch what happens when the code asks Bedrock for an answer, read the error the way IAM wrote it, then grant exactly one permission.

1. Back on the **Code** tab, replace the whole file with:

   ```python
   # ORBIT Field Academy - Mission 02 - Step 2
   # Runs ONLY inside AWS Lambda. Uses the boto3 that Lambda already bundles.
   import json
   import os

   import boto3

   MODEL_ID = "amazon.nova-lite-v1:0"          # cheapest text model in US West (Oregon)
   bedrock = boto3.client("bedrock-runtime")   # credentials come from the execution role


   def lambda_handler(event, context):
       recruit = (event or {}).get("recruit", "unknown recruit")
       question = (event or {}).get("question", "In one sentence, what does a Lambda execution role do?")
       print(f"ORBIT telemetry: {recruit} asked: {question}")

       response = bedrock.converse(
           modelId=MODEL_ID,
           system=[{"text": "You are Ground Control for ORBIT Field Academy. Answer in at most two sentences."}],
           messages=[{"role": "user", "content": [{"text": question}]}],
           inferenceConfig={"maxTokens": 120, "temperature": 0.2},
       )

       answer = response["output"]["message"]["content"][0]["text"]
       usage = response["usage"]
       print(f"ORBIT telemetry: tokens in={usage['inputTokens']} out={usage['outputTokens']}")

       return {
           "statusCode": 200,
           "mission": "02-IGNITION",
           "recruit": recruit,
           "model": MODEL_ID,
           "answer": answer,
           "input_tokens": usage["inputTokens"],
           "output_tokens": usage["outputTokens"],
           "region": os.environ.get("AWS_REGION"),
       }
   ```

   **Deploy** (⇧⌘U / Ctrl+Shift+U).

   ![The Bedrock-calling code in the editor](images/m02-10-lambda-bedrock-code.png)

2. **Raise the timeout.** Open **Configuration → General configuration → Edit**. Set **Timeout** to `0` min `30` sec and choose **Save**. (A model call usually takes under a second, but the 3-second default leaves no margin for a slow cold start.)

   ![General configuration editor with memory, ephemeral storage, timeout and execution role](images/m02-11-lambda-general-config-edit.png)

3. Open the **Test** tab and choose **Test** with your saved `ignition-check` event. **It fails.** Expand **Details** and read the **Response**:

   ```
   An error occurred (AccessDeniedException) when calling the Converse operation:
   User: arn:aws:sts::962804699607:assumed-role/orbit-MY_ID-lambda-role/orbit-MY_ID-hello
   is not authorized to perform: bedrock:InvokeModel
   on resource: arn:aws:bedrock:us-west-2::foundation-model/amazon.nova-lite-v1:0
   because no identity-based policy allows the bedrock:InvokeModel action
   ```

   ![The failed test with the AccessDeniedException](images/m02-12-lambda-bedrock-denied.png)

   Read it as four facts. **Who:** the *assumed role* session of your Lambda role (not you). **What:** `bedrock:InvokeModel`. **On which resource:** the Nova Lite model ARN. **Why:** *no identity-based policy allows* it, an **implicit deny** in Mission 01's vocabulary. The fix is therefore an Allow on the role, for that action, on that resource.

4. **Grant exactly that.** Open **IAM → Roles → `orbit-MY_ID-lambda-role` → Add permissions → Create inline policy**. Switch the policy editor to **JSON**, replace the contents with:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "InvokeNovaLiteOnly",
         "Effect": "Allow",
         "Action": "bedrock:InvokeModel",
         "Resource": "arn:aws:bedrock:us-west-2::foundation-model/amazon.nova-lite-v1:0"
       }
     ]
   }
   ```

   ![The inline policy JSON in the IAM policy editor](images/m02-13-iam-inline-policy-json.png)

   Choose **Next**. **Policy name:** `orbit-MY_ID-bedrock-invoke`. The review table shows *Bedrock · Limited: Read · region string like us-west-2*. Choose **Create policy**.

   ![Review and create with the policy name filled in](images/m02-14-iam-inline-policy-review.png)

   ![The role now lists two policies: the managed logging policy and your inline Bedrock policy](images/m02-15-iam-role-with-policy.png)

5. Back in Lambda, **Test** again. **Executing function: succeeded.** The response carries the model's answer and the token counts; in the verified run Nova Lite answered *"A Lambda execution role grants AWS Lambda permissions to access AWS resources."* using 30 input and 16 output tokens, and the whole invocation took about 350 ms.

   ![The successful test with the model's answer and token counts](images/m02-16-lambda-bedrock-success.png)

   Record the **Request ID** in your ledger.

**Mini-debrief:** You never gave *yourself* permission to call Bedrock, and you never gave the role `"Resource": "*"`. One action, one model ARN. Change `MODEL_ID` to `amazon.nova-pro-v1:0` and the same error comes back, which is exactly what least privilege is supposed to do. IAM changes propagate in seconds; if the second test still fails, wait ten seconds and test again.

**🧩 Investigation checkpoint**

- [ ] I can recite the four facts in an AccessDenied message.
- [ ] I can say why the policy went on the **role** and not on my user.
- [ ] I know what would have to change in the policy to use a model through a `us.` inference profile (see the concepts section).

---

## 🧭 Phase 5 — Scouting Amazon Bedrock (5 minutes)

1. Search for `Bedrock` and open **Amazon Bedrock**. In the left navigation choose **Model catalog**. Filter by provider **Amazon** and find **Nova Lite**. Each card shows the modalities (text, image, video in), context length and whether the model is available in this Region.

   ![Bedrock model catalog filtered to Amazon models](images/m02-18-bedrock-model-catalog.png)

2. Open **Nova Lite**. The detail page shows the **Model ID** you used in code (`amazon.nova-lite-v1:0`), the input modalities (text, image, video), the 300K-token context window, **Deployment type: Serverless** (on-demand in this Region), a pricing link, and under *Usage* a code example for the very Converse call your function makes. Models that are only reachable through a cross-Region profile show the `us.` profile ID here instead.

   ![Nova Lite model detail page](images/m02-19-bedrock-nova-lite.png)

3. Choose **Model access** in the left navigation. In this account the Amazon and Anthropic models the course uses are **Available**. If a model is not, this is where an administrator requests access; no code or policy can bypass it.

   ![Model access page](images/m02-20-bedrock-model-access.png)

**🧩 Investigation checkpoint**

- [ ] I can find a model's ID and ARN in the catalog without guessing.
- [ ] I can explain the two gates a model call passes: **model access** (account level) and **IAM** (role level).

---

## 🤖 Phase 6 — Scouting AgentCore and its execution roles (10 minutes)

**Concept:** Amazon Bedrock AgentCore is where the ORBIT agent will live from Mission 04 on. Today you learn the map and the one security idea that carries over from Lambda: **an agent runs as a role**.

1. Search for `AgentCore` and open **Amazon Bedrock AgentCore**. The **Overview** page has four cards (build, observe, evaluate, explore). The left navigation is the map for the rest of the course:

   | Menu | What lives there | Mission |
   |---|---|---|
   | **Agents** | Your agents (a *harness* is an agent defined by configuration, running on AgentCore *Runtime*) with sessions, errors, latency and token cost | 04, 05, 06 |
   | **Playground** | Chat with an agent and see every tool call | 04, 06 |
   | **Observability** | Sessions, traces and spans across agents | 04, 06 |
   | **Connections** | Knowledge Bases (Mission 03), **Outbound Auth** (credentials an agent uses to call other systems), Browser, Code Interpreter | 03, 05 |
   | **Resources** | Memory, **Evaluations**, **Gateways** (tools exposed through MCP), **Policy** (rules for tool calls) | 04, 06 |

   ![AgentCore Overview with the left navigation](images/m02-21-agentcore-overview.png)

2. Open **Agents**. Each card shows one agent's usage, errors, P90 latency and token cost. In the shared account you will see classmates' agents later; filter by your `MY_ID`. (The screenshot is from a later verification run; yours will be empty today.)

   ![Agents page with one agent card](images/m02-22-agentcore-agents.png)

3. Open **Connections → Outbound Auth**. Its *How it works* panel explains the two directions of identity you will meet: **inbound** (who may call the agent: IAM, or a JWT from a pool like the Cognito one you inspect next) and **outbound** (how the agent proves itself to other APIs).
4. Open **Observability**. Agent sessions, traces and spans land here, built on the same CloudWatch you used in Phase 3.

   ![AgentCore Observability page](images/m02-24-agentcore-observability.png)

5. **The execution-role comparison.** Open IAM in a second tab and put the two trust policies side by side:

   | | Your Lambda role (Mission 01, Phase 5) | The mystery role (Mission 01, Phase 4) = AgentCore pattern |
   |---|---|---|
   | Trusted service principal | `lambda.amazonaws.com` | `bedrock-agentcore.amazonaws.com` |
   | Conditions | none | `aws:SourceAccount` = this account, `aws:SourceArn` like `arn:aws:bedrock-agentcore:us-west-2:ACCOUNT:*` |
   | Typical permissions | `logs:*` (basic execution) + what the code needs (`bedrock:InvokeModel` on one model) | `bedrock:InvokeModel*` on allowed models and inference profiles, `bedrock-agentcore:InvokeGateway` on its gateways, `logs:*` on `/aws/bedrock-agentcore/runtimes/*`, read-only tool access, explicit denies on writes |

   The two roles are the same idea with different principals. In Mission 04 the console creates an AgentCore role for you; now you will be able to read it.

**🧩 Investigation checkpoint**

- [ ] I can name the AgentCore menu that would host the Lambda function as a **tool** (Resources → Gateways) and the one that holds the **agent** (Agents).
- [ ] I can explain why an agent's role should not be the same role as its tools' roles.

---

## 🔐 Phase 7 — Watch a person get a token from the shared Cognito pool (15 minutes)

**Concept:** AgentCore and other APIs do not know your users; they trust **tokens** issued by an identity provider. The instructor's pool `orbit-academy-pool` is such a provider. You will read how it is configured, confirm that its public endpoints really are public, and sign in once as a recruit to see the first half of the most common sign-in flow on the web. **You only read and sign in. You create and change nothing.**

Here is the whole flow before you start, so nothing in step 6 surprises you:

```mermaid
sequenceDiagram
    autonumber
    participant B as 🧑 You (browser)
    participant C as 🪪 Cognito hosted login<br/>orbit-academy-….auth.us-west-2.amazoncognito.com
    participant A as 💻 The app's callback<br/>https://localhost:8080/callback
    B->>C: View login page (client ID + allowed callback in the URL)
    C->>B: Username → password → set a new password
    C-->>B: Redirect to the callback with ?code=…
    B->>A: Browser follows the redirect
    Note over A: Nothing is listening on your laptop,<br/>so the browser shows "This site can't be reached".<br/>That is expected: the code is in the address bar.
    A-->>C: (a real app would now swap the code for tokens: Challenge C)
```

### Step A: Read the pool (read-only)

1. Search for `Cognito` and open it. **User pools** lists every pool in the account, including ones classmates create in Mission 05. Open **`orbit-academy-pool`** and nothing else.

   ![Cognito user pools list](images/m02-25-cognito-pools.png)

2. The pool **Overview** shows the **User pool ID** (`us-west-2_…`), the ARN, the **estimated number of users** (31: thirty recruits and one for the instructor), the feature plan (*Essentials*), and two URLs you will use in a moment: the **OpenID Connect configuration URL** and the **Token signing key URL**.

   ![Pool overview](images/m02-26-cognito-pool-overview.png)

3. **User management → Users**: thirty recruits, each with **Confirmation status** *Force change password* until its first sign-in. Find **your** `recruitNN` (NN = your participant number). **Use only that one.** A recruit's password changes the first time someone signs in with it, so a classmate who borrows your number locks you out.

   ![Users list with the recruit accounts](images/m02-27-cognito-users.png)

4. **Applications → App clients → `orbit-academy-web`**. Read **App client information**:
   - **Client ID**: like a username for the *application*. Public by design.
   - **Client secret: -**: none. This is a **public client**: it is meant for code that runs in a browser or on a phone, where a secret could not be kept anyway.
   - **Authentication flows** (*Choice-based sign-in*, *SRP*) and token lifetimes (access and ID tokens 60 minutes, refresh token 1 day).

   ![App clients list](images/m02-28-cognito-app-clients.png)

   ![App client information for orbit-academy-web](images/m02-28b-cognito-app-client-detail.png)

5. Open the **Login pages** tab. It answers the three questions every sign-in page needs answered:

   | Setting | Value | Why it matters |
   |---|---|---|
   | **Allowed callback URLs** | `https://localhost:8080/callback` | The *only* place Cognito will send a user after sign-in. An attacker cannot redirect the result to their own site. |
   | **OAuth grant types** | Authorization code grant | The browser receives a short-lived **code**, not the tokens themselves. |
   | **Scopes** | `openid`, `email`, `profile`, `orbit-academy-api/read` | What the app is asking permission for. |

   ![Login pages tab with the View login page button](images/m02-28c-cognito-view-login-page.png)

### Step B: Prove the public parts are public

6. In a new browser tab open the pool's **discovery document** (replace `POOL_ID` with the **User pool ID** from the Overview):

   ```
   https://cognito-idp.us-west-2.amazonaws.com/POOL_ID/.well-known/openid-configuration
   ```

   You get JSON with `issuer`, `authorization_endpoint`, `token_endpoint`, `jwks_uri` and the supported scopes, with no sign-in. Open the `jwks_uri` too: two RSA **public** keys. Any service can verify a token offline with them; the private keys never leave Cognito. In Mission 05 you paste a URL like this one into AgentCore as the *discovery URL* of your own pool.

### Step C: Sign in as your recruit

7. Back on the **Login pages** tab, choose **View login page**. A new tab opens Cognito's hosted sign-in page, already carrying this app client's ID and allowed callback in its address. (No URL to copy from the whiteboard: the console builds the correct one for you.)
8. Enter **Username** `recruitNN` → **Next** → the **temporary password** from the instructor → **Sign in**.

   ![Hosted login page: username step](images/m02-30-cognito-hosted-login.png)

   ![Password step](images/m02-31-cognito-password-step.png)

9. Cognito asks for a **new password** because the account is in *Force change password* state. Choose one you will remember (10+ characters with upper case, lower case, a number and a symbol), type it twice, and submit. Do not reuse a real password.

   ![The forced new-password step](images/m02-32-cognito-new-password.png)

10. The browser jumps to `https://localhost:8080/callback?code=…` and shows **This site can't be reached**. **That is the success screen of this exercise.** Cognito did its job: it verified you and sent your browser to the app's registered callback with a one-time **authorization code**. There is simply no app running on your laptop to receive it. Look at the address bar: the `code` value is valid for **five minutes** and can be used **once**. Copy it now if you want to try Challenge C.

    ![The redirect to the callback URL with the authorization code in the address bar](images/m02-33-cognito-callback.png)

> [!TIP]
> **If sign-in fails**
> - *Incorrect username or password* on your **first** sign-in: check the recruit number (it is your participant number) and retype the temporary password. If it still fails, someone may already have used your recruit; tell the instructor rather than trying another number.
> - *Temporary password has expired*: temporary passwords last 30 days from when the instructor created the users. The instructor resets it from the pool (Users → your recruit → Reset password).
> - *An error was encountered with the requested page*: you edited the login URL. Close the tab and use **View login page** again.

**🧩 Investigation checkpoint**

- [ ] I can say which Cognito objects are **public** (issuer, discovery document, JWKS, login page, client ID) and which are **secret** (passwords, private signing keys, a confidential client's secret).
- [ ] I can explain why seeing *This site can't be reached* meant the sign-in **worked**.
- [ ] I can explain why an app client with no secret still cannot be abused to sign in as someone else (the user's password is still required, and the code can only go to the allowed callback).

---

## 🔎 Phase 8 — Inspect the OpenSearch domain (5 minutes)

**Concept:** Managed knowledge bases (Mission 03) hide the vector store. An OpenSearch domain is that store with the lid off: you can see the nodes, the index and the vector field.

> [!NOTE]
> **This phase depends on a shared domain the instructor may not have created.** A small domain costs about $0.90 per day while it exists, so some classes skip it. Open **Amazon OpenSearch Service → Domains**: if `orbit-academy-search` is not listed, read this phase as a guided tour (the facts below describe exactly what you would see), skip Challenge D, and move on.

1. Search for `OpenSearch` and open **Amazon OpenSearch Service → Domains**. Open **`orbit-academy-search`**.

   ![OpenSearch domains list](images/m02-29-opensearch-domains.png)

2. On the domain dashboard read **General information**: the **Domain endpoint** (`https://search-orbit-academy-search-….us-west-2.es.amazonaws.com`), the **engine version** (OpenSearch 2.19), **Deployment type** (dev/test, single AZ), the **data node** type (`t3.small.search`, 1 node) and storage (10 GiB gp3). Note the domain ARN.
3. Check **Cluster health**: with a single node the *cluster status* is **green** only when every index has zero replicas; a yellow status means replica shards have nowhere to go. The sample index is configured with zero replicas, so expect green.
4. Open **Security configuration**. The **access policy** is a resource-based policy: it allows the instructor identity everything (`es:*`) and the account's other identities only `es:ESHttpGet` and `es:ESHttpHead` on the domain. Fine-grained access control is off for this teaching domain. Compare the shape of this document with the trust policy of a role: a **Principal** element, because the policy sits on the resource and must say *who*.
5. The domain holds one index, `orbit-policies`, with four ORBIT policy documents and an 8-dimensional `knn_vector` field called `embedding` (real embeddings have 256–1,536 dimensions). Challenge D shows you the mapping from CloudShell.

**🧩 Investigation checkpoint**

- [ ] I can explain why a resource-based policy needs a `Principal` and an identity-based policy does not.
- [ ] I can describe, in one sentence each, *domain*, *index*, *document* and *vector field*.

---

## 🏆 Phase 9 — Challenge exercises (optional)

### Challenge A · Alarm bell (CloudWatch)

Create an alarm that fires when your function errors. **CloudWatch → Alarms → Create alarm → Select metric → Lambda → By Function Name → `orbit-MY_ID-hello` · Errors** → statistic *Sum*, period *1 minute*, threshold *Greater or equal to 1*, name `orbit-MY_ID-hello-errors`, no notification. Then break the function on purpose: change `MODEL_ID` to `"amazon.nova-pro-v1:0"`, deploy, and test (AccessDenied, because your policy names only Nova Lite). Within one to three minutes the alarm state moves *Insufficient data → In alarm*. Fix the model ID, deploy, and watch it return to *OK* after the next successful test. Delete the alarm when done.

### Challenge B · Telemetry analyst (Logs Insights)

Write one Logs Insights query over your log group that returns: number of invocations, average and maximum duration, maximum memory used in MB, and the number of cold starts (hint: `@initDuration` exists only on cold-start `REPORT` lines; use `count(@initDuration)`). Then answer: at 128 MB, how many invocations of your Bedrock function would one dollar buy at $0.0000167 per GB-second plus $0.20 per million requests?

### Challenge C · Token autopsy (Cognito + CloudShell)

Within five minutes of signing in (Phase 7, step 10), exchange the authorization code for tokens. The **domain** is on the app client's **Login pages** tab (or in the address bar of the hosted login page); the **client ID** is on the app client page. In CloudShell, replace the values and run:

```bash
DOMAIN=orbit-academy-XXXXX.auth.us-west-2.amazoncognito.com   # from App client → Login pages
CLIENT_ID=XXXXXXXXXXXXXXXXXXXXXXXXXX                            # from App client
CODE=PASTE_THE_CODE_FROM_THE_ADDRESS_BAR
curl -s -X POST "https://$DOMAIN/oauth2/token" \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d "grant_type=authorization_code&client_id=$CLIENT_ID&code=$CODE&redirect_uri=https%3A%2F%2Flocalhost%3A8080%2Fcallback" > /tmp/tokens.json
python3 - <<'PY'
import json, base64
t = json.load(open('/tmp/tokens.json'))
for name in ('id_token', 'access_token'):
    payload = t[name].split('.')[1]
    payload += '=' * (-len(payload) % 4)
    print(f"--- {name} ---")
    print(json.dumps(json.loads(base64.urlsafe_b64decode(payload)), indent=2))
PY
```

Questions: which claim says who issued the token (`iss`) and does it match the discovery document? Which token carries your `cognito:username`, and which carries `scope`? What is `exp` minus `iat`? Why would AgentCore reject this exact token if its *allowed clients* list named a different client ID? (If the code has expired you get `invalid_grant`; sign in again for a fresh one.)

### Challenge D · Vector safari (OpenSearch + CloudShell)

Every request to the domain must be **signed**, exactly like the STS call in Mission 01. In CloudShell:

```bash
ENDPOINT=https://search-orbit-academy-search-XXXX.us-west-2.es.amazonaws.com   # from General information
eval $(aws configure export-credentials --format env)
sig() { curl -s --aws-sigv4 aws:amz:us-west-2:es --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" -H "x-amz-security-token: $AWS_SESSION_TOKEN" "$@"; }
sig "$ENDPOINT/_cat/indices?v"
sig "$ENDPOINT/orbit-policies/_mapping" | python3 -m json.tool
sig "$ENDPOINT/orbit-policies/_search?q=tent&pretty"
```

Find the `knn_vector` field and its `dimension`. Then try an unsigned `curl` on the same URL and a signed `PUT` (`-X PUT "$ENDPOINT/orbit-policies/_doc/test" -d '{}'`): which error do you get for each, and which policy statement decides it?

### Challenge E · Change the model (Bedrock + IAM)

Switch `MODEL_ID` to the Nova Micro inference profile `us.amazon.nova-micro-v1:0`, deploy, test, and read the new AccessDenied. Extend your inline policy so the call works **without** using `*` as the resource. Hint: a cross-Region profile needs the profile ARN (`arn:aws:bedrock:us-west-2:ACCOUNT:inference-profile/us.amazon.nova-micro-v1:0`) *and* the foundation-model ARN in each Region it routes to (`arn:aws:bedrock:*::foundation-model/amazon.nova-micro-v1:0`). Set it back to Nova Lite when you are done.

### Challenge F · Configuration, not code

Move `MODEL_ID` into an **environment variable** (Configuration → Environment variables) and read it with `os.environ["MODEL_ID"]`. Why is this the right place for a model ID and the wrong place for a client secret?

---

## 🔭 Production lens: the two numbers that matter most

You met two facts today that run every production agent's budget and on-call rota.

- **Tokens are the meter.** Your Lambda reported `inputTokens` and `outputTokens`. An agent re-sends its instructions and tool definitions on every model call, so input tokens dominate the bill. In Mission 06 you will cut an agent's input tokens by about 30% by removing two tools it never needed.
- **Logs only help if you planned them.** Your `ORBIT telemetry:` lines were useful because you printed them on purpose. In Mission 06 the tool gate prints one JSON object per decision, and a single Logs Insights query turns those into an audit report. Decide what you will need to know at 2 a.m. *before* you write the code.

---

## 🏁 Mission completion · Earn the Ignition Badge

One line each, in your own words:

1. **Execution role** — who assumes it and when.
2. **REPORT line** — three numbers it gives you.
3. **AccessDenied** — the four facts in the message.
4. **Model access vs. IAM** — the two gates.
5. **Token** — what a user pool publishes so others can verify one.

- [ ] My function ran, logged, and answered a question through Nova Lite with a one-model policy.
- [ ] I found its log group, read a `REPORT` line, and ran a Logs Insights query.
- [ ] I toured Bedrock and AgentCore and can describe an AgentCore execution role.
- [ ] I inspected the shared Cognito pool and OpenSearch domain and know what is public, what is secret, and what is read-only.

---

## 🧯 Shared-account guardrails, costs, and cleanup

**Cleanup at the end of the course** (your own resources only):

1. Lambda → Event Functions → `orbit-MY_ID-hello` → **Actions → Delete**.
2. CloudWatch → Alarms → `orbit-MY_ID-hello-errors` → Delete (if you did Challenge A).
3. CloudWatch → Log management → `/aws/lambda/orbit-MY_ID-hello` → Delete (optional; it costs a fraction of a cent per month).
4. IAM → Roles → `orbit-MY_ID-lambda-role` → Delete (last; Mission 04 creates its own roles).

Never touch `orbit-academy-pool`, `orbit-academy-search`, or `orbit-academy-mystery-role`.

**What this mission costs** (US West (Oregon) list prices, October 2026):

| Item | Price | Typical use | Cost per participant |
|---|---|---|---|
| Lambda requests and duration | $0.20 per 1M requests; $0.0000167 per GB-second | about 8 invocations, 128 MB, < 1 s each | < $0.001 |
| Nova Lite via the Converse API | $0.06 per 1M input tokens, $0.24 per 1M output tokens | about 3 calls × 50 tokens | < $0.001 |
| CloudWatch Logs ingestion | $0.50 per GB | a few KB | < $0.001 |
| CloudWatch alarm (Challenge A) | $0.10 per alarm-month after 10 free | 1, deleted | $0 |
| IAM, Bedrock catalog, AgentCore console, Cognito (31 users), OpenSearch *inspection* | free | | $0 |
| **Total** | | | **effectively $0** |

The instructor's OpenSearch domain (`t3.small.search`) costs about **$0.90 per day** while it exists and is deleted after the course.

**Quotas that matter with 20+ people at once:** Lambda allows 1,000 concurrent executions (you use one at a time); the Bedrock on-demand throughput for Nova Lite is shared, so a `ThrottlingException` means wait a few seconds and test again; CloudShell allows **10 concurrent sessions per account per Region**, so Challenges C and D run in waves.

### If something fails

| Symptom | Check |
|---|---|
| Create function fails with a role error | You chose a role without your `MY_ID`, or the role's trust policy is not Lambda; recreate it as in Mission 01 Phase 5 |
| `Runtime.UserCodeSyntaxError` on test | Indentation changed on paste; select all in the editor, delete, paste again from the guide, Deploy |
| Test shows *Task timed out after 3.00 seconds* | You skipped Phase 4 step 2; set the timeout to 30 s |
| AccessDenied persists after the policy | Policy is on the **role** (not the user), resource ARN is exactly the Nova Lite ARN, Region in the ARN is `us-west-2`; wait 10 s and retry |
| `ValidationException … model identifier is invalid` | Typo in `MODEL_ID`, or you used a profile ID without the `us.` prefix |
| Logs Insights returns nothing | Log group selected, time range *1h*, you invoked the function at least once |
| Hosted login says *An error was encountered with the requested page* | Open the page with **View login page** on the app client's **Login pages** tab instead of typing a URL |
| *Temporary password has expired* | Ask the instructor to reset your recruit's password |
| Cognito *Access denied* on Edit | Correct: students are read-only on the shared pool |
| OpenSearch `sig` returns `403 … explicit deny` on PUT | Correct: the access policy allows the account only GET and HEAD |

**AWS docs to consult:** [Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html) · [Lambda console code editor](https://docs.aws.amazon.com/lambda/latest/dg/code-editor.html) · [Lambda REPORT log line](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html) · [Logs Insights query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html) · [Bedrock Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html) · [Bedrock inference profiles and IAM](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-prereq.html) · [AgentCore execution roles](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-permissions.html) · [Cognito user pool OIDC endpoints](https://docs.aws.amazon.com/cognito/latest/developerguide/userinfo-endpoint.html) · [OpenSearch Service access policies](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ac.html).

*End of Mission 02 · Next: Mission 03, THE KNOWLEDGE HEIST — the Bedrock Playground, a real knowledge base, and the line between evidence and action.*
