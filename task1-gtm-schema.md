# Task 01 — GTM Event Schema (OrthoNow)

This document defines the complete GTM event tracking schema for the OrthoNow website, ahead of any paid campaign launch.

---

## 1. Full Event Schema

| Event Name | Trigger Type | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `clinic_specialty_selected` | Custom Event (dataLayer push) | `clinic_location`, `specialty`, `step_number: 1` | Funnel Exploration (Step 1), Booking Funnel Audience |
| `booking_details_entered` | Custom Event (dataLayer push) | `clinic_location`, `preferred_date`, `step_number: 2` | Funnel Exploration (Step 2), Remarketing Audience (drop-offs) |
| `booking_confirmed` | Custom Event (dataLayer push) | `clinic_location`, `specialty`, `booking_id`, `step_number: 3` | Key Event / Conversion, Google Ads import |
| `call_now_click` | Click Trigger (Just Links / Click ID) | `page_location`, `clinic_name` (if on clinic page), `button_position` (header/footer/sticky) | Engagement report, Call-conversion audience |
| `whatsapp_chat_click` | Click Trigger (Click Classes = whatsapp widget) | `page_location`, `page_type` (home/clinic/landing) | Engagement report, Assisted-conversion path |
| `patient_guide_form_submit` | Form Submission Trigger | `form_id`, `lead_source`, `page_location` | Lead-gen Key Event, Nurture Audience |
| `patient_guide_download` | Custom Event (fires after form + PDF link resolves) | `file_name`, `file_extension`, `lead_source` | File Download report |
| `clinic_page_view` | Page View Trigger (URL contains `/clinics/`) | `clinic_name`, `city`, `page_location` | Location-level engagement, Local campaign attribution |
| `blog_scroll_depth` | Scroll Depth Trigger (25/50/75/90%) | `percent_scrolled`, `article_title`, `article_category` | Content engagement report, Blog remarketing audience |

**Notes:**
- All "Custom Event" triggers rely on the front-end developer pushing a `dataLayer.push()` at the right moment — GTM listens for these, it does not generate them on its own for JS-driven interactions (see funnel section below).
- `call_now_click` and `whatsapp_chat_click` can use GTM's built-in **Click Trigger** since they are standard `<a>` tags — no custom dataLayer code needed from the dev team here.
- `clinic_page_view` can use a simple **Page View trigger with a URL condition**, since it's a static page load, not a JS interaction.

---

## 2. Booking Form Funnel Tracking (3 Steps)

**The core problem:** OrthoNow's booking form is a single-page JS component with no page reloads between steps. GTM cannot "see" step transitions on its own — the front-end developer must fire a `dataLayer.push()` at the exact moment each step is completed. GTM only listens; it does not detect DOM/state changes in a multi-step form automatically.

### Step 1 — Location + Specialty selected

Trigger: GTM Custom Event trigger listening for `event: booking_step_complete` AND `step_number: 1`

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Joint Care"
}
```

### Step 2 — Contact details entered

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Koramangala",
  "preferred_date": "2026-07-10"
}
```

### Step 3 — Booking confirmed

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Joint Care",
  "booking_id": "ORTH-88213"
}
```

### How this gets built (dev handoff)

- **Who writes this code?** The front-end developer, not GTM. I (as the GTM implementer) hand the dev team the exact JSON keys/values above as a spec — they insert `window.dataLayer.push({...})` inside the `onClick`/`onSubmit` handler for each step's "Next" button in the form's JS.
- **GTM's job:** create one Custom Event trigger per step (matching on `event` = `booking_step_complete` and `step_number` = 1/2/3), then a corresponding GA4 Event tag for each, mapping `step_name`, `clinic_location`, etc. as event parameters.
- **Briefing the dev for Step 2 specifically:** "On the button that moves the user from the contact-details screen to the confirmation screen, right before you transition state/screen, add: `window.dataLayer.push({event: 'booking_step_complete', step_number: 2, step_name: 'contact_details_entered', clinic_location: <value from step 1's state>, preferred_date: <value from the date field>})`. This must fire only once per step completion, not on every keystroke."

### Surfacing drop-off in GA4

1. Register all three `booking_step_complete` events as **Key Events** in GA4 (Admin → Events → Mark as key event), or as regular custom events registered via GTM's GA4 Event tag.
2. Go to **Explore → Funnel Exploration**.
3. Build an open funnel with 3 steps:
   - Step 1: `booking_step_complete` where `step_number = 1`
   - Step 2: `booking_step_complete` where `step_number = 2`
   - Step 3: `booking_step_complete` where `step_number = 3`
4. Enable **"Show elapsed time"** to see time-to-drop and set the funnel to **"Open funnel"** so partial completions are still counted.
5. The funnel visualization directly shows the % drop-off between each step, which segment (clinic_location) drops off most, and can be broken down by device/channel using a secondary dimension.

---

## 3. Google Ads Conversion Import

**Recommended event to import: `booking_confirmed` (Step 3)**

**Why this one over the others:**
- It is the only event that represents an actual completed appointment booking — a true bottom-of-funnel conversion, not just intent (like `call_now_click` or `whatsapp_chat_click`, which only indicate a click, not a confirmed action).
- `patient_guide_form_submit` is a softer, top-of-funnel lead (someone downloading a guide, not booking), so importing it as the primary conversion would cause Google Ads' bidding algorithm to optimize toward low-intent users.
- Google Ads Smart Bidding performs best when optimized toward the single highest-value, most reliable signal. Importing `booking_confirmed` ensures budget is directed toward campaigns/keywords that actually drive real bookings, not just form starts.
- Secondary/observation conversions (`patient_guide_form_submit`, `call_now_click`) can still be imported as **secondary conversion actions** for reporting, without them counting toward primary bidding optimization.
