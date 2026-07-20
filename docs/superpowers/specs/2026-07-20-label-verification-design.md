# AI-Powered Alcohol Label Verification: Design

Date: 2026-07-20
Status: Approved

## Problem

TTB reviews ~150,000 label applications a year with 47 agents. Much of the work is
mechanical comparison: does the brand name on the label artwork match the brand name on
the application, is the ABV the same number, is the government warning present and
verbatim. Agents spend roughly half their day on this.

This prototype automates the comparison and surfaces only what needs human judgment.

## Constraints (from stakeholder interviews)

| Constraint | Source | Design response |
|---|---|---|
| Results in under ~5 seconds | Sarah Chen; a prior vendor pilot failed at 30-40s | One model call per label, client-side image downscale, measured and reported |
| Usable by a non-technical 73-year-old | Sarah Chen | One primary action per screen, large type, plain-English verdicts, no settings |
| Batch upload of 200-300 labels | Sarah Chen / Janet, Seattle | ZIP + CSV batch mode with bounded-concurrency processing and live progress |
| Government warning must be exact | Jenny Park | Dedicated rule check: verbatim statutory text plus `GOVERNMENT WARNING:` in caps |
| Case/punctuation differences are not real mismatches | Dave Morrison | Three-state verdict with a Review tier |
| Imperfect photos: angle, glare, low light | Jenny Park | Vision model rather than OCR; unreadable images return a result, not an error |
| Standalone prototype, no COLA integration, no sensitive data stored | Marcus Williams | No database, no persistence, stateless request handling |

## Scope

In scope: single-label review, batch review, field extraction, comparison verdicts,
TTB rule checks, deployed URL, documentation.

Out of scope: COLA integration, authentication, persistence, audit logging, mobile
layout, beverage-type-specific rule variation beyond the common required elements.

## Architecture

A single Next.js application deployed to Vercel.

Surfaces:

- `/` Single review. Image upload plus expected-field form. Results render beside the image.
- `/batch` Batch review. ZIP of images plus a CSV of expected values, matched by filename.
- `/api/verify` POST endpoint. Accepts one image and its expected fields, returns a verdict object.

The Claude vision call extracts label fields against a defined output schema, so the
response is typed data rather than prose requiring parsing. The model reads the label.
It does not decide pass or fail. All comparison and rule logic is deterministic
TypeScript, which keeps the decision auditable and unit-testable.

### Data flow, single label

```
image ──────────► /api/verify ──► Claude vision (extract) ──► ExtractedLabel
                                                                   │
expected fields ───────────────────────────────────────► compare() │
                                                                   ▼
                                              TTB rule checks (warning, required elements)
                                                                   ▼
                                                            overall verdict
```

### Core types

```ts
type ExtractedLabel = {
  brandName: string | null
  classType: string | null
  alcoholContent: string | null
  netContents: string | null
  bottlerNameAddress: string | null
  countryOfOrigin: string | null
  governmentWarning: string | null
  warningIsAllCapsHeader: boolean
  warningLegibilityConcern: boolean
  imageLegible: boolean
}

type Verdict = 'match' | 'review' | 'mismatch'

type FieldVerdict = {
  field: string
  expected: string | null
  found: string | null
  verdict: Verdict
  reason: string   // plain English, shown directly to the agent
}
```

## Comparison rules

`compare(expected, found)` per field:

1. Normalize both sides: trim, collapse whitespace, casefold, strip punctuation,
   normalize unicode quotes and dashes.
2. Normalized strings equal, and raw strings differ only in case or punctuation:
   **Match**. This resolves the `STONE'S THROW` versus `Stone's Throw` case.
3. Close but not equal (Levenshtein distance within a threshold scaled to string
   length): **Review**, with a reason naming the difference.
4. Otherwise: **Mismatch**.
5. Expected value present but field absent from the label: **Mismatch**, reason
   "Not found on label".

Alcohol content is compared numerically, not textually. `45% Alc./Vol.`, `45.0%`, and
`90 Proof` all resolve to 45.0 and are a Match. A difference within 0.1 is a Match,
otherwise Mismatch.

Net contents is normalized across units (750 mL, 750ml, 0.75 L) before comparison.

## TTB rule checks

Independent of the application comparison:

- Government warning present.
- Warning text matches the statutory wording verbatim after whitespace normalization.
  Any wording deviation is a Mismatch, not a Review, per Jenny Park.
- `GOVERNMENT WARNING:` appears in all caps. Title case is a Mismatch.
- Model-reported legibility concern (small or buried warning) raises a Review.
- Required elements present: brand name, class/type, alcohol content, net contents,
  bottler name and address. Country of origin is checked only when the application
  marks the product as imported.

## Overall verdict

Worst field result wins. Any Mismatch makes the label a Mismatch. Otherwise any Review
makes it a Review. Otherwise Match.

## Latency

Target: under 5 seconds end to end for a single label.

- Images downscaled client-side to a max dimension of ~1500px before upload.
- Exactly one model call per label.
- Batch mode uses a bounded concurrency pool of ~8, with per-row results streaming into
  the table as they complete and a live progress indicator.
- Measured p50 and p95 for single-label review are recorded in the README.

## User experience

Designed for the least technical agent, not the most.

- One primary action per screen.
- Large type, high contrast, no dense tables in single-review mode.
- Verdicts as colored rows with a plain sentence: "Label says 45% Alc./Vol., application
  says 40% Alc./Vol."
- Image and results visible together without scrolling on a standard laptop.
- Batch results table sorts problems to the top; clicking a row opens that label's
  detail view.
- No settings screen, no configuration, no jargon.

## Error handling

Every failure gets a specific message and a clear next step.

| Condition | Behavior |
|---|---|
| Unsupported file type | Rejected at selection with the accepted list named |
| File too large | Rejected with the limit stated |
| Image not legible | Returns a result of "Image not legible", suggests requesting a better photo. Not an error. |
| Model timeout or API error | Row marked Failed with a retry action. Batch continues. |
| Missing API key | Clear startup and runtime message naming the variable |
| Malformed CSV | Named parse error with the offending row number |
| CSV row with no matching image | Listed as skipped, with filenames |
| Image with no matching CSV row | Listed as skipped, with filenames |

## Testing

- Unit tests on `compare()`: exact match, case difference, punctuation difference,
  near-miss, real conflict, missing field, ABV numeric equivalence across formats, net
  contents unit normalization.
- Unit tests on the warning rule checker: verbatim pass, altered wording, title-case
  header, missing warning.
- Unit tests on CSV parsing and filename matching, including the mismatch cases above.
- Fixture label set: clean, angled, glare, wrong ABV, altered warning, title-case
  warning.
- Vision extraction accuracy is verified by a documented manual pass over the fixture
  set rather than a CI assertion, since model output is not deterministic.

## Assumptions

- Prototype has no authentication and stores nothing. Uploaded images are held in memory
  for the duration of the request only.
- The statutory warning text used is the standard 27 CFR 16.21 wording.
- Beverage-type-specific labeling variation is out of scope. The common required
  elements apply to all types.
- The firewall constraint Marcus described applies to their internal network, not to
  this externally deployed prototype. It is documented as a limitation with a note on
  what an on-premises deployment would require.

## Trade-offs and limitations

- Model-based extraction is not deterministic. Comparison logic is deterministic, so the
  same extracted values always yield the same verdict, but extraction itself can vary
  between runs on a hard image.
- Batch throughput is bounded by API rate limits, not by application code.
- No persistence means no history, no audit trail, and no resuming an interrupted batch.
- Rules cover the common required elements only, so a class/type-specific requirement
  such as an appellation rule for wine is not enforced.

## Deliverables

1. Public GitHub repository under `reallydeep` with all source, a README covering setup,
   run instructions, approach, tools, assumptions, and trade-offs.
2. Deployed Vercel URL.
