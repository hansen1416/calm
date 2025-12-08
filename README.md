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
