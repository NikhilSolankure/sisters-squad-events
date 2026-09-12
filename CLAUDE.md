# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project summary

This repo is the website and booking system for **Sisters Squad**, an event planning company in Kolhapur, India, live at **sisterssquadevent.events**. It is a set of static, framework-free HTML pages (public site, ticket booking, an internal phone-booking tool, an admin dashboard, and a QR entry scanner) backed by a Google Apps Script web app that reads/writes a Google Sheet and generates ticket-pass images in Google Drive. The system was built for, and proven at, one real event — बाईपण भारी नाईट 3.0 (Baipan Bhaari Night 3.0), 6 September 2026 — and is now in a light post-event cleanup phase rather than active rebuild.

## Hosting and deployment

- Static HTML/CSS/JS hosted on **GitHub Pages**, deployed from `main`, root folder. No build step — every `.html` file is committed as-is and served directly.
- A commit to `main` auto-deploys within 1–2 minutes via GitHub Pages/Actions. There is nothing to run locally to "build" the site — editing a file and pushing is the entire deploy process.
- Custom domain **sisterssquadevent.events** is configured via DNS (A records + CNAME) at Namecheap; `CNAME` file in this repo holds the domain for GitHub Pages. HTTPS is GitHub Pages' free managed certificate.
- Backend is a **Google Apps Script** project deployed as a web app (`doGet`/`doPost`), using a Google Sheet as the database and Google Drive to store generated ticket-pass images. **The Apps Script project is not in this repo** — it lives entirely in Google's environment and is not version-controlled. Any change to booking/approval/scanning logic must be made in the Apps Script editor directly; this repo only has the client-side code that calls it.
- All pages call the same Apps Script web app URL, hardcoded independently in each file's `<script>` config block (`booking.html`, `team.html`, `admin.html`, `scanner.html`). If the Apps Script is ever redeployed under a new URL, it must be updated in all four places — there is no shared config file.
- Google Analytics 4 (`gtag.js`, measurement ID `G-G846JJ5PXQ`) is embedded on the public-facing pages only (`index.html`, `booking.html`). It is deliberately absent from the password-gated internal tools (`team.html`, `admin.html`, `scanner.html`).

## Repository layout

| File | Purpose |
|---|---|
| `index.html` | Public landing page. Hero, featured/current event card, services grid, "why us" values, and a WhatsApp-based enquiry form (`sendEnquiry()` builds a `wa.me` link client-side — no backend involved). Links to `/booking` and an in-page `#enquiry` anchor. |
| `booking.html` | Customer-facing ticket booking flow: enter name/phone/email, pick adult/kid ticket counts (stepper UI), submit to the Apps Script API to reserve a booking and get a reference number, then a payment screen showing a UPI ID/phone number to pay manually, and a "send screenshot on WhatsApp" button for manual verification. Ends on a "booking received, pending verification" screen. |
| `team.html` | Password-gated internal tool for staff to create bookings on behalf of customers calling in by phone. Same ticket-stepper UI as `booking.html` but posts with `internal:true`; on success shows the reference and a pre-filled WhatsApp message (including the event WhatsApp group link) to send the pass directly, skipping the payment-pending step since cash/UPI is collected on the call. |
| `admin.html` | Password-gated team dashboard: live stats (revenue, tickets sold, adults/kids, pending, to-send, checked-in), tabs for Pending / To Send / All bookings, search, and per-booking actions (Approve, Reject, Send pass on WhatsApp, Mark sent, Call). Auto-refreshes every 30s while the tab is visible. |
| `scanner.html` | Camera-based QR check-in scanner (uses the `html5-qrcode` library from a CDN) with a live "guests arrived / total" counter, manual ticket-ID entry fallback (`BPB3-###`), and audio+vibration feedback for valid/invalid scans. Polls scan stats every 60s. |
| `CNAME` | GitHub Pages custom-domain file, contains `sisterssquadevent.events`. |
| `social-preview.png` | Open Graph / Twitter card image referenced by `index.html` and `booking.html`. |

There is no `Code.gs`, `package.json`, build config, linter, or test suite in this repo — there is nothing to install, build, lint, or test locally. The only "commands" are git commands to commit and push to `main`.

## Safe restore point

Tag **`v1.0-bpb3-3.0`** marks the exact working state that ran the live event on 6 September 2026 (40+ real bookings, live door scanning, all four flows exercised end-to-end). Treat this tag as a permanent, known-good checkpoint. **Always mention that this tag exists before making or describing any risky/destructive change** (e.g. `git reset`, force-push, large rewrites of the booking/scanner logic), so the user knows they can restore to it if something breaks.

## Current phase vs. future phase — do not conflate these

**Current phase (active now): light cleanup, not a rebuild.**
- Remove the event-specific booking/team/scanner flow from public navigation now that the event has passed (the event has already ended, so `index.html`'s "currently booking" event card and `/booking` link are stale and due for rework).
- Keep `admin.html` reachable (even if unlinked from nav) so the team can still review event data.
- Evolve `index.html` into a proper company site: add a real photo/video gallery from the event, a "past events" credibility section, etc.
- Do **not** restructure the stack, introduce a build system/framework, or migrate off Google Sheets/Apps Script during this phase unless explicitly told to.

**Future phase (context only — do not act on this yet):**
- Eventual migration to a proper stack: Next.js (or similar), a real database (likely Supabase/Postgres) instead of Google Sheets, and a real payment gateway (Razorpay) instead of manual UPI.
- This is expected to happen in a **separate, fresh repository**, once Sisters Squad has run enough events to justify multi-event support (concurrent events, per-event ticket types/pricing, etc.). Google Sheets and Apps Script get retired only once that new system is proven.
- Do not start any of this unless the user explicitly asks for it.

## Conventions to follow when editing these files

- Each HTML file is fully self-contained: inline `<style>` in `<head>`, inline `<script>` at the end of `<body>`, no shared CSS/JS files, no imports/bundlers. Keep new work in the same self-contained style rather than introducing shared modules.
- Shared visual language across pages: dark maroon/burgundy gradient backgrounds (`#0a0307`/`#2b0a12`/`#35081b` family), gold accent (`#e6b422` / `#c9a227` / `#E8B54A`), `Playfair Display` for headings/serif accents and `Inter` for body text (both loaded from Google Fonts). Match this palette and font pairing in any new UI.
- Config values (Apps Script API URL, WhatsApp number, ticket prices, max ticket counts) are declared as `const` at the top of each page's `<script>` block, clearly marked with a `// ===== CONFIG =====` comment banner. Follow this pattern for any new tunable value, and remember these are duplicated per file, not shared.
- Password-gated pages (`team.html`, `admin.html`) use an identical pattern: a `.gate` div with a password `<input>`, a `login()` function that validates the password against the Apps Script API (`?action=stats&pw=...`) and, on success, caches it in `sessionStorage` (`bpb3pw`) and swaps `.gate` for `#app`.
- JS is plain ES5/ES6 with no framework — direct DOM manipulation via `document.getElementById`, inline `onclick=""` handlers, and `fetch().then()` chains (no async/await in most places). Keep additions consistent with this style rather than introducing a different pattern in one file.
- UI text mixes English with Marathi/Devanagari for event-specific copy (e.g. "बाईपण भारी नाईट 3.0"); currency is always formatted with `.toLocaleString('en-IN')` and a `₹` prefix.

## Known limitations

- **Manual UPI payments, not a gateway**: Payments go through a personal UPI ID/phone number (not Razorpay or similar), because NPCI restrictions on P2P UPI Intent links were discovered during testing and ruled out an automated payment gateway for this phase. Customers pay manually, then send a WhatsApp screenshot for manual verification by the team. This is an accepted, known limitation, not a bug to fix.
- **No build step**: every HTML file is served exactly as committed; there's no minification, bundling, or transpilation, so browser-compatible syntax must be written directly.
- **Apps Script is not version-controlled**: the entire backend (booking creation, approval/rejection, pass image generation, scan check-in, stats) lives in Google's Apps Script editor, outside this repo. Changes there aren't visible via `git log`/`git diff` here.
- **Single shared password for internal pages**: `team.html` and `admin.html` both gate access with one team password, checked server-side via the Apps Script API — there's no per-user auth, and the password is not stored anywhere in this repo (it's validated remotely).
- **Duplicated config across files**: the Apps Script URL, WhatsApp number, and pricing constants are copy-pasted into each HTML file's script block rather than centralized, so multi-file edits are needed when any of these change.
