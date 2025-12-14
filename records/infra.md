### 1) CloudFront Functions vs Lambda@Edge (why split them)

**CloudFront Functions** are designed for *very small, latency-critical* logic at the viewer edge: URL rewrites, redirects, header normalization, simple bot detection, A/B routing hints, etc. They run in a constrained JavaScript runtime and are optimized for sub-millisecond execution at very high scale. **Lambda@Edge** is for *heavier* edge compute: more triggers (including origin request/response), longer execution time, and runtimes suitable for SSR (Node.js), but with higher operational overhead and cost/latency. ([AWS Documentation][1])

In your design, a common pattern is:

* **CloudFront Function (viewer-request)**: detect crawler via User-Agent → rewrite URI to an SSR prefix (e.g., `/__ssr/<path>`) or set a routing header.
* CloudFront behavior routes `/__ssr/*` to the SSR path where **Lambda@Edge (Node.js SSR)** runs, while normal paths go to **S3**.
  This keeps *classification/routing* cheap and fast, while SSR stays in Lambda@Edge where it belongs. ([AWS Documentation][1])

---

### 2) Cognito: “frontend design still needed” + email/phone + Google login

Yes—Cognito provides the **identity backend**, but you still need the **frontend auth UX** in Svelte. You typically choose either:

* **Cognito Hosted UI** (fastest, less custom UI work), or
* **Custom UI** in Svelte using OAuth2/OIDC flows against Cognito.

Cognito **User Pools** support native sign-up/sign-in (email/phone verification, password recovery, optional MFA) and **federation with social IdPs (e.g., Google)**. ([AWS Documentation][2])

“Identity Pools” remain optional and are only needed if your browser/mobile client must obtain temporary AWS credentials to call AWS services directly.

---

### 3) “Send later” scheduling: EventBridge Scheduler vs “CloudWatch”

CloudWatch Events scheduling is effectively **EventBridge scheduled rules** today.
You have two viable options:

* **EventBridge Scheduler (recommended when schedules are per-job/per-user)**: supports **one-time schedules**, time zones, flexible delivery windows, and first-class retry/DLQ controls per schedule. ([AWS Documentation][3])
* **EventBridge Scheduled Rules (legacy cron/rate)**: perfectly fine for a **single periodic poller** (e.g., every minute scan DynamoDB for due jobs). It is simpler, but schedules run in **UTC** and feature-set is more limited. ([AWS Documentation][4])

“More effective” depends on your model:

* **Poller model** → scheduled rule is adequate.
* **True per-email ‘at time X’ model at scale** → Scheduler is usually better.

---

### 4) Open/click tracking: SES configuration sets vs your own endpoints vs Pinpoint

You *can* use **Amazon SES configuration sets** to enable **open and click tracking**, including configuring a **custom tracking domain** so links/pixels appear under your domain. ([AWS Documentation][5])
For broader engagement/deliverability metrics (including opens/clicks), AWS also points to **Virtual Deliverability Manager** as a monitoring approach. ([Repost][6])

If you need **marketing-style segmentation, campaigns/journeys, richer analytics and dashboards**, **Amazon Pinpoint** can be used alongside SES (often with event streaming pipelines). ([Amazon Web Services, Inc.][7])

---

## Updated table (features → AWS services)

| Feature / function                                    | AWS services needed                                                                       | Notes / options                                                                                                            |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Frontend (SvelteJS) static delivery                   | **S3** + **CloudFront**                                                                   | Svelte build → S3; CloudFront CDN + TLS                                                                                    |
| Bot vs user traffic separation                        | **CloudFront** + **CloudFront Function (viewer-request)** + **Lambda@Edge (Node.js SSR)** | CF Function: classify UA + rewrite to `/__ssr/*` (fast/cheap). SSR: Lambda@Edge only on SSR path. ([AWS Documentation][1]) |
| Normal users → SPA                                    | **CloudFront → S3 origin**                                                                | Cacheable static assets                                                                                                    |
| Search engine bots → SSR HTML                         | **Lambda@Edge (Node.js)**                                                                 | SSR output returned to crawler; typically attached to SSR behavior (origin-request/response) ([AWS Documentation][1])      |
| Custom domain + HTTPS                                 | **Route 53** + **ACM**                                                                    | ACM cert on CloudFront                                                                                                     |
| Authentication backend                                | **Cognito User Pools**                                                                    | Email/phone sign-up, verification, recovery, MFA; federated login incl. Google ([AWS Documentation][2])                    |
| Authentication frontend (still required)              | **Svelte auth UI** + (Cognito Hosted UI **or** custom OAuth/OIDC flow)                    | Hosted UI reduces frontend effort; custom UI gives full control                                                            |
| Public APIs (CRUD: contacts/templates/campaigns/jobs) | **API Gateway** → **Lambda**                                                              | HTTP API or REST API                                                                                                       |
| Contacts + segmentation metadata                      | **DynamoDB**                                                                              | GSIs for segment queries                                                                                                   |
| Templates (metadata + content)                        | **DynamoDB** (+ **S3** optional)                                                          | S3 if large HTML/assets; DynamoDB for metadata/vars                                                                        |
| Task store (scheduled/queued/sent/failed)             | **DynamoDB** *(or RDS)*                                                                   | DynamoDB preferred for serverless scheduling/state                                                                         |
| “Send later” scheduling (dispatch due jobs)           | **EventBridge Scheduler → Lambda (Scheduler/Dispatcher)**                                 | Best for per-job schedules; retries/DLQ supported ([AWS Documentation][3])                                                 |
| Alternative scheduling (polling scan)                 | **EventBridge Scheduled Rule (legacy “CloudWatch Events”) → Lambda**                      | Good for “run every minute” scans; UTC-based schedules ([AWS Documentation][4])                                            |
| Queue-driven sending                                  | **SQS (+ DLQ)**                                                                           | Decouple UI from send workload; retries + poison-message capture                                                           |
| Email sending worker                                  | **Lambda (EmailSender)** + **SES**                                                        | SQS-triggered Lambda sends via SES                                                                                         |
| Idempotency (avoid duplicates)                        | **DynamoDB**                                                                              | Conditional writes / state transition guards per `job_id`                                                                  |
| Delivery events (bounce/complaint/reject)             | **SES → SNS → Lambda**                                                                    | Update suppression flags + persist events                                                                                  |
| Open/click tracking (managed)                         | **SES Configuration Sets** (+ custom tracking domain optional)                            | SES can emit open/click events; custom domain supported ([AWS Documentation][5])                                           |
| Open/click tracking (self-hosted alternative)         | **CloudFront → API Gateway/Lambda** + **DynamoDB/S3**                                     | Pixel + redirect endpoints you control (if you don’t use SES tracking)                                                     |
| Advanced engagement/marketing analytics (optional)    | **Pinpoint** (alongside SES)                                                              | Segmentation/journeys; can stream engagement events ([Amazon Web Services, Inc.][7])                                       |
| Payments (Stripe webhooks)                            | **API Gateway → Lambda** + **DynamoDB**                                                   | Store customer/subscription state                                                                                          |
| Secrets                                               | **Secrets Manager** *(or SSM Parameter Store)*                                            | Stripe keys, webhook signing secret                                                                                        |
| Observability                                         | **CloudWatch Logs/Metrics/Alarms**                                                        | Lambda errors, SQS depth, DLQ count, SES metrics                                                                           |
| Security / abuse protection                           | **IAM** + **WAF (CloudFront)** + **KMS** + **CloudTrail**                                 | Least privilege, rate limiting, encryption, audit                                                                          |

[1]: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html?utm_source=chatgpt.com "Differences between CloudFront Functions and Lambda@ ..."
[2]: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-social-idp.html?utm_source=chatgpt.com "Using social identity providers with a user pool"
[3]: https://docs.aws.amazon.com/eventbridge/latest/userguide/using-eventbridge-scheduler.html?utm_source=chatgpt.com "Amazon EventBridge Scheduler"
[4]: https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html?utm_source=chatgpt.com "Creating a scheduled rule (legacy) in Amazon EventBridge"
[5]: https://docs.aws.amazon.com/ses/latest/dg/configure-custom-open-click-domains.html?utm_source=chatgpt.com "Configuring custom domains to handle open and click tracking"
[6]: https://repost.aws/knowledge-center/ses-email-opens-clicks?utm_source=chatgpt.com "Monitor opens, clicks, and bounces for Amazon SES emails"
[7]: https://s3.amazonaws.com/solutions-reference/digital-user-engagement-events-database/latest/digital-user-engagement-events-database.pdf?utm_source=chatgpt.com "Digital User Engagement Events Database - Amazon S3 - AWS"




-------------------




Below are **graph-ready flow descriptions** (edges) and a **Mermaid diagram** you can paste into most markdown diagram tools.

## Connection + flow descriptions (edges)

### A) Web delivery + bot SSR split

1. **User Browser / Search Bot → Route 53 → CloudFront (ACM TLS)**
2. **CloudFront → CloudFront Function (viewer-request)**: classify request (bot vs normal)
3. **Normal user path**: CloudFront → **S3 (Svelte static site)** → CloudFront → Browser
4. **Bot path (SSR)**: CloudFront → **Lambda@Edge (Node.js SSR)** → CloudFront → Bot
5. **Protection**: **WAF** attached to CloudFront (rate limits / bot rules)

### B) Authentication (Cognito) and API authorization

1. Browser loads Svelte app (A)
2. Svelte auth UI uses **Cognito User Pools**:

   * Either **Hosted UI redirect flow**, or **custom UI + OAuth2/OIDC**
   * Supports **email/phone registration** and **Google social login**
3. Browser receives Cognito tokens → calls APIs with `Authorization: Bearer <JWT>`
4. **API Gateway (JWT authorizer / integration) → Lambda (API handlers) → DynamoDB/S3 → API Gateway → Browser**

### C) Core data flows (CRUD)

* **Contacts / Segments**: API Gateway → Lambda → DynamoDB (GSIs for segment queries)
* **Templates**: API Gateway → Lambda → DynamoDB (metadata/vars) and optionally S3 (large HTML/assets)
* **Jobs / Campaigns**: API Gateway → Lambda → DynamoDB (task store / campaign state)

### D) Scheduling + dispatch (send later)

Two operational modes (pick one or mix):

1. **Per-job schedules**: **EventBridge Scheduler → Scheduler/Dispatcher Lambda → DynamoDB (query due jobs) → SQS**
2. **Periodic poller**: **EventBridge Scheduled Rule (“CloudWatch Events”) → Scheduler/Dispatcher Lambda → DynamoDB → SQS**

### E) Queue-driven sending + reliability

1. **SQS → EmailSender Lambda (batch)** → **SES (SendEmail/SendRawEmail)**
2. EmailSender updates **DynamoDB job status** (sent/failed, timestamps, message ids)
3. Failures: SQS retries; after max receives → **SQS DLQ**
4. **Idempotency** enforced by DynamoDB conditional writes / status transitions keyed by `job_id`

### F) Delivery events + suppression

1. **SES → SNS → EventHandler Lambda**
2. EventHandler Lambda updates:

   * **DynamoDB**: bounce/complaint records, suppression flags, per-campaign counters
   * (Optional) **S3**: append-only archive of events

### G) Engagement tracking (opens/clicks) – two options

**Option 1 (Managed via SES Configuration Sets):**

* Emails sent with **SES Configuration Set** that enables **open/click tracking**
* Events can be routed to destinations (commonly **SNS / CloudWatch / streams**) for persistence/analytics

**Option 2 (Self-hosted tracking endpoints):**

* Email HTML includes:

  * **Open pixel URL** → **CloudFront → API Gateway/Lambda** → DynamoDB/S3
  * **Click redirect URL** → **CloudFront/API Gateway → Lambda** (log) → **302 redirect** to target
* Use when you want full control over event schema, attribution, and reporting.

**Optional (Pinpoint alongside SES):**

* Add **Pinpoint** when you need marketing-grade segmentation/journeys and richer engagement analytics beyond basic open/click.

### H) Payments (Stripe)

1. User completes payment in Stripe UI → Stripe sends webhook
2. **Stripe → API Gateway (webhook endpoint) → Lambda (webhook handler)**
3. Lambda reads secrets from **Secrets Manager**, verifies signature, updates **DynamoDB** (plan/subscription state)
4. Optional: enqueue follow-up actions via **SQS** (e.g., enable sending limits, trigger onboarding email)

### I) Observability + security baseline

* **CloudWatch Logs/Metrics/Alarms**: Lambda, API Gateway, SQS depth, DLQ count, SES metrics
* **CloudTrail**: audit trail of AWS API actions
* **KMS**: encryption at rest for DynamoDB/S3/Secrets
* **IAM**: least-privilege roles per Lambda/component

---

## Mermaid diagram (single end-to-end graph)

```mermaid
flowchart LR
  %% --- Clients ---
  U[User Browser] --> R53[Route 53]
  B[Search Engine Bot] --> R53

  %% --- Edge / Delivery ---
  R53 --> CF[CloudFront + ACM TLS]
  WAF[WAF] -. protects .- CF

  CF --> CFF[CloudFront Function\n(viewer-request: classify UA)]
  CFF -->|normal user| S3[S3: Svelte static site]
  S3 --> CF --> U

  CFF -->|bot| LEE[Lambda@Edge (Node.js SSR)]
  LEE --> CF --> B

  %% --- Auth ---
  U -->|sign-in/up| COG[Cognito User Pools\n(email/phone + Google)]
  COG -->|JWT tokens| U

  %% --- APIs / App Data ---
  U -->|API calls (JWT)| APIGW[API Gateway]
  APIGW --> API1[Lambda: API handlers]
  API1 --> DDB[(DynamoDB:\ncontacts/templates/campaigns/jobs)]
  API1 --> S3A[S3 (optional:\nlarge templates/assets/import/export)]
  API1 --> APIGW --> U

  %% --- Scheduling / Dispatch ---
  EBS[EventBridge Scheduler\n(per-job)] --> SCH[Lambda: Scheduler/Dispatcher]
  EBR[EventBridge Scheduled Rule\n(poller)] --> SCH
  SCH -->|query due| DDB
  SCH -->|enqueue| SQS[SQS: EmailQueue]
  SQS -->|failures| DLQ[SQS DLQ]

  %% --- Sending ---
  SQS --> SEND[Lambda: EmailSender]
  SEND --> SES[Amazon SES]
  SEND -->|update status/idempotency| DDB

  %% --- Delivery events ---
  SES --> SNS[SNS: Delivery events]
  SNS --> EVH[Lambda: EventHandler]
  EVH --> DDB
  EVH --> S3E[S3 (optional:\narchive events)]

  %% --- Engagement tracking ---
  subgraph Tracking[Engagement Tracking]
    SESCS[SES Configuration Set\n(open/click tracking)] -. enables .- SES
    U -->|pixel/click URLs (optional self-hosted)| CF
    CF --> TRK[API Gateway/Lambda\ntracking endpoints]
    TRK --> DDB
    TRK --> S3T[S3 (optional:\nraw logs)]
  end

  %% --- Payments ---
  STR[Stripe] -->|webhooks| APIGW2[API Gateway: webhook endpoint]
  APIGW2 --> PAY[Lambda: Stripe webhook handler]
  PAY --> SM[Secrets Manager]
  PAY --> DDB
  PAY -->|optional follow-ups| SQS

  %% --- Observability / Governance ---
  CW[CloudWatch Logs/Metrics/Alarms] <-. telemetry .-> APIGW
  CW <-. telemetry .-> API1
  CW <-. telemetry .-> SCH
  CW <-. telemetry .-> SEND
  CW <-. telemetry .-> SQS
  CW <-. telemetry .-> SES

  CT[CloudTrail] -. audit .- CF
  CT -. audit .- APIGW
  CT -. audit .- API1

  KMS[KMS] -. encrypts .- DDB
  KMS -. encrypts .- S3A
  KMS -. encrypts .- SM
```

If you want a cleaner diagram, you can split it into 3 sub-graphs: **(1) Web/Auth/API**, **(2) Scheduling/Sending**, **(3) Tracking/Events/Payments**.




-----------------------




**Mode 2: Simple Q&A**

Here is the breakdown of your optimized architecture.

###1. Core Data & CRM (The "Relational" Fix)* **Problem:** You need to filter contacts by multiple tags, projects, and custom attributes (e.g., "Find VIPs in New York"). DynamoDB makes this hard/expensive.
* **AWS Service:** **Amazon RDS for PostgreSQL**.
* **Why:**
* It handles complex relationships natively (Contacts ↔ Projects ↔ Invoices).
* **JSONB Support:** You can store custom contact fields (tags, social links) in a `JSONB` column and index them, giving you the flexibility of NoSQL with the structure of SQL.
* **Search:** You can use PostgreSQL's built-in **Full Text Search** for finding names/emails, avoiding the $175/mo cost of OpenSearch.



###2. Immediate Email Sending (The "Buffer")* **Problem:** Sending 50,000 emails instantly will throttle your system or hit AWS limits.
* **AWS Service:** **Amazon SQS** (Standard Queue).
* **Flow:** `App (PostgreSQL)` → `SQS` → `Lambda` → `Amazon SES`.
* **Why:** The SQS queue absorbs the instant burst of "Send" requests from your app. The Lambda pulls messages at a controlled rate (e.g., 50/sec) to ensure reliable delivery without errors.

###3. Automated Sequences (The "Send Later" Engine)* **Problem:** SQS cannot delay messages for more than **15 minutes**. You cannot use it alone for "Send email in 3 days."
* **AWS Service:** **Amazon EventBridge Scheduler**.
* **Why:** It is a serverless "timer" service. When a user enters a sequence, you create a schedule: *"On, send this payload to the SQS queue."*
* **Alternative (Since you have SQL):** You can use a **Polling Pattern**. Store a `due_at` timestamp in PostgreSQL. A cron job runs every minute: `SELECT * FROM email_queue WHERE due_at <= NOW() FOR UPDATE SKIP LOCKED` and sends them. This avoids a new AWS service but adds code complexity.

###4. Client Tracking (Analytics)* **Problem:** If you send a newsletter to 100k people, 20k might open it in the first 10 minutes. Direct writes to your database can lock tables and slow down your CRM.
* **AWS Service:** **Amazon SNS** + **SQS**.
* **Flow:** `SES (Open/Click Event)` → `SNS` → `SQS` → `Lambda` → `PostgreSQL`.
* **Why:** SES cannot write to your database directly. It sends events to SNS. You add an SQS queue here to **buffer** the spikes of traffic. The Lambda processes the queue in batches and updates your PostgreSQL `analytics_events` table efficiently.

###Summary Checklist| Feature | Service / Tool | Function |
| --- | --- | --- |
| **CRM Data** | **RDS (PostgreSQL)** | Stores contacts, projects, tags. Uses `JSONB` for flexibility. |
| **Search** | **PostgreSQL** | Uses `ILIKE` or `tsvector` for contact search (saves money). |
| **Sending** | **SQS** | Buffers outgoing emails so you don't hit limits. |
| **Sequences** | **EventBridge Scheduler** | "Wakes up" your app when it's time to send a follow-up. |
| **Tracking** | **SNS + SQS** | Buffers incoming "Open/Click" events before saving to DB. |