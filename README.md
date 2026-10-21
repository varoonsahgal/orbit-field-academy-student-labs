# 🛰️ ORBIT FIELD ACADEMY — Student Labs

Hands-on, **AWS Management Console-only** missions for the *AWS Agentic AI Microcredential Bootcamp*
(Amazon Bedrock AgentCore). You play a recruit at **ORBIT Outfitters**, a fictional outdoor retailer that
is moving its support assistant onto Amazon Bedrock AgentCore. Every mission runs in a shared training
account in **US West (Oregon)**, and every resource you create carries your personal `MY_ID`.

> **Verified against the live AWS Console in October 2026.** The screenshots in `images/` are from the
> verification run (example participant `s07x42`). If a button has moved, look for the same words nearby.

## Mission sequence

Run the missions in order; each one builds on the last.

| # | Mission | Course module | What you build or inspect | Time |
|---|---|---|---|---|
| 01 | [GROUND CONTROL](01-ORBIT-Ground-Control.md) | 1 · AWS Foundations | Regions and AZs, console navigation, IAM policies and the mystery-role puzzle, your first IAM role, CloudShell and signed requests | 60–75 min |
| 02 | [IGNITION](02-ORBIT-Ignition.md) | 1 · AWS Foundations | A Lambda function, its CloudWatch logs and metrics, a Bedrock model call that fails until IAM says yes, the AgentCore console tour, a Cognito user pool and an OpenSearch domain | 60–75 min |
| 03 | [THE KNOWLEDGE HEIST](03-ORBIT-The-Knowledge-Heist.md) | 2 · Agentic AI and Bedrock Fundamentals | Bedrock Playground, a managed knowledge base with inline documents, retrieval vs. generation, Agents Classic migration map | 75–90 min |
| 04 | [LAUNCH CONTROL](04-ORBIT-Launch-Control.md) | 3 · AgentCore Architecture · 4 · Migration · 6 · Tools and Gateway | An AgentCore Harness, a Lambda tool through AgentCore Gateway, the knowledge base as a gateway target, the four-principals permission fix, observability | 120–150 min |
| 05 | [IDENTITY LOCKDOWN](05-ORBIT-Identity-Lockdown.md) | 5 · Authentication with Cognito | A Cognito M2M app identity, a JWT-protected Harness, Secrets Manager, a Lambda test client proving accepted and rejected calls | 100–130 min |

Each mission ends with **challenge exercises** for people who finish early and a **cleanup** list.

## Before you start

1. **Credentials.** Your instructor gives you an IAM user name and password for the training account (`962804699607`, *Training*).
2. **Region.** Everything is in **US West (Oregon)** `us-west-2`. Check the Region selector every time you open a new service.
3. **Your `MY_ID`.** Seat number plus two random digits, lowercase, for example `s07x42`. Use the same one in every mission; it is what keeps your resources apart from 29 classmates'.
4. **Shared-account rules.** Never edit or delete anything that does not contain your `MY_ID`. Resources that start with `orbit-academy-` are shared and read-only.
5. **Browser.** A current Chrome, Edge or Firefox. No local installs, no CLI on your laptop; Mission 01 introduces AWS CloudShell for the terminal parts.

## Keep a ledger

Each mission has a *resource ledger* table. Fill it in as you go: IDs, ARNs and names flow from one mission to the next (your Lambda role from Mission 01 is used in Mission 02; your knowledge base from Mission 03 is attached to the agent in Mission 04).

## Costs

The whole five-mission sequence costs roughly **$0.25 per participant** at October 2026 list prices (Missions 01 and 02 are essentially free; the model calls in 03–05 are cents). Nothing you create keeps billing meaningfully while idle except a Secrets Manager secret ($0.40 per month), which Mission 05's cleanup removes.
