# Lead capture: first-party forms posting server-side to GoHighLevel

Both forms (Inquiry, Plan request) are native site forms posting to a site-owned Next.js server endpoint, which writes to GoHighLevel via `POST /contacts/` on API v2 with a Private Integration Token. GHL-embedded forms were rejected **knowing the cost**: GHL's attribution machinery (First/Latest attribution, UTMs, click IDs) fires only on GHL-native surfaces, so the site attributes leads itself — first-touch UTM + click IDs captured in a 90-day first-party cookie and written to the same contact fields the studio's Meta integration already uses. Design control over the forms and the swappable-CRM seam (ADR 0002) outweighed the vendor's attribution dashboard, which the studio was not positioned to exploit anyway.

- **Store-first delivery buffer.** Every submission is written to a lightweight site-owned store, then forwarded to GHL and marked delivered; all rows purge after 90 days. GHL is the system of record — the buffer exists so a GHL outage cannot lose a lead. On a failed write: immediate email alert to the studio carrying the full lead details, plus automated retry of undelivered rows. *This refines ADR 0002's "the site keeps no lead store": the first-party principle stands; the store is a transient delivery buffer, never a second CRM.*
- **Everything after delivery lives in GHL.** Notification (workflow: new website-source contact → email + push) and Opportunity creation are GHL-side configuration, not site code. The site sends nothing on success and holds no pipeline or stage IDs.
- **Spam control** is invisible: honeypot + time-trap + server-side rate limiting + Cloudflare Turnstile. No visible CAPTCHA.
- **Consent is required, transactional, and soft-voiced.** Both forms carry a required checkbox consenting to calls and texts *about the prospect's own request* (positive-affect wording, not legalese) — required consent is compliant precisely because it is scoped to responding to the inquiry, not marketing.
- **Plan attribution** rides the payload: a visible, pre-selected Plan dropdown (seeded from the referring Plan page's URL parameter, degrading to an ordinary dropdown) written server-side to a contact-level single-option custom field plus a distinct website Contact Source.

## Considered Options

- **GHL-embedded forms** — rejected (above, and ADR 0002): native attribution wasn't worth surrendering form design and the lead pipe.
- **FormSpree or similar relay** — rejected: a second vendor in front of the CRM with neither GHL's attribution nor first-party control.
- **Store-on-failure only** — rejected: retains less data but a crash between the failed GHL call and the fallback write loses the lead; store-first is simpler and cannot lose one.
- **GHL Inbound Webhook trigger as endpoint** — rejected: documented as unauthenticated; unusable as a public endpoint.

Decided in wayfinder ticket [Inquiry destination and handling (#11)](https://github.com/canyon-onehouse/compass-sales/issues/11).
