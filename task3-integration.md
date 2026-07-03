# Task 03 — Integration Design (Landing Page → HubSpot → WhatsApp → Google Ads)

## End-to-end architecture

I would use a **direct server-side API call from a lightweight middleware function**, not Zapier/Make and not HubSpot's native embedded form.

**Flow:** Landing page form → serverless function (e.g. AWS Lambda / Vercel Function) → HubSpot Contacts API + Karix WhatsApp API in parallel → response back to browser → GTM/Google Ads conversion fires client-side.

Why not native HubSpot embed: it can't run custom dedup logic or trigger WhatsApp in the same flow — it only writes to HubSpot.

Why not Zapier/Make: added latency (typically 1–3 min polling delay in the free/standard tiers) directly threatens the 2-minute WhatsApp SLA, and they're harder to unit-test and version-control than code we own.

**Step-by-step:**
1. Form submits to our own lightweight backend endpoint (not directly to HubSpot), so we control logic before anything touches third parties.
2. Backend **first checks HubSpot by phone number** using the Contacts Search API (`filterGroups` on the phone property), since phone is the real unique identifier here, not email.
3. If found → **update** existing contact (append new enquiry as a note/timeline event, update Lead Status). If not found → **create** new contact with Name, Phone, Clinic Preference, Source, Lead Status.
4. Backend calls Karix's WhatsApp Business API directly with a templated confirmation message.
5. Backend returns success to the browser, which pushes the `consultation_form_submitted` event to `dataLayer`, letting the GTM-configured GA4/Google Ads tag fire client-side.

## The dedup trap

HubSpot's default deduplication key is **email**, not phone. Since this form never collects email, every resubmission by the same patient would silently create a **duplicate contact** instead of updating the existing one. If two patients submit with the same phone but different names, my setup would flag this as a **collision**, not silently overwrite: I'd search by phone first, and if the existing contact's name meaningfully differs, log it to a "manual review" property/list in HubSpot rather than blindly renaming the record — since it could be a shared family phone, a typo, or the same patient using a slightly different name.

## Biggest failure point + fallback

The single biggest failure point is the **synchronous chain**: if the HubSpot API call fails, we lose the lead entirely and the patient sees an error with no recovery. Fallback: write every submission to a durable queue (e.g. a database table or SQS) *before* calling any external API, then process HubSpot/WhatsApp calls from that queue with retries. The user always sees success once their submission is queued; failures retry in the background instead of being lost.

## Monitoring the WhatsApp 2-minute SLA

What could break it: Karix API rate limits, WhatsApp template approval issues, or queue backlog during traffic spikes. I'd monitor via a scheduled job that checks "queued but unsent after 90 seconds" and alerts the team on Slack/PagerDuty, plus a daily dashboard tracking p50/p95 time-to-send.
