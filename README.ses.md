## localstack

`awslocal ses verify-email-identity --email hello@example.com`

`awslocal ses list-identities`

{
    "Identities": [
        "hello@example.com"
    ]
}

awslocal ses send-email \
        --from "hello@example.com"   \
        --message 'Body={Text={Data="This is the email body"}},Subject={Data="This is the email subject"}'   \
        --destination 'ToAddresses=hanlongzhen4466@gmail.com'

-----------------


With **SES event publishing** (via configuration sets), the `eventType` values you can track are: ([AWS Documentation][1])

* **Send** — SES accepted the send request and will attempt delivery. ([AWS Documentation][2])
* **Delivery** — SES handed the message to the recipient’s mail server. ([AWS Documentation][1])
* **Bounce** — Delivery failed (hard/permanent or soft/transient after retries stop). ([AWS Documentation][1])
* **Complaint** — Recipient marked the message as spam (feedback loop). ([AWS Documentation][1])
* **Reject** — SES rejected the message before attempting delivery (e.g., policy/suppression/etc.). ([AWS Documentation][1])
* **Open** — Open tracking signal (tracking pixel loaded; HTML + images required). ([AWS Documentation][1])
* **Click** — Click tracking signal (tracked links). ([AWS Documentation][1])
* **Rendering Failure** — SES couldn’t render the template (templated email). ([AWS Documentation][1])
* **DeliveryDelay** — Delivery is delayed (includes delay type metadata). ([AWS Documentation][1])
* **Subscription** — Subscription/unsubscription preference changes when using SES contact lists/topics (subscription management). ([AWS Documentation][1])
