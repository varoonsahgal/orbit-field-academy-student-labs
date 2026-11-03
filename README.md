# 🛰️ ORBIT FIELD ACADEMY — Student Labs

Hands-on, **AWS Management Console-only** missions for the *AWS Agentic AI Microcredential Bootcamp*
(Amazon Bedrock AgentCore). You play a recruit at **ORBIT Outfitters**, a fictional outdoor retailer that
is moving its support assistant onto Amazon Bedrock AgentCore. Every mission runs in a shared training
account in **US West (Oregon)**, and every resource you create carries your personal `MY_ID`.

**Read the missions on the course website:** <https://varoonsahgal.github.io/orbit-field-guide/>. Enter your
`MY_ID` once there and every command, policy and name on every page is personalised for you.

> **Verified against the live AWS Console between October 20 and November 3, 2026.** The screenshots in
> `images/` are from the verification runs (example participants `s07x42` and `v06x11`). AWS reorganised the
> AgentCore console in late October 2026; the guides use the new menu names. If a button has moved, look for
> the same words nearby.

## Mission sequence

Run the missions in order; each one builds on the last. Missions 01–02 are optional foundations for people new to AWS; the core path is 03–08.

| # | Mission | Course module | What you build or inspect | Time |
|---|---|---|---|---|
| 01 | [GROUND CONTROL](01-ORBIT-Ground-Control.md) | 1 · AWS Foundations | Regions and AZs, console navigation, IAM policies and the mystery-role puzzle, your first IAM role, CloudShell and signed requests | 60–75 min |
| 02 | [IGNITION](02-ORBIT-Ignition.md) | 1 · AWS Foundations | A Lambda function, its CloudWatch logs and metrics, a Bedrock model call that fails until IAM says yes, the AgentCore console tour, a read-only look at a Cognito sign-in, and an OpenSearch domain | 60–75 min |
| 03 | [THE KNOWLEDGE HEIST](03-ORBIT-The-Knowledge-Heist.md) | 2 · Agentic AI and Bedrock Fundamentals | Bedrock Playground, a managed knowledge base with inline documents, retrieval vs. generation, Agents Classic migration map | 75–90 min |
| 04 | [LAUNCH CONTROL](04-ORBIT-Launch-Control.md) | 3 · AgentCore Architecture · 4 · Migration · 6 · Tools and Gateway | An AgentCore Harness, a Lambda tool through AgentCore Gateway, the knowledge base as a gateway target, the four-principals permission fix, observability | 120–150 min |
| 05 | [IDENTITY LOCKDOWN](05-ORBIT-Identity-Lockdown.md) | 5 · Authentication with Cognito | Your own Cognito M2M app identity, a JWT-protected Harness, Secrets Manager, a Lambda test client proving accepted and rejected calls | 110–140 min |
| 06 | [MISSION ASSURANCE](06-ORBIT-Mission-Assurance.md) | Production readiness | A refund tool, then five controls around it (allowed tools and limits, a Bedrock Guardrail made mandatory with IAM, Cedar policy on the gateway in log-only then enforce mode, a lifecycle-hook action gate), observability with Logs Insights, and live evaluations with a custom LLM-as-a-judge | 150–180 min |
| 07 | [HULL BREACH](07-ORBIT-Hull-Breach.md) | Agent security | Build an insecure support copilot the hackathon way, then exploit and seal five breaches: a secret in the prompt, indirect prompt injection through a ticket, a poisoned third-party tool, shared and poisoned long-term memory, and an over-privileged role reachable through the shell | 165–195 min |
| 08 | [CONSTELLATION](08-ORBIT-Constellation.md) | Multi-agent systems | A crew of three agents: an analyst with a code interpreter, a scout with a browser, a commander that delegates through a gateway and waits for a human approval desk; trust boundaries, a reckless-commander drill, and the economics of a crew | 150–180 min |

Each mission ends with **challenge exercises** for people who finish early and a **cleanup** list.
Missions 03–05 have **field notes** (`companionguides/`): question-and-answer companions for the concepts behind each step.

## Before you start

1. **Credentials.** Your instructor gives you an IAM user name (`participant-NN`) and password for the training account (`962804699607`, *Training*).
2. **Region.** Everything is in **US West (Oregon)** `us-west-2`. Check the Region selector every time you open a new service.
3. **Your `MY_ID`.** The letter `p`, your two-digit participant number, the letter `x` and two digits you pick: `participant-07` → for example `p07x42`. Lowercase letters and digits only. Use the same one in every mission; it is what keeps your resources apart from your classmates'. Never type the placeholder itself into AWS.
4. **Shared-account rules.** Never edit or delete anything that does not contain your `MY_ID`. Resources that start with `orbit-academy-` are shared and read-only. Never change account-wide settings (Bedrock enforced guardrails, CloudWatch Transaction Search).
5. **Browser.** A current Chrome, Edge or Firefox. No local installs, no CLI on your laptop; Mission 01 introduces AWS CloudShell for the terminal parts.

## Keep a ledger

Each mission has a *resource ledger* table. Fill it in as you go: IDs, ARNs and names flow from one mission to the next (your Lambda role from Mission 01 is used in Mission 02; your knowledge base from Mission 03 is attached to the agent in Mission 04; Mission 06 upgrades the Mission 04 agent, gateway and Lambda; Missions 07 and 08 build their own agents and can be run after 06 in either order).

## Costs

The whole eight-mission sequence costs roughly **$1 per participant** at November 2026 list prices (Missions 01 and 02 are essentially free; the model calls, evaluations and policy checks in 03–06 are cents). Two things keep billing while idle, and both cleanups remove them: a Secrets Manager secret ($0.40 per month, Mission 05) and an *enabled* evaluation configuration (Mission 06), which scores every new session of your agent.
