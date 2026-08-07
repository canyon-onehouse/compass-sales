# Day-one seams: commerce and portal arrive without a rewrite

Launch scope is marketing and lead capture only, but house-plan commerce and a client portal are expected later. We decided the seams day one must preserve — and, as deliberately, what it must not build — so either arrives without rewriting the site. The canonical prototype contains no commerce or auth UI, so these seams are purely architectural.

- **Portal**: ships later as a separate app on its own subdomain. The marketing site stays permanently auth-free — no accounts, sessions, or auth-dependent rendering or middleware — keeping it static and cacheable. Subdomain naming lands with the domain/SEO work.
- **Work-item identity**: one canonical record per item, addressed by **(type, slug)**, with Project and Plan as the types. Slugs are globally unique across types, never reused (even after archival), and immutable once published; display titles change freely. A rename is a deliberate migration with redirects, not a content edit. A revised plan is a **new record with a new slug**; the superseded record is archived, never deleted, because issued licenses and delivered files keep referencing it. A Project that results from a Plan is a bespoke customization for one client engagement — a new record, never a reusable variant.
- **References run inward only.** Systems outside the content store — payment objects, order records, license grants, file paths — reference content by (type, slug) and never copy its fields. The content store holds no outbound references; the mapping from a Plan to its purchasable item(s) lives on the commerce side, whoever provides it.
- **No reserved commerce fields.** `price` is owned by whatever processes payment and never belongs on Plan. A `sku` field would assert one purchasable item per Plan — cardinality we don't know — so none exists. Adding commerce data later is an additive, one-place change by construction.
- **Plan-request is permanent architecture**, not scaffolding. After commerce launches it becomes the talk-first / semi-custom-upsell lane beside a direct-purchase CTA; labels and copy may evolve, the flow and structure stay.
- **Lead capture is first-party.** Our forms post to a site-owned endpoint; the site keeps no lead store; a Plan request always carries its Plan's slug. What sits behind the endpoint (GoHighLevel directly, an interim relay such as Formspree) is a swappable delivery detail.

Together these keep both commerce futures open — a first-party shop as a separate subdomain app, or a third-party provider such as Shopify — without prejudging either.

**Revisit before the first license document is issued**: whatever names a plan on that document must already be immutable. No plan number or internal code exists in how the firm operates today and none is reserved; if one becomes necessary it is introduced as an additive field before that first document, never retrofitted after.

## Considered Options

- **Reserved nullable commerce fields** (`price`, `sku` on Plan now) — rejected. Dead weight that invites accidental rendering and presumes a schema and cardinality not yet designed.
- **Portal inside this Next.js app** (future route group) — rejected. Drags auth, sessions, and middleware into a site whose value is being static, fast, and simple to operate.
- **Embedded CRM form widgets** — rejected. Styling, attribution, and the lead stream itself would live inside the vendor's widget; swapping CRMs would mean redoing the forms.
- **Dual identity now** (slug + internal ID) — rejected. No such identifier exists in the firm's operations; if licensing demands one it is additive later, before the first license document.

Decided in wayfinder ticket [What launch must not foreclose (#7)](https://github.com/canyon-onehouse/compass-sales/issues/7).
