**Core SES email queue – action list**

1. **SES setup**

   * Verify sending domain (DKIM/SPF).
   * Request production access.
   * Create IAM user/role for SES and test `send_email` via AWS SDK.

2. **Email job model**

   * Create `EmailJob` storage with fields:
     `id, to, from, subject, body_html, body_text, send_at, status (PENDING/SENDING/SENT/FAILED/CANCELLED), attempt_count, last_error, ses_message_id, opened_at`.

3. **Worker**

   * Periodically fetch a batch of jobs where `status='PENDING'` and `send_at <= now` (using row locking / atomic claim).
   * Mark jobs as `SENDING`.
   * Call `ses.send_email()` for each; store returned `MessageId`.
   * On success: `status='SENT'`.
   * On error: increment `attempt_count`; if `< MAX_ATTEMPTS` → reschedule with backoff; else `status='FAILED'` and log `last_error`.

4. **Rate limiting**

   * Limit batch size per loop (e.g. N jobs).
   * Sleep between batches or adjust dynamically on throttling errors.

5. **Unsubscribe & suppression**

   * For each contact, store `unsubscribe_token` and flags `is_unsubscribed`, `is_bounced`, `is_complaint`.
   * Add `/unsubscribe?token=...` endpoint → set `is_unsubscribed = true`.
   * In enqueuer (campaign → jobs) **and/or** immediately before sending, skip any contact with suppression flags.

6. **Bounce/complaint handling (SNS)**

   * Configure SES to publish bounce/complaint notifications to SNS.
   * Subscribe your backend HTTPS endpoint.
   * For each event, mark the affected email as `is_bounced` or `is_complaint` and optionally cancel future jobs for that address.

7. **Recurring scheduler**

   * Maintain `Campaign` with `schedule_type` and `next_run_at`.
   * Periodic scheduler process:

     * Find campaigns where `next_run_at <= now`.
     * Generate `EmailJob`s for all eligible recipients (enqueuer).
     * Set `next_run_at` to next occurrence (e.g. `+7 days` for weekly).

---

**Open tracking via SES configuration sets – action list**

1. **Create configuration set**

   * In SES, create a **Configuration Set** (e.g. `marketing-config`).
   * Enable **open and click tracking** for this configuration set.

2. **Configure event destination**

   * Add an event destination (SNS / EventBridge / Kinesis).
   * Include at least `send`, `delivery`, `bounce`, `complaint`, `open`, `click` events.

3. **Send with config set**

   * In your worker’s `send_email` call, set `ConfigurationSetName='marketing-config'`.

4. **Consume engagement events**

   * Subscribe your backend to the event destination (e.g. HTTPS endpoint subscribed to SNS).
   * For each event:

     * Extract SES `messageId`.
     * Map `messageId` → your `EmailJob`.
     * On `open` event: set `opened_at = now()` (and optionally increment open counters per campaign).



----------------------------



Good — with 3 verified emails, the next practical step is to **validate your sending path**, then **wrap it with a queue + worker**.

## 1) Send one email via API (sanity check)

Use **SESv2 `SendEmail`** (simple) from your verified sender → one verified recipient. ([AWS Documentation][1])
Make sure you’re using the **same AWS region** where the identity is verified. ([AWS Documentation][2])

## 2) Build the queue architecture (SQS → Lambda → SES)

**Recommended minimal pipeline**

1. **Create an SQS queue** (plus a DLQ).
2. **Create a Lambda consumer** triggered by SQS.
3. Lambda parses each message and calls **SESv2 `SendEmail`**.
4. On success: delete message; on failure: let Lambda/SQS retry, then DLQ.

AWS has canonical guidance for “Lambda → SES send” and “Lambda consumes SQS” that matches this pattern. ([Repost][3])

## 3) Apply sandbox constraints (important for queues)

While in the **SES sandbox**, enforce throttling in your worker:

* **Max 200 messages / 24h** and **~1 message/second** (sandbox defaults). ([AWS Documentation][4])
  So set **Lambda reserved concurrency = 1** (simple throttle) or implement rate limiting.

## 4) IAM permissions for the worker

Give the Lambda execution role:

* `ses:SendEmail` (and `ses:SendRawEmail` only if you need MIME/attachments),
* CloudWatch Logs permissions.

This is required for Lambda to call SES. ([Repost][3])

## 5) (Optional, but good) Observability + failure handling

* Use CloudWatch logs/metrics for sends.
* If you later move out of sandbox, add bounce/complaint handling (SNS/event publishing), but you can postpone this for sandbox-only testing.

If you want, I can give you a minimal SQS message schema + a short Lambda (Python/Node) `SendEmail` handler that respects the 1 msg/s sandbox rate.

[1]: https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_SendEmail.html?utm_source=chatgpt.com "SendEmail - Amazon Simple Email Service"
[2]: https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html?utm_source=chatgpt.com "Creating and verifying identities in Amazon SES"
[3]: https://repost.aws/knowledge-center/lambda-send-email-ses?utm_source=chatgpt.com "How do I use Lambda and Amazon SES to send email?"
[4]: https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas.html?utm_source=chatgpt.com "Managing your Amazon SES sending limits"


-------------


* **Frontend**: Host as static files on **S3 + CloudFront**. No server-side rendering at this stage; revisit later if needed.
* **Backend**: Run **Laravel locally with Sail** for development.
* **Email workflow**: Use **SQS** for queuing and **SES** for sending; emulate both locally with **LocalStack** via the **AWS SDK**.
* **Future deployment**: If moving Laravel to **Lambda**, adapt at that stage; current focus is feature development.
* **Authentication**: **Cognito** can also be emulated locally using **LocalStack**.
* **Overall approach**: Develop everything locally first, then deploy smoothly to AWS when ready.
