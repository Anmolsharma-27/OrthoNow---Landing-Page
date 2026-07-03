# OrthoNow — Landing Page

A campaign landing page and analytics/integration assignment for **OrthoNow**, an orthopaedic consultation booking service targeting working professionals in Bengaluru. The project covers a conversion-focused lead capture page, a GTM/GA4 event tracking schema, and a backend integration architecture for routing leads to HubSpot, WhatsApp, and Google Ads.

## Overview

This repository contains the deliverables for a developer assignment, structured around three tasks:

| Task | File | Description |
|---|---|---|
| Task 1 | [`task_1.md`](./task_1.md) | GTM event schema — event names, trigger types, key parameters, and corresponding GA4 reports |
| Task 2 | [`index.html`](./index.html) | The landing page itself — a single-file, responsive lead capture form with GTM `dataLayer` tracking |
| Task 3 | [`task_3.md`](./task_3.md) | End-to-end backend architecture for routing form submissions to HubSpot, WhatsApp (Karix), and Google Ads |
| — | [`lighthouse-report.html`](./lighthouse-report.html) | Lighthouse performance/accessibility/SEO audit of the landing page |
| — | [`Developer_Assignment_Anmol_Sharma.docx`](./Developer_Assignment_Anmol_Sharma.docx) | Original assignment brief/writeup |

## Task 2 — Landing Page (`index.html`)

A self-contained, dependency-free HTML page with inline CSS and JavaScript.

**Key features:**
- Campaign-targeted hero section ("For Bengaluru professionals, age 28–50")
- Trust-building panel: patient rating, specialist count, clinic locations, appointment availability
- Minimal-friction consultation form (name + 10-digit phone number only)
- Client-side validation (required name, 10-digit numeric phone pattern)
- `dataLayer.push()` fired on successful submission for GTM/GA4 tracking, capturing:
  - `event: 'consultation_form_submitted'`
  - `form_id`, `city`, `name_length`, `phone_last4`
- Accessible markup (`aria-label`, `aria-live` for the thank-you state)
- Fully responsive layout (single column on mobile, two-column grid on desktop via CSS Grid + media query)
- No external dependencies — plain HTML/CSS/JS, viewable by opening the file directly in a browser

**To view locally:**
```bash
open index.html
# or serve it
python3 -m http.server 8000
```

## Task 1 — GTM Event Schema (`task_1.md`)

Defines the tracking plan for the funnel: engagement events (`call_button_click`, `whatsapp_widget_click`, `clinic_page_view`, `blog_scroll_depth`) through to conversion events (`patient_guide_download`, `booking_step1_complete`, `booking_step2_complete`, `booking_confirmed`), each mapped to its GA4 report and audience.

## Task 3 — Backend Architecture (`task_3.md`)

Proposes a direct server-side integration (over Zapier/Make) for handling form submissions:

- **Fan-out architecture**: backend calls HubSpot, Karix WhatsApp, and Google Ads independently so one failure doesn't block the others
- **Phone-based contact deduplication** in HubSpot (since the form doesn't collect email)
- **Shared phone number edge case** handling (e.g., family members sharing a number) to avoid overwriting existing contact records
- **Queue-based WhatsApp delivery** with automatic retries and a dead-letter queue to meet a 2-minute SLA
- Monitoring and alerting thresholds for send latency, queue depth, and failure rate

## Tech Stack

- HTML5 / CSS3 (vanilla, no framework)
- Vanilla JavaScript
- Google Tag Manager / GA4 (tracking plan only — GTM container not included)
- Proposed backend: serverless functions + HubSpot API + Karix WhatsApp API + Google Ads Conversion API

## Repository Structure

```
.
├── index.html                              # Landing page (Task 2)
├── task_1.md                               # GTM event schema (Task 1)
├── task_3.md                               # Backend architecture (Task 3)
├── lighthouse-report.html                  # Performance audit
└── Developer_Assignment_Anmol_Sharma.docx  # Assignment brief
```

## Author

**Anmol Sharma** — [Anmolsharma-27](https://github.com/Anmolsharma-27)
