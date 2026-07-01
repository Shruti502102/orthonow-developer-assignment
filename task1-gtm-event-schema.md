# Task 01 — GTM Event Schema (OrthoNow)

## Part 1: Full Event Schema

| Event Name | GTM Trigger Type | Key Parameters (min. 3) | GA4 Report / Conversion / Audience |
|---|---|---|---|
| `call_now_click` | Click Trigger — "All Elements" / "Just Links", condition: Click URL contains `tel:` (fallback: CSS class `.call-now-btn` for icon-only buttons that aren't real anchor tags) | `page_location`, `clinic_name` (empty on homepage), `button_position` (`header` / `clinic_page` / `sticky_mobile_bar`) | Engagement > Events; feeds a **"High-Intent — Phone"** remarketing audience used for Search and Performance Max exclusion/inclusion lists |
| `whatsapp_click` | Click Trigger — "Just Links", condition: Click URL contains `wa.me` | `page_location`, `clinic_name`, `widget_state` (`floating_button` / `inline_embed`) | Engagement > Events; feeds a **"WhatsApp Intent"** remarketing audience |
| `patient_guide_form_submit` | Form Submission trigger scoped to the gated-download form's Form ID (Custom Event trigger instead if the form intercepts native submit with JS validation) | `form_id`, `page_location`, `lead_source` | Marked as a GA4 **Key Event** (secondary/soft conversion); feeds a **"Guide Downloaders"** nurture audience for remarketing and email follow-up |
| `patient_guide_download` | Click Trigger — "Just Links", condition: Click URL file extension equals `.pdf`, fired only after the gate succeeds (the PDF link is rendered dynamically post-submit, not present in the DOM pre-submit) | `file_name`, `file_url`, `page_location` | GA4 File Download report (standard auto-event pattern, made explicit here so it's tied to `form_id`); cross-checked against `patient_guide_form_submit` volume to confirm the content gate is actually holding |
| `clinic_page_view` | Custom Event trigger — `dataLayer.push({event:'clinic_page_view', ...})` fired inline by the page template on load. **Not** relying on GA4's automatic `page_view` hit, because that event doesn't carry `clinic_name` as a usable dimension without this explicit push | `clinic_name`, `clinic_city`, `page_path` | Feeds **9 separate location-level remarketing audiences** (e.g. "Interested — Koramangala Clinic") for geo-targeted campaigns per clinic |
| `blog_scroll_depth` | Built-in GTM **Scroll Depth** trigger, Vertical Scroll Percentages: 25, 50, 75, 90 | `percent_scrolled`, `page_path` (or `article_title`), `content_category` | Engagement report; feeds an **"Engaged Content Readers"** audience for top-of-funnel remarketing into the consultation campaign |
| `booking_step_complete` | Custom Event trigger — `event equals booking_step_complete` — matches a `dataLayer.push()` fired by the front end on every step transition. One trigger, `step_number` read as a parameter (not three separate triggers) | `step_number`, `step_name`, `clinic_location`, `specialty` — full detail per step in Part 2 | GA4 **Funnel Exploration** — this is the funnel's structural backbone |
| `booking_confirmed` | Custom Event trigger — `event equals booking_confirmed` — fires only after the backend returns a successful booking response, never on the step-3 button click itself | `booking_id`, `clinic_location`, `specialty`, `lead_source` | GA4 **Key Event** — this is the **primary conversion**, and the one imported into Google Ads (Part 3) |

---

## Part 2: Tracking the 3-Step Booking Funnel

### Why GTM cannot natively track this form

GTM's out-of-the-box triggers — Click, Form Submission, Page View, History Change — all depend on something the browser exposes natively: a link click, a native `submit` event, a URL change. A 3-step booking form that advances via JavaScript, without a page reload and without changing the URL, gives GTM **no native signal to listen to**. The step-3 "Confirm" button usually intercepts the real submit event with `preventDefault()` and only sends data to the backend at the very end — so a Form Submission trigger would fire once, at the end, telling you nothing about where people dropped off between steps 1 and 2.

**The front-end developer must implement `dataLayer.push()` calls explicitly**, one at each step transition, in the code that actually changes the step (e.g. inside the "Next" button's `onClick` handler, immediately after validation passes and before the DOM swaps to the next step). GTM's only role is to listen for these pushes via a Custom Event trigger — it does not generate this data on its own.

### PII restriction

None of the pushes below include `name` or `phone`, in either raw or hashed form. GA4's terms of service prohibit sending personally identifiable information. Where a step needs to signal that contact details were captured, it sends a **boolean flag** or a **non-reversible lead ID**, never the underlying value.

### dataLayer JSON — Step 1 (location + specialty selected)

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

### dataLayer JSON — Step 2 (contact details entered)

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

Note the absence of `name` and `phone` — `contact_info_entered: true` is the boolean substitute. If a de-duplication or personalization use case genuinely needs to tie this event to a specific lead later, the correct pattern is a **non-reversible `lead_id`** generated client-side (e.g. a UUID stored in a first-party cookie), not the raw contact fields.

### dataLayer JSON — Step 3 (booking reviewed and confirmed by the user)

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_review_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

### dataLayer JSON — Final confirmation (fires only after backend success)

```json
{
  "event": "booking_confirmed",
  "booking_id": "{{server-generated booking id}}",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "lead_source": "google_ads_consultation_lp"
}
```

**`booking_step_complete` (step 3) vs. `booking_confirmed` are deliberately two different events.** Step 3 fires the moment the user clicks "Confirm" — it represents *submission intent*, not a guaranteed outcome. `booking_confirmed` fires only after the backend responds with success (e.g. the API call to create the appointment record returns a `200` with a booking ID). This distinction matters because a network failure, a double-booked slot, or a backend validation error between "user clicked confirm" and "appointment actually created" would otherwise get counted as a conversion when no real appointment exists — which would corrupt both GA4 reporting and, more seriously, the Google Ads conversion signal used for bidding.

### What GTM listens to

- **One** Custom Event trigger: `event equals booking_step_complete` → fires a single parameterized GA4 Event tag, mapping `step_number` and `step_name` as event parameters. This is one trigger serving all three steps, not three separate triggers — the step number is data, not trigger logic.
- **A second, separate** Custom Event trigger: `event equals booking_confirmed` → fires a GA4 Event tag marked as a **Key Event** in GA4 Admin (Admin > Events > Mark as key event).
- Both triggers are validated in **GTM Preview mode** before publishing — I'd step through the form live in Preview, confirming each push appears in the "dataLayer" tab of the Preview debugger with the correct parameter values, before touching the production container.

### GA4 custom dimensions

`step_number` and `step_name` must be registered as **event-scoped custom dimensions** in GA4 Admin (Admin > Custom definitions > Create custom dimension, scope: Event). Without this registration step, neither parameter is selectable as a breakdown dimension anywhere in GA4's UI, including Funnel Exploration — this is the step people most commonly forget, and the funnel report will look empty or unusable until it's done.

### GA4 Funnel Exploration configuration

1. Build a funnel with four **open** steps (not closed): `booking_step_complete` where `step_number = 1` → `booking_step_complete` where `step_number = 2` → `booking_step_complete` where `step_number = 3` → `booking_confirmed`. "Open funnel" is used deliberately so partial completions still count — the entire point of this report is to see where people abandon, not just who finished.
2. Leave **"make step immediate"** off, allowing realistic time gaps between steps — people get interrupted mid-form on mobile, and forcing immediate sequencing would undercount legitimate completions.
3. Add a **breakdown dimension of `clinic_location`** on step 1, to distinguish a location-specific problem (e.g. one clinic showing no near-term availability, causing step-2 abandonment for that location specifically) from a global UX issue affecting all clinics equally.
4. Use the built-in **"time to complete step"** metric to catch a slow or confusing step — e.g. an unusually long median time between step 2 and step 3 usually points at a confusing review screen rather than genuine hesitation.

### Analyzing drop-off

Drop-off is read directly off the funnel's step-over-step percentages: the report shows what fraction of users who completed step *N* also completed step *N+1*. Any step showing an unusually low pass-through rate (say step 2 → step 3 dropping to 40% against a 75% baseline) is the one to investigate first, and the `clinic_location` breakdown tells you whether that drop-off is concentrated at specific clinics or global.

**Who writes the dataLayer push:** the front-end developer, not GTM, and not the marketing team. My brief to the dev team for step 2 specifically: *"In the handler that validates and advances from step 2 to step 3, immediately after validation passes and before you render step 3's DOM, add `window.dataLayer.push({...})` with this exact JSON shape. Do not include the phone number field — push `contact_info_entered: true` instead. Test it yourself first by opening the browser console, completing step 2, and confirming the object appears in `window.dataLayer` — then I'll validate independently in GTM Preview before anything goes to the live container."*

---

## Part 3: Conversion Action to Import into Google Ads

**Import `booking_confirmed`.**

### Why not the others

- **`call_now_click` / `whatsapp_click`** are soft-intent signals — a click, not a commitment. Importing either as the primary conversion would train Smart Bidding to spend budget chasing people who click a button, not people who actually book an appointment. Given the account's current 2.1% baseline conversion problem, the campaign needs to learn from the signal closest to actual business value, not a proxy several steps upstream of it.
- **`booking_step_complete` (any step, including step 3)** is a submission-intent event, not a confirmed outcome. Step 3 specifically can fire even when the backend booking call subsequently fails — importing it would let failed bookings train the bidding algorithm as if they were real conversions, quietly degrading Smart Bidding's model over time in a way that's hard to detect after the fact.
- **`patient_guide_form_submit`** is a legitimate secondary/nurture signal, but it's several steps removed from an actual appointment and would dilute optimization toward a much lower-value action.

### Implications for Smart Bidding

`booking_confirmed` fires only after a verified backend success, so it's a clean signal Smart Bidding (Target CPA / Maximize Conversions / target ROAS) can optimize against without contamination from abandoned or failed submissions. Once volume stabilizes, `patient_guide_form_submit` can be added as a **secondary, non-bid-affecting conversion** in Google Ads (imported as "Observation" rather than counted toward the primary goal) — this gives the algorithm additional training signal on earlier-funnel intent without diluting optimization away from the true down-funnel outcome, which remains `booking_confirmed`.
