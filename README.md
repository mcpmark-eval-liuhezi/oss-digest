# oss-digest

A weekly habit: pick one busy open-source project, study a slice of its community activity, and write it up.

- **Week 1 project:** [vercel/next.js](https://github.com/vercel/next.js) — The React Framework (143k+ stars, 28,700+ discussions, 2,600+ open PRs)

---

## 1. Starred-list snapshot (as found this week)

The starred list was **not empty** — it contained 6 repositories. Snapshot taken at the start of this week:

| # | Repository | Language | Stars | Notes |
|---|-----------|----------|-------|-------|
| 1 | [OpenBMB/AgentCPM](https://github.com/OpenBMB/AgentCPM) | Python | ~833 | End-to-end infrastructure for training and evaluating LLM agents |
| 2 | [PaddlePaddle/ERNIE](https://github.com/PaddlePaddle/ERNIE) | Python | ~7.7k | Official repo for ERNIE 4.5 and ERNIEKit |
| 3 | [FudanNLP/fastNLP](https://github.com/FudanNLP/fastNLP) | Python | ~3.1k | Modularized, extensible NLP framework |
| 4 | [maplezzz/NTU_ML2017_Hung-yi-Lee_HW](https://github.com/maplezzz/NTU_ML2017_Hung-yi-Lee_HW) | Jupyter Notebook | ~874 | NTU ML2017 course homework (Hung-yi Lee) |
| 5 | [fighting41love/funNLP](https://github.com/fighting41love/funNLP) | Python | ~84k | Curated Chinese/English NLP resource collection |
| 6 | [996icu/996.ICU](https://github.com/996icu/996.ICU) | — | ~277k | Protest repo for counting stars; archival |

**Why none of them was picked:** none fit the "genuinely busy community" bar —
- AgentCPM, fastNLP: no Discussions section at all (0 threads).
- ERNIE: has open PRs (37) but only 3 discussion threads ever, and the unanswered ones get no maintainer replies.
- funNLP, NTU homework, 996.ICU: static resource/course/protest repos, not actively developed communities.

So a fallback project was selected manually (vercel/next.js). Note: an attempt to star it for future weeks was blocked by the access token's permissions, so it is **not yet** in the starred list — worth starring manually.

---

## 2. Discussion summary

**Thread:** [#98310 — "How to stop Next.js from caching notFound() renders for invalid dynamic route params?"](https://github.com/vercel/next.js/discussions/98310) (Help category, opened Sep 2025, still active Oct 2026)

**The question:** A developer runs a music-streaming site with a Next.js 16 App Router route `app/[slug]/page.jsx` (~2,200 real song pages). Bots constantly probe the route with garbage URLs (`wp-login.php`, `.env`, URL-encoded junk). Every one of those invalid slugs resolves to `notFound()` — and Next.js persists a cache entry (`.html`/`.meta`/`.rsc` under `.next/server/app`) for **each unique invalid slug**, as if it were a successful page. The disk cache ballooned to ~2 GB for a 2,200-page site. They tried migrating to Cache Components (`"use cache"` with `notFound()` called outside the cached function), expecting that only successful data would be persisted — but invalid slugs still got cached, because the App Shell / ISR upgrade mechanism persists the per-param render result regardless of the outcome.

**Useful answers from the thread:**

1. **The two-layer cache explanation (top answer):** `'use cache'` only controls the *data* cache — the return value of the cached function. The App Shell upgrade mechanism operates at the *routing* layer: an unlisted dynamic param is served a shell, then a background render for that specific param is persisted no matter whether the page rendered successfully or called `notFound()`. From the router's perspective, a 404 response **is** the resolved result for that param, so it caches it like any other. There is no way from inside the page component to tell the router not to persist.
2. **The maintainer reply:** "We are working on an API to allow you to control this behavior, it is a bit early to share impl and API details, but it'll happen soon."
3. **The recommended interim pattern:** validate in `proxy.ts` (formerly `middleware.ts`) with a narrow `matcher` — reject obvious scanner traffic with cheap regex patterns (`.php`, `.env`, `.sql`, `../`) first, then consult an authoritative existence check (a backend endpoint or a shared, bounded cache) for plausible slugs, returning a real 404 **before** the request reaches the render/ISR path. This keeps on-demand caching for valid slugs while placing a hard boundary in front of unbounded negative params.
4. **A correction that improved the suggested code:** don't return 404 on *any* non-OK upstream response — `!response.ok` would also swallow 401/429/5xx and make real songs vanish during an API outage. Return 404 only when existence is explicitly denied; fail open or return 503 otherwise.
5. **On state:** avoid a mutable module-level `Set` for the allowlist — proxy code runs separately from render code and shouldn't rely on shared globals surviving restarts or multiple workers.
6. **Documented trade-offs** (from the top answer): Cache Components + proxy/CDN validation vs. the old `dynamicParams = false` model (requires enumerating all valid params) vs. fully dynamic rendering (loses the page cache).

---

## 3. Pull-request summary

**PR:** [#100582 — "Turbopack: Fix CSS URLs for custom asset rules"](https://github.com/vercel/next.js/pull/100582) (open, opened Oct 22, 2026, +45/−1 across 5 files)

**What it does at a high level:** Turbopack currently treats every asset matched by a custom rule with `type: 'asset'` as a *JavaScript URL module*. When that same asset is referenced from CSS via `url(...)`, the generated URL points at `/_next/static/chunks/...` — which 404s — instead of an emitted `/_next/static/media/...` asset. The fix makes the module-type selection depend on the *reference type*: CSS references get the CSS static-URL module (correct emitted asset path) while direct JS imports keep the JavaScript module.

**Files / areas touched:**

- `turbopack/crates/turbopack/src/module_options/module_rule.rs` — introduces a new `ModuleType::StaticUrl` variant (replacing the hardcoded `StaticUrlJs` mapping for configured `Asset` types), so the effective module type can be decided per-reference rather than per-rule.
- `turbopack/crates/turbopack/src/lib.rs` — in `apply_module_type`, dispatches the new `StaticUrl` type: if the reference is `ReferenceType::Url(UrlReferenceSubType::CssUrl)` build a `StaticUrlCssModule`, otherwise build the existing `StaticUrlJsModule`.
- `test/e2e/app-dir/webpack-loader-module-type/app/page.tsx` (+ `style.module.css` added) — test fixture: imports the new CSS module and renders an element using `styles.background`, whose `background-image: url('./test.svg')` exercises the CSS-asset path.
- `test/e2e/app-dir/webpack-loader-module-type/webpack-loader-module-type.test.ts` — regression test asserting the computed `background-image` URL matches `/_next/static/(immutable/)?media/test.<hash>.svg` **and** that fetching that URL returns 200, verifying the asset is actually emitted and reachable.

**Why it's interesting:** a textbook community bug fix — it closes a long-standing issue (#89828), keeps both import paths (JS and CSS) working, and includes end-to-end coverage that checks not just the URL shape but that the file is actually served. Labels: `Turbopack`, `tests`.
