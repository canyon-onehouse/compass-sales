# Vercel + Next.js constraints for image-heavy sites

Research for [issue #9](https://github.com/canyon-onehouse/compass-sales/issues/9). All claims cite the primary source that owns them. Researched 2026-08-07 against vercel.com/docs (pricing pages last updated June–August 2026) and nextjs.org/docs (which self-identify as Next.js 16.3.0 — several defaults changed from the v14/v15 era and are flagged below). Plan tier is noted on every number.

**Site profile assumed throughout:** static-ish Next.js marketing site, ~20 pages (12 image-heavy portfolio pages), ~490 MB of source imagery, deployed and operated by a non-specialist owner.

---

## Headline answers

| Question | Answer |
| --- | --- |
| Can this site run on Hobby? | **No.** Hobby is restricted to non-commercial personal use; a design studio's marketing site is commercial. Also, 490 MB exceeds Hobby's 100 MB CLI source-upload cap. |
| Does 490 MB of imagery fit in a deployment? | **Yes on Pro** (1 GB CLI source cap, 15,000-file cap). Pre-optimizing masters is still strongly advised. |
| How is image optimization billed? | Per **transformation** plus **cache reads/writes in 8 KB units** — no longer per source image (that model is legacy, Enterprise-only). |
| Expected monthly cost | **≈ $20/month** (one Pro seat). At small-studio traffic, all usage fits inside Pro's included quotas and $20 usage credit; realistic overage is $0. |
| Does ISR matter here? | Barely. A static-ish site's ISR costs are fractions of a cent; the practical notes are stale-while-revalidate semantics and that the ISR cache resets on each deployment. |

---

## 1. Image Optimization on Vercel

### 1.1 Current billing model

Image Optimization is billed on three metrics — **image transformations, image cache reads, and image cache writes** ([vercel.com/docs/image-optimization/limits-and-pricing](https://vercel.com/docs/image-optimization/limits-and-pricing)). The old **"source images"** billing (per unique `src` value per billing period, $5.00 per 1,000 beyond 5,000 on Pro) is legacy: it is "only available to Enterprise teams created before February 18th, 2025" ([vercel.com/docs/image-optimization/legacy-pricing](https://vercel.com/docs/image-optimization/legacy-pricing)). New Hobby/Pro accounts are on the transformation model.

| Metric | Hobby included | On-demand rate (regional range; US/iad1 at low end) |
| --- | --- | --- |
| Image transformations | 5,000/month | $0.05–$0.0812 per 1,000 |
| Image cache reads | 300,000/month | $0.40–$0.64 per 1M (8 KB units) |
| Image cache writes | 100,000/month | $4.00–$6.40 per 1M (8 KB units) |

Sources: quota/rate table on [vercel.com/docs/image-optimization/limits-and-pricing](https://vercel.com/docs/image-optimization/limits-and-pricing) (same table on [vercel.com/docs/pricing](https://vercel.com/docs/pricing)); regional ranges on [vercel.com/docs/pricing/regional-pricing](https://vercel.com/docs/pricing/regional-pricing), US base rates on [vercel.com/docs/pricing/regional-pricing/iad1](https://vercel.com/docs/pricing/regional-pricing/iad1).

**Pro has no per-metric image quota.** The included amounts above are labeled Hobby-only; on Pro, image usage draws down the plan's **$20/month included usage credit** ("across all infrastructure resources"), then bills on-demand at the rates above ([vercel.com/docs/plans/pro-plan](https://vercel.com/docs/plans/pro-plan), [vercel.com/docs/pricing](https://vercel.com/docs/pricing)).

**Hobby overage behavior (images):** once Hobby limits are exceeded, *new* images fail to optimize with a 402 runtime error (the `alt` text renders instead); already-cached optimized images keep working; you are **not** charged ([vercel.com/docs/image-optimization/limits-and-pricing](https://vercel.com/docs/image-optimization/limits-and-pricing), Billing → Hobby).

### 1.2 What counts as one transformation

- A **transformation is billed on every cache MISS and STALE**. The cache key is: project ID + `q` (quality) + `w` (width) + `url` (absolute URL for remote images; **content hash of the file** for local images) + the normalized `Accept` header (which selects output format). So one unique source × width × quality × format combination = one cache entry ([vercel.com/docs/image-optimization](https://vercel.com/docs/image-optimization), cache-key sections; [limits-and-pricing](https://vercel.com/docs/image-optimization/limits-and-pricing)).
- **Cache writes** are billed in 8 KB units on every MISS/STALE (storing the transformed image in the global cache). **Cache reads** are billed in 8 KB units only when the image must be fetched from the shared global cache — a recently-accessed image is cached in-region and incurs no read cost ([vercel.com/docs/image-optimization/limits-and-pricing](https://vercel.com/docs/image-optimization/limits-and-pricing)).
- Delivering the optimized bytes additionally counts toward **Fast Data Transfer and Edge Requests** like any other response ([vercel.com/docs/image-optimization](https://vercel.com/docs/image-optimization)).

### 1.3 Limits and format support (all plans)

From [vercel.com/docs/image-optimization/limits-and-pricing](https://vercel.com/docs/image-optimization/limits-and-pricing) (Limits section):

- Maximum **transformed** image size: **10 MB** (the docs state the cap against the transformed output, via the cacheable-responses limit, not the source file).
- Maximum source dimensions: **8192 × 8192 px**.
- Source must be `image/jpeg`, `image/png`, `image/webp`, or `image/avif` to be optimized; **any other format (GIF, SVG, …) is served as-is**.
- Output formats: WebP and AVIF, negotiated via the `Accept` header; which ones are generated is controlled by Next.js `images.formats` ([vercel.com/docs/image-optimization](https://vercel.com/docs/image-optimization); [managing-image-optimization-costs](https://vercel.com/docs/image-optimization/managing-image-optimization-costs)).
- Vercel recommends `unoptimized` for animated GIFs, SVGs, and images under ~10 KB ([vercel.com/docs/image-optimization](https://vercel.com/docs/image-optimization), When to use Image Optimization).

### 1.4 Cache TTL and invalidation

From [vercel.com/docs/image-optimization](https://vercel.com/docs/image-optimization):

- **Local images** (in the deployment): cached on the Vercel CDN for **up to 31 days**, keyed by content hash.
- **Remote images**: TTL is the larger of the upstream `Cache-Control: max-age` and `minimumCacheTTL` (Vercel's doc states a 3600 s default; the current Next.js 16 doc states its own `minimumCacheTTL` default is now **14400 s / 4 hours** — [nextjs.org image reference](https://nextjs.org/docs/app/api-reference/components/image#minimumcachettl)).
- **Redeploying does not invalidate the image cache.** For local images, invalidation happens by content change (new hash) + redeploy, or by manually purging the CDN cache ([vercel.com/docs/image-optimization](https://vercel.com/docs/image-optimization)). Practical consequence: transformations for unchanged imagery are essentially a one-time cost, re-incurred only on TTL expiry.
- Cost lever: long `Cache-Control`/`minimumCacheTTL` values reduce transformations and cache writes; **statically imported images get an immutable ~1-year Cache-Control automatically** ([vercel.com/docs/image-optimization/managing-image-optimization-costs](https://vercel.com/docs/image-optimization/managing-image-optimization-costs); [nextjs.org](https://nextjs.org/docs/app/api-reference/components/image#minimumcachettl)).

## 2. next/image facts that drive transformation counts

All from [nextjs.org/docs/app/api-reference/components/image](https://nextjs.org/docs/app/api-reference/components/image) unless noted. Docs are for Next.js 16.3.0.

- **`remotePatterns`** — required allowlist for remote images ("allow images from specific external paths and block all others"); unmatched requests get 400. Supports `*` (one segment/subdomain) and `**` (leading subdomains / trailing path segments); wildcards don't work mid-pattern. The old `domains` config is deprecated since v14. Irrelevant if all imagery ships in the repo.
- **`localPatterns`** — same idea for local paths, e.g. `{ pathname: '/assets/images/**', search: '' }`; anything else 400s. Useful hardening for a public-facing optimizer endpoint.
- **`deviceSizes` default** `[640, 750, 828, 1080, 1200, 1920, 2048, 3840]`; **`imageSizes` default** `[32, 48, 64, 96, 128, 256, 384]` — together the 15 candidate widths from which `srcset` entries are generated.
- **Variants per image:** without a `sizes` prop, Next.js generates a *limited* srcset — the docs' example emits exactly **two** candidate URLs (1x/2x); with `sizes`, it generates a *full* srcset drawn from the 15 widths. The browser downloads one candidate per render; each distinct width actually requested becomes one transformation. Total stored-variant multiplier = matching widths × configured formats × allowed qualities ([#sizes](https://nextjs.org/docs/app/api-reference/components/image#sizes), [#imagesizes](https://nextjs.org/docs/app/api-reference/components/image#imagesizes), [#formats](https://nextjs.org/docs/app/api-reference/components/image#formats)).
- **`formats` default is `['image/webp']`** — AVIF is opt-in and, per the docs, encodes ~50% slower for ~20% smaller files, and "Next.js will cache each format separately" (doubling stored variants/cache writes).
- **`qualities` default is `[75]`** and the field is enforced in v16 (non-listed qualities snap to the closest allowed value; direct API calls with unlisted values 400).
- **`unoptimized`** — per-image prop or global `images: { unoptimized: true }` (since 12.3.0): serves the source as-is, bypassing transformation billing entirely. SVG `src`s are unoptimized automatically.
- **`loader` / `loaderFile`** — swap in an external image CDN (Cloudinary, Imgix, Cloudflare, etc.) without changing `<Image>` call sites; ready-made loader examples on [nextjs.org/docs/app/api-reference/config/next-config-js/images](https://nextjs.org/docs/app/api-reference/config/next-config-js/images).
- **Static imports vs `public/`**: `public/` assets get `Cache-Control: public, max-age=0` because Next.js "cannot safely cache" them ([public-folder](https://nextjs.org/docs/app/api-reference/file-conventions/public-folder)); statically imported images are content-hashed and cached "forever" with `immutable`, and get automatic `width`/`height` (CLS prevention) and `blurDataURL` ([getting-started/images](https://nextjs.org/docs/app/getting-started/images)). For an image-heavy portfolio, static imports (or the documented dynamic-`import()` pattern) are the right default.

## 3. ISR

### 3.1 Semantics (Next.js)

From [nextjs.org/docs/app/guides/incremental-static-regeneration](https://nextjs.org/docs/app/guides/incremental-static-regeneration):

- **Time-based** (`export const revalidate = N`, or `fetch(url, { next: { revalidate: N } })`): stale-while-revalidate — after the interval, the next visitor still gets the cached (stale) page instantly while a fresh version regenerates in the background; failures keep serving the last good version. The lowest `revalidate` across a route's layouts/pages wins, and the value must be statically analyzable (`600`, not `60 * 10`).
- **On-demand**: `revalidatePath(path, type?)` and `revalidateTag(tag, profile)` mark entries for regeneration on next visit (not eagerly; eager regeneration currently exists only via Pages Router `res.revalidate`). Note the v16 change: `revalidateTag` now takes a second argument — `'max'` recommended for stale-while-revalidate semantics; the single-argument form is deprecated ([revalidateTag reference](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)).
- `generateStaticParams` prerenders known paths at build; with `dynamicParams: true` (default) unknown params generate on-demand and are cached; `false` 404s them ([dynamicParams](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/dynamicParams)).
- A fully static page (no `revalidate`, default `false`) never regenerates — "semantically equivalent to `revalidate: Infinity`" ([caching guide](https://nextjs.org/docs/app/guides/caching-without-cache-components#route-segment-config-revalidate)). For this site, that's the sensible mode for all 20 pages.

### 3.2 Behavior and billing (Vercel)

From [vercel.com/docs/incremental-static-regeneration](https://vercel.com/docs/incremental-static-regeneration) and [its limits-and-pricing page](https://vercel.com/docs/incremental-static-regeneration/limits-and-pricing):

- Two-level cache: free ephemeral CDN cache in every region + a **durable ISR cache in a single region** (the project's default Function region). Revalidation purges all regions within ~300 ms.
- **The ISR cache is per-deployment** — "each new deployment uses its own ISR cache and does not reuse the cache from a previous deployment" (this enables instant rollback). Expect regeneration after every deploy.
- **Billing units (current model): 8 KB read/write units against the durable cache**, not per-request counts. CDN cache hits are free; a write is only incurred when regenerated content actually changed. Prices: ISR reads $0.40–$0.64 per 1M units, ISR writes $4.00–$6.40 per 1M units (iad1 at the low end); priced by the Function region. Neither Hobby nor Pro docs list an explicit included ISR allotment — on Pro it consumes the $20 usage credit ([limits](https://vercel.com/docs/limits), [regional-pricing](https://vercel.com/docs/pricing/regional-pricing), [iad1](https://vercel.com/docs/pricing/regional-pricing/iad1)).
- Scale check: a 100 KB page = 13 write units = $0.000052 at iad1. ISR cost is noise for this site.

## 4. Network pricing (delivery of ~everything, including optimized images)

From [vercel.com/docs/limits](https://vercel.com/docs/limits), [vercel.com/docs/manage-cdn-usage](https://vercel.com/docs/manage-cdn-usage) (canonical page for CDN/networking pricing), [regional-pricing](https://vercel.com/docs/pricing/regional-pricing), and [iad1](https://vercel.com/docs/pricing/regional-pricing/iad1):

| Metric | What it measures | Hobby included | Pro included | Overage (US/iad1; regional range) |
| --- | --- | --- | --- | --- |
| Fast Data Transfer | CDN ↔ visitor bytes (full request+response) | 100 GB/mo | 1 TB/mo | $0.15/GB ($0.15–$0.35) |
| Fast Origin Transfer | CDN ↔ Functions/Blob/ISR bytes | up to 10 GB/mo | usage-based (credit) | $0.06/GB ($0.06–$0.43) |
| Edge Requests (CDN Requests) | every request the CDN serves — each image is one | 1,000,000/mo | 10,000,000/mo | $2.00/1M ($2.00–$3.20) |

Hobby cannot buy overage: exceeding a Hobby quota generally pauses the feature until 30 days pass, with no charge ([vercel.com/docs/plans/hobby](https://vercel.com/docs/plans/hobby)). There is also a minor "Edge Request CPU Duration" meter (free at ≤10 ms per request; $0.30/hr at iad1) that is negligible for a static site ([manage-cdn-usage](https://vercel.com/docs/manage-cdn-usage), [iad1](https://vercel.com/docs/pricing/regional-pricing/iad1)).

## 5. Hard limits relevant to 490 MB of source imagery

From [vercel.com/docs/limits](https://vercel.com/docs/limits) (last updated 2026-08-03):

- **CLI source-upload cap: 100 MB on Hobby, 1 GB on Pro** — "If the size of the source files exceeds this limit, the deployment will fail." **490 MB of imagery cannot be CLI-deployed on Hobby; it fits on Pro.** (The docs word this limit specifically for CLI deploys; they do not state whether the same cap applies to Git-triggered deploys — treat 100 MB as the safe planning number on Hobby.)
- **15,000 source files max per deployment** (CLI); no documented upper limit on *output* files beyond the 45-minute build timeout.
- The oft-cited 250 MB unzipped limit is a **function bundle** limit ([vercel.com/docs/functions/limitations](https://vercel.com/docs/functions/limitations)), not a static-asset limit; no per-file static-asset cap is documented.
- Builds: 45 min max; 100 deploys/day on Hobby, 6,000 on Pro.

## 6. Plan constraint: Hobby is not allowed for this site

Vercel's Fair Use Guidelines: "Hobby teams are restricted to non-commercial personal use only. All commercial usage of the platform requires either a Pro or Enterprise plan," where commercial usage includes "any Deployment that is used for the purpose of financial gain of anyone involved in any part of the production of the project," explicitly including advertising the sale of a product or service and being paid to create or host the site ([vercel.com/docs/limits/fair-use-guidelines#commercial-usage](https://vercel.com/docs/limits/fair-use-guidelines#commercial-usage)). A design studio's marketing site is squarely commercial. **Pro is required: $20/user/month, including the $20/month usage credit** ([vercel.com/docs/plans/pro-plan](https://vercel.com/docs/plans/pro-plan)).

---

## 7. What this means for this site

**Plan:** Pro, one seat — **$20/month**, non-negotiable (commercial-use terms; the 490 MB payload independently rules out Hobby's 100 MB CLI cap). Everything below is about whether anything gets added on top of that $20; at this site's scale, the answer is almost certainly no.

### Rough monthly cost scenarios (Pro, iad1 rates)

Assume ~490 MB ≈ **~400 source images** at ~1.2 MB average (rescale linearly if the real count differs), default next/image config (WebP only, quality 75), images statically imported.

**One-time-ish: first month / after imagery changes**

- Transformations: with proper `sizes` props, expect roughly 4–8 distinct widths actually requested per image over a month → ~1,600–3,200 transformations → **$0.08–$0.16**. (Local-image cache entries persist up to 31 days and survive redeploys, so steady-state months are far lower.)
- Cache writes: ~150 KB per optimized variant ≈ 19 units × ~3,200 variants ≈ 60K units → **~$0.24**.
- Adding AVIF (`formats: ['image/avif','image/webp']`) roughly doubles transformations and writes: still **under $1**.

**Recurring: traffic-driven delivery** (portfolio page ≈ 2.5 MB of optimized images, ~5 pages/visit ≈ 12.5 MB + ~75 image requests per visit)

| Traffic | Fast Data Transfer | Edge Requests | Overage cost |
| --- | --- | --- | --- |
| 2,000 visits/mo | ~25 GB | ~0.15M | $0 (within 1 TB / 10M) |
| 20,000 visits/mo | ~250 GB | ~1.5M | $0 |
| 100,000 visits/mo | ~1.25 TB | ~7.5M | ~$38 FDT overage — the first point real money appears |

ISR: not needed for a 20-page brochure site (build fully static; redeploy on content change). Even with `revalidate` enabled everywhere, costs would be fractions of a cent; the operative fact is behavioral — ISR cache resets per deployment, image cache does not.

**Bottom line: ~$20/month flat** unless traffic reaches the ~80K-visits/month range, and image-optimization spend is measured in cents, not dollars — the transformation quota fears that apply to huge dynamic catalogs do not apply to ~400 fixed images.

### Standard mitigations (in order of leverage)

1. **Pre-optimize and dedupe the masters before they enter the repo.** Resize to ≤ 3840 px on the long edge (the largest default `deviceSize`; anything bigger is wasted bytes and risks the 8192 px optimizer limit), re-encode to high-quality JPEG/WebP, drop unused/duplicate exports. 490 MB of masters typically compresses to well under 150 MB with no visible loss — faster deploys, comfortable margin under the 1 GB cap, and smaller cache-write bills. This is the one step that needs doing regardless of pipeline choice.
2. **Use static imports (or the documented dynamic-`import()` pattern), not bare `public/` paths**, so images get content-hashed immutable caching, automatic dimensions (no CLS), and blur placeholders ([nextjs.org](https://nextjs.org/docs/app/getting-started/images)).
3. **Set honest `sizes` props** on portfolio grids so browsers pick small variants — this is the main lever on Fast Data Transfer, the only metric that can ever cost real money here.
4. **Keep the default WebP-only `formats`** unless the ~20% AVIF savings matters; **mark SVG/GIF `unoptimized`** (SVG already is automatically).
5. **`unoptimized: true` globally** is the escape hatch if optimization ever misbehaves — zero transformation cost, at the price of shipping full-size images (which step 1 makes tolerable).
6. **External image CDN via `loaderFile`** (Cloudinary/Imgix/Cloudflare Images/etc.) is the standard outgrow path — but at this site's numbers it adds operational surface the owner doesn't need. Not recommended at launch.

### Caveats

- On-demand rates are regional ranges; this doc quotes the US/iad1 base rates. Per-region tables: [vercel.com/docs/pricing/regional-pricing](https://vercel.com/docs/pricing/regional-pricing).
- The Hobby 100 MB / Pro 1 GB source cap is documented for CLI deploys; Git-deploy applicability is not stated in the docs.
- Two doc discrepancies observed: Vercel's image-optimization page cites a 3600 s `minimumCacheTTL` default while Next.js 16 docs say 14400 s; and Vercel's legacy-pricing page has a visible templating bug ("$5000000.00 per 1000 source images") — the table value ($5.00) is the reliable figure.
