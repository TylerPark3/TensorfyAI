# Tensorfy — redesign brief

Edit the existing site. Do not rebuild it.

**File:** `/Users/tylerpark/Desktop/Tensorfy/site/public/index.html` (single self-contained file)
**Live:** https://tensorizeai.com — Cloudflare deploys on push to `main`
**Never move confidential archives into `public/`.** `ddap-archive.html` and `advisors-archive.html` stay outside it.

---

## Rule 0 — nothing on this site may reveal that the application is credit assessment

This overrides everything else in this brief. It was raised twice; treat it as the acceptance criterion.

A scan of the current page finds **49 occurrences across 20 terms** still visible:

```
credit 11 · scorecard 5 · default 4 · loss tolerance 3 · salary 3 · lender 2
approval 2 · charge-off 2 · debt 2 · term 2 · AUC 2 · applicant 2 · portfolio 2
FICO 1 · delinquency 1 · approvable 1 · adverse 1 · utilization 1 · inquiries 1 · loan 1
```

Every one of these must be gone from the served HTML — body copy, figure captions, SVG `<text>`, table cells, `alt`, `aria-label`, and `<title>`. A reader must not be able to infer the vertical.

**Keep the graphics, strip the labels.** Redacted structure reads as withheld; an unlabelled diagram reads as classified, which is the intended effect. Replace identifying labels with redaction bars rather than deleting the elements.

Positioning that survives: *numerical intelligence for institutions*, application sealed.

---

## Overall

The page reads like it is explaining itself to an engineer. It should read like a lab that assumes you already know why you are here.

- **Too much information for a company at this stage. Cut, don't add.**
- The opening should carry small, precise details about Tensorfy — not paragraphs.
- **Fonts are still too small.** Raise them again. Body and the uppercase statement register both. Legibility beats density; if something has to give, let it be the amount of text, not the size.
- Investors reading this are high-level. Every figure should land in about three seconds.

---

## 1 — Opening section: who we are

Make the identity statement the loudest thing on the page.

- **"NUMERICAL AI FRONTIER LAB" must be significantly bigger.** It is the whole positioning and currently it sits as a small eyebrow.
- The introduction of who we are needs to come forward and read first.
- Keep the tensor-field hero animation and its sequence exactly as it is.
- Hero copy stays extremely short.

---

## 2 — The four data types (new figure, replaces the current positioning map)

Source: the investor deck. AI processes **four** data types. The current graph only shows language vs numerical, which understates the argument.

Build a **simple horizontal figure with four columns**:

| AUDIO | VISUAL | LANGUAGE | NUMERICAL |
|---|---|---|---|
| speech-to-text | detection, segmentation, classification | LLM | financial models |
| 1990s – 2010 | 2012 – 2020 | 2017 – now | 1980s – now |
| neural net | neural net | neural net | **still machine learning** |

- **Tensorfy sits under NUMERICAL**, marked in signal red.
- The point that must land without being explained: three modalities got their neural-network moment. The fourth never did.
- Keep it simple. Four columns, a short label each, one marker. No dense annotation.
- This replaces the current scatter/quadrant positioning map.

---

## 3 — Wrappers section: concentrate it

Section order is: **(1) title / who we are → (2) numerical AI lab → (3) wrappers, and we are not.**

- **Lead with the number: ~90% of AI startups are wrappers. We are not.** Make it large and unmissable.
- **Delete most of the writing in this section.** Only the 90% figure and the theme should remain apparent.
- State plainly: our lab builds actual neural-network technology. **Core technology is a transformer-based neural network company.**
- Remove the supporting paragraphs, the layered "democratization" explanation, and any repetition of the same claim.

---

## 4 — Map section (keep, improve)

This section works. Three changes:

- **San Francisco → San Jose.** San Jose is the disclosed location.
- **Only San Jose is disclosed.** Los Angeles, Las Vegas and every other site get the stealth treatment — keep the **red dots on the map**, but **blur/redact the location names and descriptions underneath**.
- **Improve the map itself.** The current rendering is not good enough: land shapes are rough. Keep it a rectangular world map, small points, no lines connecting locations.

---

## 5 — Delete entirely

**Remove Fig. 00b — "Language has a settled architecture. Numbers do not."** — including all three numbered cards (01 irrelevant features, 02 orientation, 03 irregular targets), the stat strip beneath it, and its citation line.

Delete the section's CSS with it. Do not leave dead rules.

---

## 6 — "Institutions emit numbers" — build a section around this

This is the best idea on the page. Make it a real section, themed around it.

The argument, in this order:

1. Institutions emit numbers continuously.
2. **There is no good way to process them today.**
3. What exists is inefficient — and inefficient because it is not accurate.
4. Turning that data into something actionable is genuinely hard. Operators say so.
5. **This is who we are.**

- Keep the large number treatment — it is the strongest visual on the page.
- Short lines only. No paragraph blocks.

---

## 7 — Execute Rule 0

Work through these specific places, which is where the 49 hits live:

- **Fig. 02a input rail** — `SALARY`, `AGE`, `WAGE TYPE`, `TENURE`, `DEBT RATIO`, `UTILIZATION`, `INQUIRIES`, `DELINQUENCY`, `TERM` → redaction bars. Keep the network animation untouched.
- **§ Selected Results** — every figure framed as a credit outcome (AUC deltas, approval rates, loss tolerance, charge-off, the leakage note). Either remove, or reduce to an unlabelled result with the metric redacted.
- **Enterprise comparison table** — row labels and every cell naming scorecards, tokenizers-vs-digits, residency in lending terms.
- **Programs** — the numerical program's description, and the savings model with its portfolio/margin inputs.
- **Embedding pipeline** — the five embedding names and any data-type labels → blur or redact.
- **Josh Lee pull quote** — check it for domain terms before keeping.

Redaction bars over labels, graphics intact.

---

## 8 — Delete the executives section

**Remove `<section id="people">` entirely** (~8,900 characters):

- All five principal cards — Josh Lee, Robert Park, Zeeshan Ali, Ammar Afif, Eric Lee — with their credential rows and provenance lines.
- **Fig. 04**, the talent-scarcity funnel, which lives inside it.
- The section's lede lines and the technology/institution pairing block.
- Its CSS: `.person`, `.person-id`, `.person-body`, `.cred`, `.src`, `.ppl-lede`, `.pairing`, `.fnl*`, `.glyph`.

Then:

- Renumber the remaining sections. Current order is `§01 hero · §02 vectors · §03 results · §04 opportunity · §05 quote · §06 advisors · §07 engagements`.
- Update every `&sect;&nbsp;NN` cross-reference in the body.
- Remove the `Principals` link from the HUD nav and check no `href="#people"` remains.
- The `glyphs()` initialiser drives the per-person canvases — remove the call if nothing else uses it.

**Open question for the founder:** the Advisory section is already fully redacted and now sits alone. Cut it too, or keep it as evidence a board is forming?

---

## 9 — Fig. 02d: more numbers

The layer-response plate should feel numerical rather than purely geometric.

- Add **more visible numbers** into the figure — activation values, layer indices, magnitudes.
- Keep them small and quiet, in the existing palette.
- No NaN, no placeholder values.

---

## Preserve

- Hero tensor-field canvas animation and its four-act sequence.
- Fig. 02a original neural-network animation (660 animated edges, `flow` / `fire` / `fireOut` keyframes). **Do not replace it.**
- The plotted tensor plate and its drag-to-rotate behaviour.
- The synthesised interface click and its SND/MUTE toggle.
- One font family (`ui-monospace` / SF Mono) everywhere. Do not introduce a second family.
- Palette: near-black, bone white, warm grey, restrained signal red. No blue wash, no gradients, no glow.
- DDAP sealed to a domain hint only. Stephen Dover and Jonathan Kao must not appear anywhere.

---

## Verify before reporting done

Run all of these and report actual results, not assumptions:

- [ ] JS parses (`new Function` over the inline script)
- [ ] Every tag balanced (`section`, `figure`, `div`, `svg`, `p`, `article`, `table`, `style`, `script`)
- [ ] **Zero** occurrences of all 20 credit terms in the **served** HTML (Rule 0)
- [ ] No principal names remain: Josh Lee, Robert Park, Zeeshan Ali, Ammar Afif, Eric Lee
- [ ] No dead anchors — every `href="#…"` resolves to an existing id
- [ ] No `ddap`, `drone`, `uav`, `interceptor`, `Dover`, `Kao`
- [ ] No `NaN`, no `undefined` rendered
- [ ] Exactly one font family in the computed styles of visible elements
- [ ] No horizontal overflow at 1440px **and** 375px
- [ ] No overlapping `<text>` pairs inside any SVG figure
- [ ] Every SVG and canvas has non-zero rendered size
- [ ] Console clean
- [ ] Deployed HTML byte-identical to local via `https://tensorizeai.com/?cb=<timestamp>`

Commit only intentional files. Push to the existing private repo and let Cloudflare publish.

---

## Notes

- Source of record for factual claims is the investor deck `UniquifyAI - Antioch v3.0`. Do not invent statistics or credentials.
- Unresolved: the page states `$5.0M pre-seed, closed` while the deck says a seed round of up to $2M. The larger figure is current per the founder.
- Benchmark figures currently on the page (`+0.0136 AUC`, `67.6% vs 64.3%`, Grinsztajn/TabPFN citations) come from a separate research artifact, not the deck — and most of them are framed in credit terms, so §7 will remove or re-frame them regardless.
