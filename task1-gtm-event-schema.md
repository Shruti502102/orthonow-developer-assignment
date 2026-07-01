# Task 01 — GTM Event Schema (OrthoNow)

## 1. Full Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | GA4 Report / Audience it Feeds |
|---|---|---|---|
| `call_now_click` | Click Trigger — "Just Links", Click URL contains `tel:` (fallback: CSS class `.call-now-btn` for icon buttons that aren't `<a>` tags) | `page_location`, `clinic_name` (blank on homepage), `button_position` (header / clinic-page / sticky-mobile-bar) | Engagement > Events; feeds a **"High-Intent — Call"** remarketing audience for Search/Performance Max |
| `whatsapp_click` | Click Trigger — "Just Links", Click URL contains `wa.me` | `page_location`, `clinic_name`, `widget_state` (floating-button / inline-embed) | Engagement > Events; feeds a **"WhatsApp Intent"** remarketing audience |
| `patient_guide_form_submit` | Form Submission trigger scoped to the gated-download form ID (or Custom Event if the form uses JS validation that blocks native submit) | `form_id`, `page_location`, `lead_source` | Marked as a GA4 **Key Event** (secondary conversion); feeds **"Guide Downloaders"** nurture audience |
| `patient_guide_download` | Click Trigger — "Just Links", File extension `.pdf` (fires only after the gated form succeeds — the PDF link should be dynamically revealed, not present pre-submit) | `file_name`, `file_url`, `page_location` | GA4 File Download report (auto-tagged); cross-checked against `patient_guide_form_submit` to confirm the gate is enforced |
| `clinic_page_view` | Custom Event trigger — `dataLayer.push({event:'clinic_page_view', ...})` fired inline by the template on load (NOT relying on GA4's automatic `page_view`, since that won't carry `clinic_name` as a usable dimension without extra config) | `clinic_name`, `clinic_city`, `page_path` | Feeds **9 location-level audiences** for geo-targeted remarketing (e.g. "Interested — Whitefield Clinic") |
| `blog_scroll_depth` | Built-in GTM **Scroll Depth** trigger, Vertical Percentages: 25, 50, 75, 90 | `percent_scrolled`, `page_path` / `article_title`, `content_category` | Engagement report; feeds a **"Engaged Readers"** audience used for top-of-funnel content remarketing |
| `booking_step_complete` | Custom Event trigger — `dataLayer.push({event:'booking_step_complete', step_number:...})` fired by front-end on each step transition | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1) — see Section 2 for full detail per step | GA4 **Funnel Exploration** — this is the funnel backbone |
| `booking_confirmed` | Custom Event trigger — `dataLayer.push({event:'booking_confirmed', ...})` fired only after the backend returns a successful booking response (not on step-3 button click — that's a submission attempt, not a confirmed booking) | `booking_id`, `clinic_location`, `specialty`, `lead_source` | GA4 **Key Event** / primary conversion. This is the one imported into Google Ads (see Section 3) |

**PII note:** none of these events push `name` or `phone` into the dataLayer. GA4 terms of service prohibit sending PII. Where step 2 needs to signal "contact details were entered," it pushes a boolean flag (`contact_info_entered: true`) or a non-reversible `lead_id`, never the raw values.

---

## 2. Booking Funnel — Step-Level Drop-off Tracking

**Why GTM can't do this natively:** the form doesn't reload the page or change the URL between steps, and the "Next" buttons intercept the native submit event. There is no browser-level signal for GTM to hook into. Every step transition has to be **explicitly pushed to the dataLayer by the front-end developer**, at the point in the code where the step actually changes (e.g. inside the `onNextStepClick()` handler, after validation passes and before the DOM swaps to the next step's fields).

GTM's only job is a **Custom Event trigger** listening for `event: 'booking_step_complete'`, which fires a GA4 event tag on every match. `step_number` is read from the dataLayer variable and sent through as an event parameter.

**dataLayer push — Step 1 (location + specialty selected):**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**dataLayer push — Step 2 (contact details entered — no PII):**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date_selected": true,
  "contact_info_entered": true
}
```

**dataLayer push — Step 3 (booking confirmed by the user, before backend response):**
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_review_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Separate final push — actual conversion, after backend success:**
```json
{
  "event": "booking_confirmed",
  "booking_id": "{{server-generated booking id}}",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "lead_source": "google_ads_consultation_lp"
}
```

**GTM setup:**
- One Custom Event trigger matching `event equals booking_step_complete` → fires a single GA4 Event tag named `booking_step_complete`, with `step_number` and `step_name` mapped as event parameters (not baked into the trigger — one trigger, parameterized event, not three separate triggers).
- A second Custom Event trigger matching `event equals booking_confirmed` → fires a GA4 Event tag marked as a **Key Event** in GA4 Admin.
- In GA4 Admin, register `step_number` and `step_name` as **event-scoped custom dimensions** — without this they won't be selectable as a breakdown in Funnel Exploration.

**Surfacing drop-off in GA4 Funnel Exploration:**
1. Build a funnel with 4 open steps: `booking_step_complete` (step_number = 1) → `booking_step_complete` (step_number = 2) → `booking_step_complete` (step_number = 3) → `booking_confirmed`.
2. Turn **"Open funnel"** on (not closed) so partial completions are still counted, since the point is to see where people abandon, not just who completed.
3. Leave "make step immediate" off to allow for realistic time gaps between steps (people get interrupted mid-form).
4. Add a breakdown dimension of `clinic_location` on step 1 → this tells you if drop-off is a location-specific problem (e.g. a clinic with no near-term availability causing step-2 abandonment) versus a global UX problem.
5. Use the built-in "time to complete step" metric to catch a slow step (e.g. if step 2 → step 3 has a much longer median time, the review screen is probably confusing).

**Who writes the dataLayer push:** the front-end developer, not GTM. My brief to the dev team for step 2 specifically would be: "In the handler that validates and advances from step 2 to step 3, right after validation passes and before you render step 3's DOM, add `window.dataLayer.push({...})` with this exact JSON shape [above]. Do not include the phone number field — push `contact_info_entered: true` instead. Test it by opening the browser console, submitting step 2, and confirming the object appears in `window.dataLayer`." I'd then validate independently using GTM Preview mode before anything goes live.

---

## 3. Conversion Action to Import into Google Ads

**Import `booking_confirmed`, not `call_now_click`, `whatsapp_click`, or `booking_step_complete`.**

Reasoning:
- Google Ads' bidding algorithms (Target CPA / Maximize Conversions / tROAS) optimize the campaign toward whatever signal you import. Importing a soft-intent event like `call_now_click` or a mid-funnel event like `booking_step_complete` (step 1) would train the campaign to chase people who click a button or start a form — not people who actually book. Given the current 2.1% baseline conversion problem, the campaign needs to learn from the highest-intent, closest-to-revenue signal available, which is a confirmed appointment.
- `booking_confirmed` fires only after a successful backend response, so it's clean — it can't be inflated by front-end double-fires or abandoned submissions the way `booking_step_complete` (step 3) could be.
- Secondary/observation signal: once volume is healthy, `patient_guide_form_submit` can be imported as a secondary (non-bid-affecting) conversion to give the algorithm more training signal without diluting optimization toward the primary action.
