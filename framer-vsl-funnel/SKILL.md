---
name: framer-vsl-funnel
description: >-
  Turn an existing Optimally demo landing page into a PRODUCTION-READY VSL call funnel inside Framer,
  end to end. Use when the user wants to "make this demo production ready", "turn the demo into a
  Framer funnel", "build the VSL call funnel in Framer", "productionise the funnel", "ship the funnel
  in Framer", or hands over a built demo (e.g. demos.optimally.ltd/<slug>) and wants the real,
  conversion-wired Framer version. This is SEPARATE from the `vsl-funnel-demo` skill (that one makes
  the static HTML demo). This skill rebuilds that demo as a fully editable, responsive Framer site
  with a working opt-in modal, /typeform redirect, and a confirmation page.
  Mandatory precondition: run `npx @framer/agent@latest setup` and load the `framer` skill BEFORE this.
---

# Framer VSL Call Funnel — Production Builder

Takes an Optimally demo (the static landing page) and rebuilds it in Framer as a real VSL call funnel:
a funnel page whose CTAs open an opt-in modal (Optimally checkout embed) → `/typeform` redirect → Typeform
booking → a `/confirmed` thank-you page tuned for show-rate.

This skill DRIVES the Framer Server API via the `@framer/agent` CLI. You MUST have read the `framer`
skill and run `session new` (which loads the project-scoped `framer-project-<id>` skill) first. All
edits go through `framer.agent.applyChanges(dsl, { pagePath })`.

---

## 0. Inputs to gather (from the client DB / the demo)

| Input | Notes |
|-------|-------|
| **Source demo** | The built demo (content + brand): copy, colours, fonts, logo URL, OG image. Scrape `demos.optimally.ltd/<slug>` or reuse the demo's brand tokens. |
| **client_id** | From the client database (e.g. `WISPRFLOW`). Goes in the site config. |
| **redirect_url** | The funnel's own `/typeform` URL on its production domain (e.g. `https://<domain>/typeform`). |
| **typeform_url** | The client's booking Typeform (e.g. `https://optimally.typeform.com/to/XXXX`). |
| **use_webinar** | Default `false`. `true` for webinar funnels (sends `webinar_id`, uses the `.../form` checkout endpoint). |
| **webinar_id** | Only when `use_webinar` is true. |
| **checkout_base** | `https://optimally.checkoutpage.com/main-opt-in-form` (non-webinar) or `.../form` (webinar). |

If any of client_id / redirect_url / typeform_url are unknown, build with clearly-marked placeholders
and flag them; the funnel structure still stands up.

## 1. Project + session

```bash
npx @framer/agent@latest project new      # browser approval (interactive). Headless: project auth <url> <apiKey>
npx @framer/agent@latest session new "<projectId>"   # returns sessionId; use -s <id> on every call
```
Load `framer-project-<id>` skill. Resolve brand fonts via `framer.agent.readProject([{type:"font-search",name:"<font>"}])`.
Confirm available icon sets (`Logos`, `Lucide`) and shaders.

## 2. Build the funnel landing page (recreate the demo)

Recreate the demo's premium VSL structure as Framer sections — direct children of the page breakpoint
(`layout="stack"`, vertical, `height="auto"`, brand background, `overflow="clip"`). Keep one section
max-width (e.g. `1024px`) for text content; the VSL breaks out wider (see §4).

Sections (adapt to the demo): announcement banner → **hero** (logo, eyebrow, H1 with italic-accent runs,
1 subline, **full-width VSL**, primary CTA, quote, social-proof avatars) → trust marquee (logo ticker) →
pains (3-col cards) → **solution (use a DIFFERENT layout** than pains — e.g. 2-col horizontal feature
rows — to avoid repetition) → stats + CTA band → testimonials (3-col) → founder note → FAQ accordion → footer.

Brand: pull colours/fonts/logo from the demo. Headlines in the demo's display serif (e.g. EB Garamond),
body in its sans (e.g. Figtree). Subtle scroll-reveal only (see §9). No em dashes (see §10).

## 3. Reusable components (build each ONCE, instantiate everywhere)

Build each component **in a single `applyChanges` call** (temp ids resolve within one batch; editing an
un-instantiated component's internals across batches fails — see Gotchas).

- **CTA Button** — brand CTA colour (use the REAL brand CTA colour from the client's site, e.g. the
  lilac `#F0D7FF` pill with near-black text — NOT an invented colour). Controls: a `Label` string
  Variable (bind the label RichText to `var(--variable-<id>)`; initial "Learn More"). Three size
  variants Desktop/Tablet/Phone (~70-80% of the VSL width — chunky, primary). A continuous **scale
  "breathe" loop** (`loopEffect.repeatType="mirror" loopEffect.scale="1.04"` and **explicitly zero
  `loopEffect.rotate`/rotateX/Y/x/y/skew** or Framer spins it 360°). Expose an `On Click`
  `+EventHandlerVariable` and fire it from the button's `onTap` (`TRIGGER_EVENT`, controls.id =
  `var(--variable-<eventVarId>)`); remove the button's `link.href` (CTAs open the modal, not a link).
- **VSL Embed** — a 16:9 `Player` frame (`aspectRatio="1.7778"`, `overflow="clip"`, brand-gradient
  placeholder fill) containing a built-in Embed (`o1PI5S8YtkA5bP5g4dFz`) bound to controls: `Video URL`
  (string), `Embed Type` (OptionVariable cases `["url","html"]`), `Embed Code` (string). Leave blank.
- **Footer** — light (creamsoft, top border) so the dark logo renders: logo image, brand tagline,
  nav links (Privacy · Terms · Contact), divider, copyright `© 2026 …`, the **"Site made by Optimally"**
  line where the word **Optimally** is a TextRun with `link.href="https://optimally.ltd/client-site?client=<client_id>"`
  (use the client's ID), and the **Meta disclaimer** fine print (REQUIRED — see [[meta-disclaimer-footer]]):
  "This site is not a part of the Facebook website or Facebook Inc. Additionally, this site is NOT
  endorsed by Facebook in any way. FACEBOOK is a trademark of FACEBOOK, Inc." Place a Footer instance on every page.
- **Opt-in Modal** — white box (`width="100%"`, `radius 20`, `padding 50px 0 6px 0`, `overflow hidden`,
  `boxShadows.0="0px 40px 90px -24px rgba(0,0,0,0.45)"`) holding the checkout Embed (built-in Embed,
  `width="100%" height="384px"`) plus, absolutely positioned OVER the embed top (all `zIndex 3`+):
  a progress **track** (`top 22, left 24, right 92, height 8, radius 999, fill rgba(26,26,26,0.1),
  overflow clip`) with a **fill** child (`top 0, left 0, height 8, width 47%, radius 999, brand accent`);
  a **"47%"** label (`top 17, right 48`, brand accent, Figtree 600 12px); the **title** — font set on the
  RichText **ROOT** (Figtree 700 20px, not on the inner block), `top 70, left 22, right 22`, sentence
  case via runs: "Get sent more info " (ink) + "from <Brand>" (accent, bold) + " 👇"; and a **close ✕**
  frame (`top 12, right 12, 32×32, radius 100%, zIndex 6`). The Embed's `$control__type="html"`,
  `$control__hTML` = the exact opt-in embed (see §6).
  **Close button (CRITICAL — `DISMISS_OVERLAY` directly on the ✕ does NOT work** because the overlay is
  outside the component scope): add an `On Close` `+EventHandlerVariable` on the modal component, and on
  the ✕ frame set `onTap.0.action="TRIGGER_EVENT" onTap.0.controls.id="var(--variable-<onCloseVarId>)"`.
  Each overlay's modal instance then handles it with `onClose.0.action="DISMISS_OVERLAY"` (see §5). The
  dimmed dismissible backdrop covers click-outside.

## 4. Full-width VSL sizing (the preferred pattern — match exactly)

The hero VSL must break out of the centered max-width column to be near full-page-width. Split the hero
section into: [top block: logo/eyebrow/H1/subline (maxWidth 1024)] → [VSL, full width] → [bottom block:
CTA/quote/trust (maxWidth 1024)]. Then size the VSL instance with **FIXED px on desktop/tablet, fluid on phone**:

- **Desktop:** `width≈800px height≈450px` (16:9), `maxWidth≈1100px`.
- **Tablet:** `width≈600px height≈337px` (16:9), `maxWidth≈900px`.
- **Phone:** `width="100%" height="auto"` (the component's `aspectRatio` keeps 16:9).

Breakout / pre-call FAQ videos: **fixed `320px×180px` on desktop/tablet, `100%`/auto on phone.**
Do NOT use relative `%` widths on desktop/tablet — they render wrong; use fixed px.

## 5. Opt-in modal wiring (CTAs → modal)

Framer requires one `FixedOverlayNode` **per trigger** (a `FixedOverlayNode` can't live inside a
component, and a shared overlay only opens from its own parent). So:
1. For EACH CTA instance: `+FixedOverlayNode` parented to that CTA (dimmed `backdrop.fill="rgba(2,58,51,0.72)"
   backdrop.dismissible="true" backdrop.blockScroll="true"`), with a `+ComponentInstanceNode` of the
   **Opt-in Modal** inside (`visible="true"` — instances default to `false`! — `width="92%"
   maxWidth="560px" centerAnchorX="50%" centerAnchorY="50%" zIndex="10"`).
2. Wire each CTA: `SET <cta> onClick.0.action="SHOW_OVERLAY" onClick.0.controls.overlay="<overlayCanonicalId>"`.
   **CRITICAL:** `controls.overlay` only accepts a CANONICAL id (not a same-batch temp id), and the value
   is sticky — to (re)set it reliably do `SET <cta> onClick.0="null"` then re-SET with the canonical id.
3. Wire each modal instance's close: `SET <modalInstance> onClose.0.action="DISMISS_OVERLAY"` (the ✕
   inside the component fires the exposed `On Close` event → this dismisses the overlay).
All overlays embed the SAME modal component, so the modal is edited in one place. Build order that works:
(a) build the modal component fully in one batch; (b) create the per-CTA overlays + instances (this
instantiates the component); (c) re-point each CTA's `onClick.controls.overlay` to its overlay's
canonical id with the clear-then-set; (d) wire each instance `onClose`.

## 6. Site-wide custom code (the CLI can only set site-level code — use the path-gating workaround)

Framer CLI sets only the 4 global slots (headStart/headEnd/bodyStart/bodyEnd), not per-page code. So put
everything in `bodyEnd` and gate page-specific logic on `window.location.pathname`, with a MutationObserver
for SPA navigation. Put ALL editable values in ONE clearly-numbered config block at the top.

```html
<script>
/* ===== OPTIMALLY FUNNEL CONFIG — EDIT ONLY THIS BLOCK ===== */
var OPTIMALLY_CONFIG = {
  client_id:     "CLIENT_ID",                              // 1
  redirect_url:  "https://DOMAIN/typeform",                // 2 (this funnel's /typeform page)
  checkout_base: "https://optimally.checkoutpage.com/main-opt-in-form", // 3 (webinar: .../form)
  typeform_url:  "https://optimally.typeform.com/to/XXXX", // 4
  use_webinar:   false,                                    // 5
  webinar_id:    ""                                        // 6 (only when use_webinar)
};
/* ========================================================= */
window.OPTIMALLY_WEBINAR = OPTIMALLY_CONFIG;
window.buildOptimallyCheckoutUrl = function () {
  var c = OPTIMALLY_CONFIG, u = new URL(c.checkout_base), p = new URLSearchParams(window.location.search);
  u.searchParams.set("client_id", c.client_id);
  if (c.use_webinar && c.webinar_id) u.searchParams.set("webinar_id", c.webinar_id);
  u.searchParams.set("redirect_url", c.redirect_url);
  ["utm_source","utm_medium","utm_campaign","utm_content","ad_id","partner_id","closer_id","setter_id","adset_id"]
    .forEach(function (k){ var v=p.get(k); if(v!==null) u.searchParams.set(k,v); });
  return u.toString();
};
function optimallyRunPageCode(){
  var path = window.location.pathname.replace(/\/+$/, "");
  if (path === "/typeform") {
    var p = new URLSearchParams(window.location.search), tf = new URL(OPTIMALLY_CONFIG.typeform_url);
    var fn = p.get("full_name"), em = p.get("email");
    if (fn) tf.searchParams.set("full_name", fn);
    if (em) tf.searchParams.set("email", em);
    window.top.location.href = tf.toString();
  }
}
optimallyRunPageCode();
var _lp = window.location.pathname;
new MutationObserver(function(){ if(window.location.pathname!==_lp){ _lp=window.location.pathname; optimallyRunPageCode(); } })
  .observe(document.querySelector("title")||document.body,{subtree:true,characterData:true,childList:true});
</script>
```

Apply with `framer.setCustomCode({ html, location: "bodyEnd" })`.

The opt-in modal Embed's HTML (`$control__hTML`, type `html`) — it calls the PARENT page's builder:
```html
<style>#optimally-form{width:100%!important;max-width:100%!important;overflow:hidden}#optimally-form iframe{width:100%!important;max-width:100%!important;overflow:hidden!important;border:0;display:block}</style>
<div id="optimally-form"></div>
<script>(function(){var u=window.parent.buildOptimallyCheckoutUrl();document.getElementById("optimally-form").innerHTML='<iframe src="'+u+'" style="width:100%;height:384px;border:0;display:block;overflow:hidden;" scrolling="no"></iframe>';})();</script>
```

## 7. Blank /typeform redirect page

`framer.createWebPage("/typeform")` (NOT the DSL `+WebPageNode` — that makes a page with no breakpoint).
White fill, `metadata.noIndex="true"`, a small centered "Taking you to the application…" line. The site
code redirects it to the prefilled Typeform.

## 8. Confirmation page `/confirmed` (show-rate optimised)

`framer.createWebPage("/confirmed")`. Structure (from the Optimally SOPs — see [[framer-agents]]):
action banner ("your call isn't fully confirmed until you complete the steps below") → **hero with the
VSL high under the title** (single "watch this" line; logo instead of a badge) → "Three things to do"
(check email / how to prepare; no dynamic add-to-calendar) → **pre-call FAQ videos** (6 breakout videos,
each a VSL-Embed instance at 320×180 / fluid phone, titled with a question) → **testimonial wall** →
urgency/reschedule line → Footer component. `metadata.title` uses "|" not a dash; set `socialImage`.

## 9. Animations (subtle)

Intro only: `appearEffect.trigger="onInView" enter.opacity="0" enter.y="14" enter.transition="spring-duration"`
on section headers; add `enter.stagger="0.07s"` on grids so cards cascade. CTA = the breathe loop (§3).
**No hover lifts** on cards. (Static screenshots show the pre-reveal state — verify on the published URL.)

## 10. Polish rules

- **No em dashes** anywhere (looks AI-generated). Replace `—` with `-`; in titles/metadata use `|`.
- Smart quotes in copy. `© 2026`.
- Text `fontSize` MUST be in **px** ("48px") and set on the RichText **root** (not the inner block).
- Image fills are cover-only — size a logo frame to the SVG's exact aspect ratio so it isn't cropped.

## 11. Publish + verify + handoff

`framer.agent.reviewChanges()` after edits, then `framer.agent.publish({action:"preview"})` →
`{action:"confirm_publish", confirmationHash}`. Verify pages by screenshotting the **published URL**
(`readProject [{type:"screenshot", url}]`) — overlays/animations don't show in canvas/static shots, so
the modal must be click-tested live. Report: live URLs, the placeholders still to fill (client_id /
domain / typeform_url), and to delete any unused components in the Framer assets panel.

## Gotchas (these cost real time — heed them)
See [[framer-agents]] for the full list. The big ones:
- Build components in ONE batch; un-instantiated component internals aren't addressable across batches.
- `controls.overlay` / node-referencing controls need canonical ids + clear-then-set (sticky).
- One overlay per CTA trigger; component instances default `visible:"false"` (set true).
- `aspectRatio` works only when nothing inside forces a fixed height; for full-page-width video use fixed px on desktop/tablet, fluid on phone.
- `loopEffect` defaults to a 360° spin — zero every loop transform except the one you animate.
- Relay accumulates temp-id state → "existing id"; `relay restart` + `session new` clears it. Watch C: disk (npx fills npm-cache).
