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

## 0. Inputs to gather (resolve these yourself from a `prospect` identifier)

You are typically given only a **`prospect`** (a Prospect Name, Slug, or domain). Resolve everything else:

1. **Find the demo** — search Baserow **`Demo Landing Page Data`** (table `1024310`) for the prospect (match
   `Prospect Name`, then `Slug`, then `Source URL`/`Live Demo URL`; if several, prefer `Status` =
   Deployed/Sent/Live and most recent `Date Generated`). Capture `Slug`, `Live Demo URL`, `Headline`, `ICP`,
   `Primary Colour`, `Secondary Colour`, `Logo URL`. If none match, stop and report "demo not found".
2. **Get the demo CODE (source of truth for content + brand)** — `git -C "D:\Claude Cowork\demos" pull`, then
   read **`D:\Claude Cowork\demos\<Slug>\index.html`** (repo `github.com/ReubenShears/demos`). The HTML wins
   on copy/structure/sections; cross-check brand tokens against the Baserow row.
3. **client_id / redirect_url / typeform_url** — from the client record (Client Data) where available;
   `client_id` is the UPPERCASE slug. If any are unknown, build with clearly-marked placeholders and flag them.

| Input | Notes |
|-------|-------|
| **Source demo** | From step 1+2 above: the demo HTML at `demos/<Slug>/index.html` (content + brand): copy, colours, fonts, logo URL, OG image. |
| **client_id** | From the client database (e.g. `WISPRFLOW`). Goes in the site config. |
| **redirect_url** | The funnel's own `/typeform` URL on its production domain (e.g. `https://<domain>/typeform`). |
| **typeform_url** | The client's booking Typeform (e.g. `https://optimally.typeform.com/to/XXXX`). |
| **use_webinar** | Default `false`. `true` for webinar funnels (sends `webinar_id`, uses the `.../form` checkout endpoint). |
| **webinar_id** | Only when `use_webinar` is true. |
| **checkout_base** | `https://optimally.checkoutpage.com/main-opt-in-form` (non-webinar) or `.../form` (webinar). |

If any of client_id / redirect_url / typeform_url are unknown, build with clearly-marked placeholders
and flag them; the funnel structure still stands up.

## 1. Project + session — CLAIM a pool project (do NOT `project new`)

This runs **headless/remote**, so never call `project new` (it needs a browser). Instead **claim a
pre-authorized project from the pool** (see [[framer-project-pool]] — Baserow table `Framer Project Data`,
id `1033106`). Lease pattern:

1. **Claim** the first `Status = "Available"` row: `update_rows` to set `Status="Claimed"`,
   `Claimed By="<client_id>"`, `Claimed At=<now>`. Re-read the row; if `Claimed By` ≠ this client, another
   run grabbed it — take the next Available row (guards the race). If none are Available, STOP and flag the
   pool is empty (top it up with the `framer-project-pool` skill).
2. **Authenticate headlessly** with that row's `Project ID` + `API Key`:
   ```bash
   npx @framer/agent@latest project auth "<Project ID>" "<API Key>"   # headless, no browser
   npx @framer/agent@latest session new "<Project ID>"                 # returns sessionId; use -s <id>
   ```
3. Load `framer-project-<id>` skill. Resolve brand fonts via
   `framer.agent.readProject([{type:"font-search",name:"<font>"}])`. Confirm icon sets (`Logos`, `Lucide`) + shaders.
4. **On completion**, update the claimed row: `Status="Live"`, write the live `.framer.app` URL into `Notes`
   (custom domain is connected later by remixing). On failure, set `Status="Error"` with a note so the pool
   stays trustworthy (don't silently leave it `Claimed`).

## 2. Build the funnel landing page (recreate the demo)

Recreate the demo's premium VSL structure as Framer sections — direct children of the page breakpoint
(`layout="stack"`, vertical, `height="auto"`, brand background, `overflow="clip"`). Keep one section
max-width (e.g. `1024px`) for text content; the VSL breaks out wider (see §4).

**CRITICAL — set the page breakpoint to `height="auto"`.** A brand-new Framer page (and every breakpoint
you make with `CREATE_VARIANT`) defaults to a **FIXED height (~1000px)**, which CLIPS everything below the
fold on the published site and the canvas. After building, `SET <pageBreakpointId> height="auto"` on the
Desktop breakpoint **and** on each Tablet/Phone breakpoint. (`createWebPage` pages may start `auto`, but
verify — set it anyway.) Symptom: page content is cut off / canvas frame ends after ~one viewport.

**Center the hero** (eyebrow, H1, sublines, logo row all centered above the VSL) — the production funnel
hero is ALWAYS centered for a clean VSL look and consistency with the centered video, even if the source
demo's hero is left-aligned. Set the hero's top block `stackAlignment="center"` and `textAlignment="center"`
on its eyebrow/H1/sublines. (Card sections like pains/solution/founder can stay left-aligned among themselves.)

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
- **FAQ Item (accordion)** — a 2-variant toggle component. Build across **4 batches** (variable binding
  needs canonical var ids; the answer-wrapper needs the closed-variant canonical id):
  1. **Component + Closed variant + variables.** `+ComponentNode faqItem name="FAQ Item"`; a `Closed`
     FrameNode (`width="100%"`, **`height="auto"`** — do NOT use a fixed closed height, see WARNING below;
     `layout=stack vertical`, `gap 8`, `padding 19px 22px`, brand card fill, `radius 10`, `border`,
     `overflow="clip"`, `cursor="pointer"`); two string Variables `Question` and `Answer`
     (`scope="<faqItem>"`). Capture canonical ids; `serialize` to read the two variable ids (control keys
     `$control__question` / `$control__answer`).
  2. **Inner nodes** (parent into the canonical Closed-variant id): a `Row` frame (`horizontal`,
     `space-between`) holding the question RichText (`text="var(--variable-<qVarId>)"`, display font,
     `width="1fr"`) + a `Plus` Lucide IconNode (brand accent, fixed 24px); then an **answer wrapper**
     FrameNode (`width="100%"`, **`height="0px"`, `overflow="clip"`**, vertical) and inside it the answer
     RichText (`text="var(--variable-<aVarId>)"`, body font, `width="100%"`). Capture the wrapper's id.
  3. **Open variant:** `CREATE_VARIANT faqOpen from="<closedId>"; SET faqOpen name="Open"`; then reveal
     the answer **only in Open** via the compound id: `SET <openId><answerWrapperId> height="auto"`.
  4. **Wire the toggle:** `SET <closedId> onTap.0.action="SET_VARIANT" onTap.0.controls.variant="<openId>"`;
     `SET <openId> onTap.0.action="SET_VARIANT" onTap.0.controls.variant="<closedId>"`; rotate the plus
     into an ✕ on open via the compound id `SET <openId><plusId> rotation="45deg"`.
  **WARNING — never use a fixed pixel height on the Closed variant.** A fixed closed height (e.g. 62px)
  reveals the top of the answer ("answer peeks below the question") and breaks entirely when a question
  wraps to 2 lines (mobile). The robust pattern is: Closed variant `height="auto"` (fits the question
  row), the answer lives in a **height-0 clipped wrapper**, and ONLY the Open variant sets that wrapper
  to `height="auto"`. This collapses/expands cleanly for any question length and still animates.
  Instantiate per question with `$control__question` / `$control__answer` (instances default
  `visible:"false"` — set true, `width="100%"`).

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

## 10. Design system + polish rules (be granular — these prevent the recurring drift)

Do NOT set typography ad-hoc per section. Work from ONE fixed system so sizing, alignment, and contrast
are deterministic and consistent across every section and breakpoint.

### 10a. CRITICAL — `SET <node> text="…"` WIPES the node's styling back to defaults (16px, black, left)
A RichTextNode's font/size/colour/alignment do NOT survive a later `SET text="…"` — Framer rebuilds the
text with defaults. This silently breaks contrast (black text on a dark band), sizing (everything 16px),
and alignment (everything left). It is the #1 cause of "titles wrong size / wrong colour / not aligned".
**Rules:** (1) Set the FINAL copy at creation. (2) If you must edit copy afterwards (e.g. the en-dash
pass in §10c), **re-apply the full style in the SAME `SET`** (fontName, fontWeight, fontSize, textColor,
textAlignment, lineHeight, width, maxWidth) — or immediately follow with a style-only `SET`. (3) After any
copy edit, screenshot and scan for the default signature: **16px black left-aligned** text where it
shouldn't be.

### 10b. Type scale (desktop / tablet / phone — px, on the RichText ROOT)
| Role | Desktop | Tablet | Phone | Font |
|------|---------|--------|-------|------|
| Hero H1 | 52 | 40 | 30 | display (Archivo Black) |
| **Section H2 (ALL sections, uniform)** | **36** | 30 | 25 | display |
| Founder / side-module H2 | 30 | 26 | 24 | display |
| Card H3 title | 18–20 | same | same | display |
| Eyebrow (kicker) | 13 (700, letterSpacing 1.4px) | same | same | body |
| Subline / section sub | 17–18 | 16 | 15 | body |
| Card body | 15–16 | same | same | body |
| Caption / fine | 13–14 | same | same | body |
| Confirmed H1 | 40 | 32 | 27 | display |
Keep ALL section H2 the SAME size — never let one section be 38 and another 34.

### 10c. Alignment system (consistency is the goal)
- **Every section header is CENTERED:** eyebrow + H2 + sub all `textAlignment="center"`, AND the
  header block is centered in its section — set the section's inner wrapper `stackAlignment="center"`
  (otherwise a `maxWidth` header sits left even with centered text).
- **Hero fully centered** (logo row, eyebrow, H1, sublines, caption, CTA, rating) — §2.
- **Card GRIDS stay full width; card INTERNAL content is left-aligned** (icon, title, body).
- **CTA instances centered** in their section.
- Founder may be a 2-col left module (the one allowed exception) — keep its own internals consistent.

### 10d. Contrast — pick text colour FROM the section background, never a global default
- **On light bg** (cream `#FFF7ED` / white): titles = ink `#111`; body = `rgba(17,17,17,0.65–0.7)`;
  eyebrow/accents = brand colour.
- **On dark/brand band** (deep green, etc.): titles = cream `#FFF7ED`; body = `rgba(255,247,237,0.7)`;
  eyebrow/stat-numbers = the BRIGHT accent (e.g. `#2ca800`). **NEVER** ink/dark text on a dark band.
- When you add or restyle any text, decide its colour by its parent section's `fill`. Verify on the
  published screenshot that every title/body is legible on its band.

### 10e. Layout — flex children that share a row with a fixed-width sibling MUST be `width="1fr"`, never `100%`
In a horizontal stack (e.g. icon + text feature rows, or a fixed-width image + text column like the
founder section), a text/content child set to `width="100%"` tries to be the full parent width and
**overflows past the fixed sibling — the right edge gets clipped** (text cut off mid-word). Always set the
flexible child to `width="1fr"` so it takes the *remaining* space. This is a top cause of "text is cut
off". (Vertical stacks are fine with `width="100%"`.)

### 10f. Other polish
- **No em dashes** anywhere (looks AI-generated). Replace `—` with `-`; in titles/metadata use `|`.
- **CRITICAL — never use a spaced hyphen `" - "` as a clause separator.** Framer's render-time
  **smart typography** silently converts `" - "` into an **en-dash `–`** on the published site (the DSL
  stores a real hyphen, code 45, so it is INVISIBLE in `serialize`/canvas — only the published render
  shows the en-dash). There is NO API toggle (`smartQuotes`/`smartTypography` attributes are pruned).
  So in copy, replace every `" - "` with a **comma, colon, "and", or a full stop / two sentences** —
  whichever reads best. UNSPACED hyphens are safe and do NOT convert: ranges (`1-2 weeks`,
  `20-30 minutes`), compounds (`word-of-mouth`, `lead-gen`, `end-to-end`, `lock-in`), and a hyphen at
  the very start of its own text run (e.g. a coloured headline accent run `" - Straight From "`) all
  stay hyphens. After publishing, screenshot the live URL and scan for any `–`.
- Smart quotes in copy (Framer auto-curls straight `'`/`"` on render — good, leave them straight in DSL).
- Text `fontSize` MUST be in **px** ("48px") and set on the RichText **root** (not the inner block).
- Image fills are cover-only — size a logo frame to the SVG's exact aspect ratio so it isn't cropped.

## 11. Publish + verify + handoff

`framer.agent.reviewChanges()` after edits, then `framer.agent.publish({action:"preview"})` →
`{action:"confirm_publish", confirmationHash}`. Verify pages by screenshotting the **published URL**
(`readProject [{type:"screenshot", url}]`) — overlays/animations don't show in canvas/static shots, so
the modal must be click-tested live.

**MANDATORY final full audit — do NOT declare done without it.** Quality drifts silently (styling wipes,
overflow clips, contrast, peeking answers), so end EVERY build/dry-run with both an automated rect audit
and a visual QA scan.

**(1) Automated overflow/clip audit (run on every page breakpoint).** Serialize the page deep and flag
(a) any node whose rendered rect extends past its parent's right/bottom edge (true clipping), and (b) the
flex-overflow signature (a horizontal stack with both a fixed-px-width child and a `width="100%"` child →
fix the flexible child to `1fr`, §10e). It must come back **0 overflow / 0 flex-risk**:
```js
const tree = await framer.agent.serialize({ id: breakpointId, depth: 20 });
(function w(n,p){ const a=n.attributes||{};
  if(p && n.$rect && p.$rect && (n.$rect.x+n.$rect.width)-(p.$rect.x+p.$rect.width) > 1.5)
     console.log("OVERFLOW", n.id, (typeof a.text==="string"?a.text.slice(0,30):n.type));
  if(a.layout==="stack" && a.stackDirection==="horizontal"){
     const ws=(n.children||[]).map(c=>(c.attributes||{}).width||"");
     if(ws.some(x=>/px$/.test(x)) && ws.some(x=>x==="100%")) console.log("FLEXRISK", n.id, ws); }
  (n.children||[]).forEach(c=>w(c,n)); })(tree,null);
```

**(2) Visual QA scan.** Scroll-reveal sections render at `opacity:0` in a published static screenshot, so
capture each section as a CANVAS shot by node id (`readProject [{type:"screenshot", id:"<sectionId>"}]`) —
canvas ignores `appearEffect` — and check every section:
- **Type scale:** every section H2 is the same size (§10b); no title shrunk to ~16px.
- **Alignment:** every section header centered; hero fully centered; CTA centered (§10c).
- **Contrast:** no dark/ink text on a dark band; no cream text on a light band (§10d).
- **Default-signature:** no text accidentally 16px / black / left (the `SET text` styling-wipe — §10a).
- **Overflow/clip:** no text cut off at an edge (cross-check the rect audit; §10e).
- **FAQ:** closed items show ONLY the question, no answer peeking (§3 FAQ WARNING).
- **No en-dashes:** scan for `–` from the smart-typography conversion (§10c).
- **No page clipping:** every page breakpoint is `height="auto"` (§2); content runs to the footer.

Fix everything the audit surfaces, republish, and re-run the audit until clean.

**Handoff** (once the audit is clean):
- **Pool row** (`Framer Project Data` 1033106) → `Status="Live"`, live `.framer.app` URL into `Notes`.
  If you claimed a row but failed before publish, set it `Status="Error"` with a note — never leave it `Claimed`.
- **Demo row** (`Demo Landing Page Data` 1024310) → note the Framer funnel URL in `Notes`.
- **GHL** → if the lead exists, write the funnel URL onto the matching CRM record (find by domain, write the
  custom field by ID, not key — see [[ghl-demo-link-field]]).
- **Slack** → summary in Optimally house style (mrkdwn `*bold*`, `<url|label>`, `>` groups, no em dashes —
  see [[slack-message-style]]): client, live funnel URL, claimed Framer project, placeholders still to fill.

Report: live URLs, the placeholders still to fill (client_id / domain / typeform_url), and to delete any
unused components in the Framer assets panel.

## Gotchas (these cost real time — heed them)
See [[framer-agents]] for the full list. The big ones:
- Build components in ONE batch; un-instantiated component internals aren't addressable across batches.
- `controls.overlay` / node-referencing controls need canonical ids + clear-then-set (sticky).
- One overlay per CTA trigger; component instances default `visible:"false"` (set true).
- `aspectRatio` works only when nothing inside forces a fixed height; for full-page-width video use fixed px on desktop/tablet, fluid on phone.
- `loopEffect` defaults to a 360° spin — zero every loop transform except the one you animate.
- Relay accumulates temp-id state → "existing id"; `relay restart` + `session new` clears it. Watch C: disk (npx fills npm-cache).
