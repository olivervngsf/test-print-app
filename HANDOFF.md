# Handoff — Nail Checkout (thermal receipt app)

Everything a fresh agent needs to continue this project. Written 2026-09-18.

---

## 1. What this is

A single-file web app for a nail salon counter. Staff ring up a sale on an
iPhone/iPad and print a receipt to a **58mm Bluetooth thermal printer**.

- **Repo:** `olivervngsf/test-print-app`
- **Production branch:** `main`
- **Dev branch used so far:** `claude/loving-lamport-5e4kix`
- **Live:** https://test-print-app.vercel.app
- **Host:** Vercel project `test-print-app` (team `viet-nguyens-projects-3e51b071`, id `prj_HSkysEJ0kmVreFRxCxbwB9bhd6OD`)
- **Entire app:** `index.html` — no build step, no dependencies, no framework.
  `vercel.json` only sets `cleanUrls` and a no-cache header.

State lives in `localStorage` under key `nail.settings`. There is no backend,
no database, no accounts.

---

## 2. The core constraint (read this before changing print code)

iOS Safari **cannot** talk to a Bluetooth printer. No Web Bluetooth on iOS.
So the app hands the receipt to a third-party iOS app called **Thermer**
(App Store id `1599863946`) via a custom URL scheme:

```
thermer://?data=<URL-encoded JSON>
```

The JSON is an object keyed `"0"`, `"1"`, `"2"`… Each value is one *entry*:

```js
{ type: 0,              // 0 = text, 1 = image
  content: "Lucky Nails",
  bold: 0|1,
  align: 0|1|2,         // left | center | right
  format: 0|1|2|3|4 }   // normal | 2x height | 2x both | 2x width | small
```

For an image entry the app sends `type: 1` with a `data:image/png;base64,…`
value in **both** `path` and `content` (field name was inferred from the
vendor's Android docs; sending both is a hedge that works).

### Thermer's free tier limits — the whole story

| Limit | Effect |
|---|---|
| Max **3 entries** per receipt | More than that shows "Upgrade to Print This Receipt" |
| Max **3 prints per day** | Hard stop, no workaround |
| "Slow Printing" | Free tier is deliberately throttled |

The entry cap was observed at 5 earlier in the session and at 3 later
(screenshots show both wordings), so **assume 3 and don't exceed it.**

### What was tried, and what was learned

1. **One text entry per receipt line** (the original design) →
   blocked by the entry cap.
2. **Pack many lines into one text entry**, padded to full paper width,
   joined with `\n` → *printed as one run-on stream.* Thermer collapses
   runs of spaces and drops the newlines inside a text entry. **This does
   not work. Don't retry it.** Photos of the bad output are in the session.
3. **Whole receipt as one PNG** (`type: 1`) → columns, bold, margins all
   print exactly as previewed. Works. But slow, because every pixel row
   goes over Bluetooth.
4. **Hybrid, current default** → 3 entries: a blank text line, the salon
   name as printer-font text, and a PNG of everything below the name.
   Fastest option that still keeps the layout. Verified 3 entries, body
   PNG 384×349, link ≈6 KB.

**Speed is capped by Thermer, not by us.** Their paywall screen lists
"Slow Printing" as a *free-tier limitation*. No amount of shrinking the
image makes free-tier printing fast.

---

## 3. Code map (`index.html`)

All logic is in the single `<script>` block at the bottom.

| Function | Job |
|---|---|
| `buildEntries(d)` | Turns a sale into the canonical array of text entries (one per printed line). The single source of truth for receipt content. Used by print, preview, and share. |
| `saleData()` | Snapshots current cart + settings into the object `buildEntries` expects. |
| `drawPaper(entries)` | Renders the on-screen receipt preview (DOM, not canvas). |
| `sendToPrinter(entries, w)` | Branches on `S.mode` and builds the `thermer://` link. |
| `packEntries(entries, w)` | Text-mode packer. Known-broken output (see §2.2); kept only as a fallback. |
| `receiptCanvas(entries, w)` | Draws entries to a canvas at printer resolution (384px @58mm, 576px @80mm) and thresholds to pure black/white. |
| `receiptPng(entries, w)` | Encodes that canvas as a **true 1-bit PNG** using `CompressionStream("deflate")`. ~5x smaller than `canvas.toDataURL`, which only emits 8-bit. Falls back to `toDataURL` if `CompressionStream` is missing. |
| `receiptImage()` | Blob version for the Share button. |

### Print modes (Settings → "Print as")

- `hybrid` *(default)* — blank line + text header + PNG body. 3 entries.
- `image` — whole receipt as one PNG. 1 entry. Exact but slowest.
- `text` — `packEntries` output. Compact but runs together. Fallback only.

### Settings keys (`localStorage` → `nail.settings`)

`shop, addr, phone, techs, tax, w (58|80), note, top, bottom, mode`

`top` / `bottom` are blank lines for tear room, clamped 0–10. Note: in
`hybrid` mode the top spacer is a fixed single line, so the `top` setting
only affects `image` mode. The UI label says so. **This is a wart worth
fixing** — see §6.

---

## 4. How to test without a printer

There is no test suite. Verification so far has been a headless-Chromium
harness. Chromium is preinstalled in the dev container at
`/opt/pw-browsers/chromium-1194/chrome-linux/chrome`.

```bash
npm i playwright-core@1.55
```

Then drive `file:///…/index.html`, click through a sale, and call the
page's own functions to capture the exact payload the Print button builds:

```js
const r = await page.evaluate(async () => {
  const i = lastEntries.findIndex(e => e.format && e.align == 1);
  const url = await receiptPng(lastEntries.slice(i + 1), S.w);
  return { mode: S.mode, imgDataUrl: url };
});
```

Write the base64 to a `.png` and look at it. That is how every layout
change in this session was checked.

Sanity checks that caught real bugs:
- Entry count must be **≤ 3**.
- PNG header must report bit depth **1**, color type **0**.
- IDAT CRC must validate and inflate to `(ceil(W/8)+1) * H` bytes.
- Body rows must not visually overlap after line-height changes.

---

## 5. Current state

**Committed and pushed to `main`:** commit `7719716`, hybrid mode, all of §3.

**Not yet on production.** Vercel had a bad night:

- The push to `main` did not trigger a build (webhook silent for ~30 min).
- `deploy_to_vercel` returned HTTP 500 but still created a deployment,
  which then sat in `INITIALIZING` and never built.
- `create_git_project` produced a build that **did** succeed
  (`dpl_4gfptFNPCP4zzM8RHuXYetxEv1pL`, READY) — but with `target: null`,
  i.e. a **preview**, aliased only to
  `test-print-app-git-main-…vercel.app`, which is SSO-protected.

So https://test-print-app.vercel.app still serves the **previous**
version (`mode: "image"`, per-line spacing 1.4). It works fine; it is just
the slower layout.

### First thing to do

Promote `main` to production. Easiest: in the Vercel dashboard, open the
project → Deployments → the `main` build → **Promote to Production**.
Or push any commit to `main` once Vercel's webhook is healthy again, or
run `vercel --prod` from a checkout.

Verify by fetching the live page and grepping for `mode:"hybrid"` — the
old build says `mode:"image"`.

---

## 6. Open items

1. **Promote to production** (§5). Blocking.
2. **Real-world print test.** The hybrid payload has never been sent to an
   actual printer — Vercel blocked the deploy. Confirm 3 entries clear the
   free tier and that the layout is intact.
3. **`top` setting is ignored in hybrid mode.** Either make the header
   entry carry N blank lines, or hide the field unless mode is `image`.
4. **Image-entry field name is unverified.** We send both `path` and
   `content`. Thermer's iOS URL-scheme docs were unreachable from the dev
   container (`thermer.app` does not resolve; `matetech.in`,
   `apps.apple.com`, `play.google.com` are blocked by the egress proxy).
   If an image ever fails to print, this is the first suspect.
5. **The 3-prints-per-day cap makes free Thermer unusable for a real
   salon.** This is a product decision, not a code one. Options discussed
   with the owner:
   - **Thermer Lifetime** — one payment, unlimited, fast printing. Price
     had not loaded on their screen; Monthly $4.99, Yearly $19.99.
     If bought, switch the default back to `text` mode with one entry per
     line — that is the fastest and cleanest output of all.
   - **Android device at the counter** — Chrome on Android supports Web
     Bluetooth, so the app could drive the printer directly with ESC/POS
     and skip Thermer entirely. Free forever, unlimited, fast. Requires
     rewriting `sendToPrinter` against `navigator.bluetooth`.
   - **Local print box** (Raspberry Pi / old laptop paired to the printer,
     small HTTP service on the LAN). Works for iPhones too.

   The owner has not chosen yet. **Ask before building any of these.**

---

## 7. Conventions

- Keep it one file. No build step, no dependencies. That is deliberate —
  the owner deploys by pushing to `main`.
- `buildEntries` stays the single source of receipt content. Print,
  preview, and share must never diverge.
- Thermal output is 1-bit. Anti-aliased grey prints muddy, so always
  threshold before encoding.
- Fewer pixel rows = faster print. Any layout change should report its
  effect on canvas height.
- The owner is non-technical and reads results, not process. Lead with
  what changed and what they should do.
