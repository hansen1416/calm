Volume:  thousands  
Type: marketing campaigns  
Per-user control: user can define their own templates; they can schedule recurring campaigns (e.g. weekly)  
Compliance: unsubscribe links, GDPR, tracking, etc.  

--

AWS SES


--


Here is a compact checklist you can follow.

---

## A. Verify sending domain in SES

1. **Create domain identity**

   * Open **Amazon SES console → Identities → Create identity**.
   * Choose **Domain**, enter `yourdomain.com`, and enable **Easy DKIM**.

2. **Add SES TXT record for verification**

   * SES shows a **TXT** DNS record for domain verification.
   * In your DNS provider (Route 53, Cloudflare, etc.), add that TXT record.

3. **Add DKIM CNAME records**

   * In the identity’s **Authentication / DKIM** section, click **Generate DKIM settings**.
   * Copy the **three CNAME records** SES gives you and add them to your DNS zone.

4. **Wait for verification**

   * DNS can take time to propagate (up to 72h in worst case).
   * In SES, the domain should show **Verified** and DKIM **Enabled** once records are picked up.

---

## B. Configure SPF for SES

You want SPF to say “SES is allowed to send for this domain.”

1. **If you already have an SPF record** (a TXT like `v=spf1 ... -all`):

   * Edit it to **include SES**:
     `v=spf1 <your existing mechanisms> include:amazonses.com -all`

2. **If you have no SPF record yet**:

   * Create a TXT record on the root of your sending domain with:
     `v=spf1 include:amazonses.com -all`

3. (Optional but recommended)

   * Configure a **custom MAIL FROM domain** in SES and add the MX + TXT records SES suggests; this aligns SPF with that subdomain.

Once these are done and propagated, your domain is properly authenticated with **DKIM + SPF** for SES.


------------------


Here is a tight step-by-step guide for **SES with a single email identity**.

---

### 1. Open SES in the AWS console

* Go to the SES console:
  `https://console.aws.amazon.com/ses/`
* Make sure you are in the **region** you will use (top-right region selector).

---

### 2. Create an email identity

1. In the SES left menu, click **Verified identities**.
2. Click **Create identity**.
3. For **Identity type**, choose **Email address** (not domain).
4. Enter the address you will send from, e.g. `yourname@gmail.com` or `you@university.ac.uk`.
5. Leave other options at defaults for now.
6. Click **Create identity**.

---

### 3. Confirm the verification email

1. SES sends a message from `no-reply-aws@amazon.com` to that address.
2. Open that email and click the verification link.
3. Go back to **Verified identities** in SES; the status for that email should change from **Pending** to **Verified**.

Now SES allows you to use that email address as the **From/Source** when sending.

---

### 4. Use this identity in your app

* In all SES API calls (or SMTP), set `Source` / `From` to exactly this verified address.
* While your SES account is still **in sandbox**, you can only send **to**:

  * verified addresses (created in the same way), or
  * SES’s mailbox simulator.

This is enough to:

* Build and test your **queue + worker + SES integration**,
* Send emails to yourself and a small set of verified test inboxes,

without buying a domain or setting up DKIM/SPF yet.
