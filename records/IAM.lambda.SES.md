# AWS SES + Lambda Email Sending: Setup Record (Improved)

## Scope and assumptions

* SES is used in **Sandbox mode** for integration testing.
* All **sender and recipient** addresses used in tests are **verified SES identities**.
* SES identity verification and sandbox status are **region-specific**: the SES region you verify identities in must match the region your Lambda uses to call SES.

---

## Step 0: Prerequisites (one-time)

* Choose a single AWS Region for this test (e.g., `eu-west-1`) and keep SES + Lambda in that region.
* Decide the sending identity:

  * Verified **email identity** (e.g., `L.Han2@uni.brighton.ac.uk`), or
  * Verified **domain identity** (optional; not required for sandbox testing).

---

## Step 1: Prepare Amazon SES (Sandbox mode)

1. Open **Amazon SES** (in your chosen region).
2. Go to **Verified identities → Create identity → Email address**.
3. Verify:

   * **Sender** email (From address).
   * **Recipient** emails you will test with (To addresses).
4. Confirm each verification by clicking the SES link delivered to that mailbox.

**Sandbox implications**

* You can only send **from verified identities**.
* You can only send **to verified identities** (unless using SES mailbox simulator).
* Sending quotas are low in sandbox (commonly ~200 emails/day and low per-second rate); treat this as a functional test environment, not load testing.

---

## Step 2: Create IAM role for Lambda (least privilege)

1. Go to **IAM → Roles → Create role**.
2. **Trusted entity**: *AWS service* → **Lambda**
   (creates a trust policy allowing Lambda to assume the role via STS).
3. Attach managed policy:

   * `AWSLambdaBasicExecutionRole` (CloudWatch Logs).
4. Add an **inline policy** for SES sending (preferred over `AmazonSESFullAccess`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSesSendEmail",
      "Effect": "Allow",
      "Action": ["ses:SendEmail"],
      "Resource": "*"
    }
  ]
}
```

> Notes
>
> * `ses:SendEmail` is the IAM permission needed for both SES v1 and SESv2 send operations.
> * You can later restrict `Resource` to specific SES identity ARNs, but `*` is acceptable for a minimal sandbox test.

Name the role clearly, e.g., `Lambda_SES_Send_Role`.

---

## Step 3: Create the Lambda function

1. Go to **AWS Lambda → Create function → Author from scratch**.
2. Runtime: **Python 3.11** (or similar).
3. Execution role:

   * **Use an existing role** → select `Lambda_SES_Send_Role`.
4. (Recommended) Set environment variables:

   * `FROM_EMAIL = L.Han2@uni.brighton.ac.uk`
   * `SES_REGION = <your-region>` (e.g., `eu-west-1`)

---

## Step 4: Implement SESv2 email sending (minimal code)

**Define FROM_EMAIL env variable**

Use SESv2 `SendEmail` with simple text content:

```python
import os
import boto3

SES_REGION = os.environ.get("SES_REGION")
client = boto3.client("sesv2", region_name=SES_REGION) if SES_REGION else boto3.client("sesv2")

FROM_EMAIL = os.environ["FROM_EMAIL"]

def lambda_handler(event, context):
    to_email = event["to"]  # must be verified in sandbox
    subject = event.get("subject", "SES sandbox test")
    body = event.get("body", "Hello from AWS Lambda + SESv2.")

    resp = client.send_email(
        FromEmailAddress=FROM_EMAIL,
        Destination={"ToAddresses": [to_email]},
        Content={
            "Simple": {
                "Subject": {"Data": subject, "Charset": "UTF-8"},
                "Body": {"Text": {"Data": body, "Charset": "UTF-8"}}
            }
        }
    )

    return {"messageId": resp["MessageId"]}
```

**Why env vars**

* Keeps code immutable while allowing identity changes without redeploy.
* Reduces accidental commits of sensitive configuration.

---

## Step 5: Test the Lambda function

1. In Lambda, create a **Test event**:

```json
{
  "to": "verified-recipient@example.com",
  "subject": "Test from Lambda",
  "body": "If you see this, SESv2 works."
}
```

2. Click **Test**.
3. Validate success:

   * Lambda returns a `messageId`.
   * **CloudWatch Logs** show no exceptions.
   * Recipient inbox receives the email (check spam/junk).

---

## Step 6: Validation checklist + common failure modes

**Checklist**

* SES identities verified in the **same region** as Lambda/SES client.
* Sender (`FROM_EMAIL`) is verified.
* Recipient is verified (sandbox requirement).
* Lambda execution role includes `ses:SendEmail`.

**Common errors**

* `MessageRejected` / “Email address is not verified”: recipient or sender not verified *in that region*.
* `AccessDenied`: IAM role missing SES permission.
* Silent non-delivery: email landed in spam; for institutional addresses, filtering can be strict.

---

## Step 7: Next step (queue-based sending)

For an email queue, the minimal production-grade direction is:

* **SQS queue + DLQ**
* Lambda triggered by SQS (batch size small, concurrency controlled)
* Worker calls SESv2 `SendEmail`
* Log messageId + payload hash for traceability

In sandbox, enforce strict throttling (e.g., Lambda reserved concurrency = 1) to avoid exceeding send-rate limits.
