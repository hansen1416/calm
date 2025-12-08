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
