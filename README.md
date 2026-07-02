# OrthoNow Developer Assignment — Shruti Srivastava

- [Task 1 — GTM Event Schema]([./Task01_GTM_Schema.md](https://github.com/Shruti502102/orthonow-developer-assignment/blob/main/task1-gtm-event-schema.md)
- [Task 2 — Landing Page](./task2-landing-page.html) ([PageSpeed screenshot](https://github.com/Shruti502102/orthonow-developer-assignment/blob/main/photo/WhatsApp%20Image%202026-07-02%20at%2012.04.13%20AM.jpeg))
- [Task 3 — Integration Design](#task-03--integration-design-orthonow)

---
## Task 2 — Landing Page

- Live page: https://shruti502102.github.io/orthonow-developer-assignment/task2-landing-page.html
- Source file: [task2-landing-page.html](./task2-landing-page.html)
- PageSpeed Check link: https://pagespeed.web.dev/analysis/https-shruti502102-github-io-orthonow-developer-assignment-task2-landing-page-html/yykmyynz4v?form_factor=mobile
- PageSpeed Mobile score: 100 Performance / 96 Accessibility / 100 Best Practices / 100 SEO



![PageSpeed Insights Mobile score](https://github.com/Shruti502102/orthonow-developer-assignment/blob/main/photo/WhatsApp%20Image%202026-07-02%20at%206.25.02%20PM.jpeg)

# Task 03 — Integration Design (OrthoNow → HubSpot → WhatsApp → Google Ads)

## Architecture

I'd route this through **Make** (formerly Integromat) rather than a direct API call or Zapier. A direct HubSpot API call from the landing page's JS is unsafe — it exposes a private app token client-side — so it needs a server-side intermediary regardless. Between Zapier and Make, Make wins here because the flow has a conditional branch (dedup logic) and three destinations firing off one trigger; Make's visual router and per-step error handling make that explicit and debuggable, whereas Zapier's linear multi-step Zaps get awkward once you need branching logic and per-branch retries.

**Flow:** Form submit → JS fires the `consultation_form_submitted` dataLayer push (for GTM/GA4) **and**, separately, POSTs the form payload to a Make webhook (not the native HubSpot embed — the native embed's own submission handler is a black box I can't branch logic around). Make receives the payload and: (1) calls the **HubSpot CRM API's contact search-then-upsert** pattern — search by phone first, since phone is our real identity key, then create or update — setting Name, Phone, Clinic Preference, Source, and Lead Status; (2) calls Karix's WhatsApp Business API to send the confirmation template message; (3) calls the Google Ads Conversion Import API (via HubSpot's own Ads integration, or a direct offline-conversion upload keyed to the GCLID captured on page load) to fire `consultation_form_submitted`.

## The phone-dedup trap

HubSpot deduplicates contacts on **email by default**, not phone — and we don't collect email. Left as-is, every submission creates a new contact even from the same patient, silently fragmenting their record. My fix: before creating, Make runs a **CRM search by phone number** (HubSpot's Search API, filtering the phone property) and branches — match found → update that record; no match → create new. If two patients submit with the same phone number but different names (a shared family phone, common in Indian households), the setup should **not silently overwrite the name** — it should update Lead Status/Source but append the new name as a note/activity on the same contact, flagged for manual review, rather than assuming it's the same person.

## Biggest failure point & SLA risk

The single biggest failure point is the **Karix WhatsApp call**, since it's the one external dependency with no retry built into the platform itself. Fallback: Make's built-in error handler retries twice with backoff, then falls back to an SMS send via the same webhook if WhatsApp fails, and logs a failed-notification flag on the HubSpot contact so a human follows up. For the 2-minute SLA, the main threats are Make queue delays under traffic spikes and Karix template approval/rate limits — monitored via a Make scenario-history alert if any run exceeds 90 seconds, escalating to Slack.

...
