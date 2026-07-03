# OrthoNow — Developer Assignment Submission

**Candidate:** [Apna naam yahan likho]

## Structure

- `task1-gtm-schema.md` — Full GTM event schema, booking funnel dataLayer JSON, GA4 Funnel Exploration setup, Google Ads conversion recommendation.
- `task2-landing-page.html` — Self-contained landing page (HTML/CSS/JS, no frameworks). Open directly in a browser, or view live via GitHub Pages.
- `task2-pagespeed.png` — PageSpeed Insights Mobile score screenshot (add after testing — see below).
- `task3-integration.md` — Written integration architecture (HubSpot + WhatsApp + Google Ads).

## How to test the landing page

1. **Locally:** double-click `task2-landing-page.html` to open in any browser.
2. **Live (needed for PageSpeed test):** enable GitHub Pages on this repo (Settings → Pages → Source → `main` branch), then visit:
   `https://<your-username>.github.io/orthonow-assignment/task2-landing-page.html`
3. Open DevTools Console (F12 → Console tab).
4. Type `window.dataLayer` and press Enter — you'll see an empty array.
5. Fill the form (name, and a 10-digit phone starting with 6–9) and submit.
6. Type `window.dataLayer` again and press Enter — you'll see the `consultation_form_submitted` event pushed in in the array.
