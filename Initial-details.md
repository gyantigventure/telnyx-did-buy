10DLC SMS via Telnyx – Detailed Implementation & Compliance Guide

1. Introduction to 10DLC

10DLC stands for 10-Digit Long Code—a U.S. carrier-approved messaging route for A2P (Application-to-Person) SMS and MMS.
It is designed to allow businesses to send high-throughput, compliant messages using standard local phone numbers.

Key points:
	•	Required for all A2P messaging in the U.S.
	•	Carriers (AT&T, T-Mobile, Verizon, etc.) mandate brand and campaign registration through The Campaign Registry (TCR).
	•	Improves deliverability, reduces filtering, and avoids penalties for unregistered traffic.

⸻

2. Regulatory Requirements

U.S. carriers enforce strict rules for 10DLC messaging.
To use 10DLC via Telnyx:
	1.	Brand Registration – Identify your business with TCR (via Telnyx).
	2.	Campaign Registration – Define the use case(s) for your messaging traffic.
	3.	Phone Number Linking – Assign your 10-digit local numbers to the registered campaign.
	4.	Message Compliance – Follow carrier rules on opt-in, opt-out, and content.

💡 Penalties for Non-Compliance:
Unregistered traffic may be blocked or fined ($10 to $500 per message in some cases).

⸻

3. Step-by-Step Setup on Telnyx

Step 1 – Prepare Your Brand Information

You’ll need:
	•	Legal Business Name
	•	Business Type (LLC, Corporation, Sole Proprietor)
	•	EIN / Tax ID (for U.S.) or DUNS number
	•	Business Address
	•	Website
	•	Contact Information (phone & email)

Step 2 – Register Brand on Telnyx
	1.	Log in to Telnyx Mission Control.
	2.	Go to Messaging → 10DLC Campaigns.
	3.	Click Register New Brand.
	4.	Fill in required brand details.
	5.	Telnyx submits your brand to The Campaign Registry for vetting.

📌 Note: Brand vetting may result in a Trust Score (0–100), which affects throughput limits.

Step 3 – Register Campaign
	1.	In Telnyx, go to Messaging → 10DLC Campaigns.
	2.	Click Register New Campaign.
	3.	Select:
	•	Brand (from Step 2)
	•	Campaign Use Case (see Section 4 below)
	4.	Provide:
	•	Sample Messages (actual examples of what you’ll send)
	•	Opt-in Process Description
	•	Opt-out Instructions (e.g., “Reply STOP to unsubscribe”)
	5.	Submit for approval.

⏳ Approval time: Usually 1–3 business days.

Step 4 – Link Numbers to Campaign
	1.	Purchase local numbers from Telnyx or use existing numbers.
	2.	In Telnyx:
	•	Go to Numbers → My Numbers
	•	Select numbers → Assign to Campaign
	3.	Numbers are now authorized for registered 10DLC messaging.

Step 5 – Configure Messaging Profile
	1.	Create or edit a Messaging Profile in Telnyx.
	2.	Assign your campaign numbers to the profile.
	3.	Set Webhook URLs for inbound message handling (API endpoint).
	4.	Configure Encoding, DLRs, MMS settings as needed.

⸻

4. Campaign Use Cases

TCR has predefined use cases.
Common ones:
	•	Customer Care – Support and account assistance.
	•	Marketing – Promotions, sales offers.
	•	Two-Factor Authentication (2FA) – OTP messages.
	•	Account Notifications – Transaction alerts, billing reminders.
	•	Public Service – Emergency alerts, community messages.

📌 Each use case has its own carrier fees and throughput limits.

⸻

5. Messaging Best Practices

To ensure high deliverability and compliance:
	•	Opt-in Required – Obtain explicit consent before messaging.
	•	Opt-out Support – Recognize “STOP”, “UNSUBSCRIBE”, “CANCEL” keywords automatically.
	•	Content Transparency – Include business name in messages.
	•	Avoid Spam Triggers – No SHAFT content (Sex, Hate, Alcohol, Firearms, Tobacco) unless specifically allowed.
	•	Frequency Control – Avoid sending too many messages in a short time.
	•	Segment Management – Use message segmentation for >160 GSM-7 characters or >70 UCS-2 characters.

Example Compliant Message:

[BrandName]: Your appointment is confirmed for Aug 12 at 3:00 PM. Reply STOP to opt out.


⸻

6. Throughput & Pricing

Throughput is based on:
	•	Trust Score
	•	Campaign Type
	•	Carrier Policies

Typical:
	•	Low Trust Score: 1–5 MPS (Messages per second)
	•	High Trust Score: Up to 60 MPS (depending on carrier)

Costs include:
	•	Brand Registration Fee (one-time)
	•	Campaign Monthly Fee
	•	Per-Message Carrier Surcharges
	•	Telnyx per-SMS rate (U.S. domestic)

⸻

7. Technical Integration with Telnyx API

Send SMS via Telnyx API:

curl -X POST \
  https://api.telnyx.com/v2/messages \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "+12125551234",
    "to": "+13125551234",
    "text": "Hello from Telnyx 10DLC!"
  }'

Inbound Webhook Example (Node.js Express)

app.post('/inbound-sms', (req, res) => {
  const message = req.body;
  console.log(`Message from ${message.from}: ${message.text}`);
  res.sendStatus(200);
});


⸻

8. Monitoring & Reporting

Telnyx provides:
	•	Delivery Reports (DLRs) – Per message status.
	•	Campaign Health Reports – Opt-outs, complaint rates.
	•	API Logs – Message request & response data.
	•	Error Codes – To troubleshoot failed deliveries.

💡 Tip: Maintain complaint rates <1% to avoid suspension.

⸻

9. Common Pitfalls to Avoid
	•	Registering campaign with vague or incomplete use case descriptions.
	•	Not keeping sample messages consistent with actual traffic.
	•	Forgetting to include opt-out instructions.
	•	Using numbers outside of registered campaigns.
	•	Sending spam or SHAFT content without special approval.

⸻

10. Resources
	•	Telnyx 10DLC Docs: https://support.telnyx.com
	•	The Campaign Registry: https://www.campaignregistry.com
	•	CTIA Messaging Principles: https://www.ctia.org
