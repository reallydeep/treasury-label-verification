# TTB Label Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Next.js web app where a TTB compliance agent uploads an alcohol label image plus the expected application values, and gets back a per-field Match / Review / Mismatch verdict in under 5 seconds, with a batch mode for hundreds of labels at once.

**Architecture:** One Next.js app on Vercel. A Claude vision call extracts typed fields from the label image using structured outputs. All comparison and TTB rule logic is deterministic TypeScript in pure functions, unit-tested independently of the model. The API route wires extraction to comparison; the UI is two pages (single review, batch review).

**Tech Stack:** Next.js 15 (App Router), TypeScript, `@anthropic-ai/sdk` (Claude Opus 4.8 vision + structured outputs), Zod, Vitest, JSZip, Tailwind CSS.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-20-label-verification-design.md`. Read it before starting.
- Model ID is exactly `claude-opus-4-8`. Never append a date suffix.
- Do NOT pass `temperature`, `top_p`, `top_k`, or `thinking: {type: "enabled", budget_tokens: N}` to the API. All return a 400 on this model.
- Omit the `thinking` parameter entirely on the extraction call. Opus 4.8 runs without thinking when the field is absent, which is what the 5-second latency target needs.
- The model reads the label. It never decides pass or fail. All verdict logic is deterministic TypeScript.
- Statutory government warning text (27 CFR 16.21), used verbatim as the single source of truth:
  `GOVERNMENT WARNING: (1) According to the Surgeon General, women should not drink alcoholic beverages during pregnancy because of the risk of birth defects. (2) Consumption of alcoholic beverages impairs your ability to drive a car or operate machinery, and may cause health problems.`
- Verdict values are exactly the strings `match`, `review`, `mismatch`. No other spellings.
- Field keys are exactly: `brandName`, `classType`, `alcoholContent`, `netContents`, `bottlerNameAddress`, `countryOfOrigin`.
- No persistence. No database. Images are held in memory for the duration of a request only.
- Every user-facing message is plain English. No jargon, no error codes shown to the agent.
- No em dashes in any user-facing copy or documentation.

---

### Task 1: Project scaffold and test harness

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `vitest.config.ts`, `.env.example`
- Create: `src/app/layout.tsx`, `src/app/globals.css`
- Create: `src/lib/smoke.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: a working `npm test` and `npm run dev`. Every later task adds files under `src/`.

- [ ] **Step 1: Scaffold the app**

Run from `~/Desktop/treasury`:

```bash
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --no-turbopack --yes
```

If the directory is not empty, `create-next-app` will prompt. Answer yes to proceed. The existing `docs/`, `.git/`, and `.gitignore` must be preserved.

- [ ] **Step 2: Add test and runtime dependencies**

```bash
npm install @anthropic-ai/sdk zod jszip
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react
```

- [ ] **Step 3: Configure Vitest**

Create `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    include: ['src/**/*.test.{ts,tsx}'],
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
})
```

- [ ] **Step 4: Add the test script**

In `package.json`, add to `"scripts"`:

```json
"test": "vitest run",
"test:watch": "vitest"
```

- [ ] **Step 5: Write a smoke test**

Create `src/lib/smoke.test.ts`:

```ts
import { describe, it, expect } from 'vitest'

describe('harness', () => {
  it('runs', () => {
    expect(1 + 1).toBe(2)
  })
})
```

- [ ] **Step 6: Run it**

Run: `npm test`
Expected: PASS, 1 test.

- [ ] **Step 7: Add the env example**

Create `.env.example`:

```
ANTHROPIC_API_KEY=sk-ant-...
```

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "chore: scaffold Next.js app with Vitest"
```

---

### Task 2: Core types

**Files:**
- Create: `src/lib/types.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `Verdict`, `LabelField`, `LABEL_FIELDS`, `FIELD_LABELS`, `ExtractedLabel`, `ExpectedValues`, `FieldVerdict`, `RuleVerdict`, `VerificationResult`. Every later task imports from here.

- [ ] **Step 1: Write the types file**

There is no test for this task; it is type-only and is exercised by every later task's tests. Create `src/lib/types.ts`:

```ts
export type Verdict = 'match' | 'review' | 'mismatch'

export const LABEL_FIELDS = [
  'brandName',
  'classType',
  'alcoholContent',
  'netContents',
  'bottlerNameAddress',
  'countryOfOrigin',
] as const

export type LabelField = (typeof LABEL_FIELDS)[number]

export const FIELD_LABELS: Record<LabelField, string> = {
  brandName: 'Brand name',
  classType: 'Class / type',
  alcoholContent: 'Alcohol content',
  netContents: 'Net contents',
  bottlerNameAddress: 'Bottler name and address',
  countryOfOrigin: 'Country of origin',
}

/** What the model read off the label image. */
export type ExtractedLabel = {
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

/** What the agent typed in from the application. */
export type ExpectedValues = {
  brandName?: string
  classType?: string
  alcoholContent?: string
  netContents?: string
  bottlerNameAddress?: string
  countryOfOrigin?: string
  isImported?: boolean
}

export type FieldVerdict = {
  field: LabelField
  label: string
  expected: string | null
  found: string | null
  verdict: Verdict
  /** Plain English, shown directly to the agent. */
  reason: string
}

export type RuleVerdict = {
  rule: string
  verdict: Verdict
  reason: string
}

export type VerificationResult = {
  imageLegible: boolean
  overall: Verdict
  fields: FieldVerdict[]
  rules: RuleVerdict[]
}
```

- [ ] **Step 2: Verify it compiles**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add src/lib/types.ts
git commit -m "feat: add core verification types"
```

---

### Task 3: Text normalization and edit distance

**Files:**
- Create: `src/lib/normalize.ts`
- Test: `src/lib/normalize.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `normalize(s: string): string` — casefold, collapse whitespace, normalize unicode quotes and dashes, strip punctuation.
  - `collapseWhitespace(s: string): string` — trim and collapse runs of whitespace to a single space, without casefolding or stripping punctuation.
  - `levenshtein(a: string, b: string): number`
  - `isNearMatch(a: string, b: string): boolean` — true when the normalized strings are within the length-scaled distance threshold.

- [ ] **Step 1: Write the failing tests**

Create `src/lib/normalize.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { normalize, collapseWhitespace, levenshtein, isNearMatch } from './normalize'

describe('normalize', () => {
  it('casefolds', () => {
    expect(normalize('STONE')).toBe(normalize('stone'))
  })

  it('strips apostrophes and normalizes curly quotes', () => {
    expect(normalize("STONE'S THROW")).toBe(normalize('Stone’s Throw'))
  })

  it('collapses whitespace', () => {
    expect(normalize('Old   Tom\n Distillery')).toBe('old tom distillery')
  })

  it('normalizes dashes', () => {
    expect(normalize('Ready–To–Drink')).toBe(normalize('Ready-To-Drink'))
  })

  it('strips trailing punctuation', () => {
    expect(normalize('750 mL.')).toBe(normalize('750 mL'))
  })
})

describe('collapseWhitespace', () => {
  it('keeps case and punctuation', () => {
    expect(collapseWhitespace('  GOVERNMENT   WARNING:  (1) ')).toBe(
      'GOVERNMENT WARNING: (1)',
    )
  })
})

describe('levenshtein', () => {
  it('is zero for equal strings', () => {
    expect(levenshtein('abc', 'abc')).toBe(0)
  })

  it('counts a single substitution', () => {
    expect(levenshtein('abc', 'abd')).toBe(1)
  })

  it('counts an insertion', () => {
    expect(levenshtein('abc', 'abcd')).toBe(1)
  })

  it('handles an empty string', () => {
    expect(levenshtein('', 'abc')).toBe(3)
  })
})

describe('isNearMatch', () => {
  it('accepts a one character typo in a long string', () => {
    expect(isNearMatch('Kentucky Straight Bourbon', 'Kentucky Straght Bourbon')).toBe(true)
  })

  it('rejects two unrelated words', () => {
    expect(isNearMatch('Bourbon', 'Vodka')).toBe(false)
  })

  it('accepts a one character difference in a short string', () => {
    expect(isNearMatch('Gin', 'Gim')).toBe(true)
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run src/lib/normalize.test.ts`
Expected: FAIL, cannot find module `./normalize`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/normalize.ts`:

```ts
/** Trim and collapse runs of whitespace. Preserves case and punctuation. */
export function collapseWhitespace(s: string): string {
  return s.replace(/\s+/g, ' ').trim()
}

/**
 * Aggressive normalization for comparison: casefold, unify unicode quotes and
 * dashes, drop punctuation, collapse whitespace.
 */
export function normalize(s: string): string {
  return collapseWhitespace(
    s
      .replace(/[‘’‛′]/g, "'")
      .replace(/[“”‟″]/g, '"')
      .replace(/[‐-―−]/g, '-')
      .toLowerCase()
      .replace(/[^\p{L}\p{N}\s]/gu, ' '),
  )
}

export function levenshtein(a: string, b: string): number {
  if (a === b) return 0
  if (a.length === 0) return b.length
  if (b.length === 0) return a.length

  let prev = Array.from({ length: b.length + 1 }, (_, i) => i)
  const curr = new Array<number>(b.length + 1)

  for (let i = 1; i <= a.length; i++) {
    curr[0] = i
    for (let j = 1; j <= b.length; j++) {
      const cost = a[i - 1] === b[j - 1] ? 0 : 1
      curr[j] = Math.min(curr[j - 1] + 1, prev[j] + 1, prev[j - 1] + cost)
    }
    prev = curr.slice()
  }
  return prev[b.length]
}

/**
 * Close enough that a human should look, but not close enough to call a match.
 * Threshold scales with length: 15% of the longer string, floor of 1.
 */
export function isNearMatch(a: string, b: string): boolean {
  const na = normalize(a)
  const nb = normalize(b)
  if (na === nb) return true
  const longer = Math.max(na.length, nb.length)
  const threshold = Math.max(1, Math.floor(longer * 0.15))
  return levenshtein(na, nb) <= threshold
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run src/lib/normalize.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/normalize.ts src/lib/normalize.test.ts
git commit -m "feat: add text normalization and edit distance"
```

---

### Task 4: Unit parsing for alcohol content and net contents

**Files:**
- Create: `src/lib/units.ts`
- Test: `src/lib/units.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `parseAbv(s: string): number | null` — returns percent alcohol by volume. Handles `45%`, `45.0% Alc./Vol.`, `90 Proof`, `45% Alc./Vol. (90 Proof)`.
  - `parseVolumeMl(s: string): number | null` — returns millilitres. Handles `750 mL`, `750ml`, `0.75 L`, `1.75 L`, `12 FL OZ`.

- [ ] **Step 1: Write the failing tests**

Create `src/lib/units.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { parseAbv, parseVolumeMl } from './units'

describe('parseAbv', () => {
  it('reads a bare percent', () => {
    expect(parseAbv('45%')).toBe(45)
  })

  it('reads a decimal percent with suffix', () => {
    expect(parseAbv('45.0% Alc./Vol.')).toBe(45)
  })

  it('prefers the percent when proof is also present', () => {
    expect(parseAbv('45% Alc./Vol. (90 Proof)')).toBe(45)
  })

  it('converts proof when no percent is present', () => {
    expect(parseAbv('90 Proof')).toBe(45)
  })

  it('returns null when there is no number', () => {
    expect(parseAbv('Alc./Vol.')).toBeNull()
  })
})

describe('parseVolumeMl', () => {
  it('reads millilitres', () => {
    expect(parseVolumeMl('750 mL')).toBe(750)
  })

  it('reads millilitres with no space', () => {
    expect(parseVolumeMl('750ml')).toBe(750)
  })

  it('converts litres', () => {
    expect(parseVolumeMl('0.75 L')).toBe(750)
  })

  it('converts a larger litre value', () => {
    expect(parseVolumeMl('1.75 L')).toBe(1750)
  })

  it('converts fluid ounces', () => {
    expect(parseVolumeMl('12 FL OZ')).toBeCloseTo(354.88, 1)
  })

  it('returns null when there is no unit', () => {
    expect(parseVolumeMl('750')).toBeNull()
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run src/lib/units.test.ts`
Expected: FAIL, cannot find module `./units`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/units.ts`:

```ts
const ML_PER_FL_OZ = 29.5735

/** Percent alcohol by volume, or null when no value can be read. */
export function parseAbv(s: string): number | null {
  const percent = s.match(/(\d+(?:\.\d+)?)\s*%/)
  if (percent) return Number(percent[1])

  const proof = s.match(/(\d+(?:\.\d+)?)\s*proof/i)
  if (proof) return Number(proof[1]) / 2

  return null
}

/** Volume in millilitres, or null when no value with a unit can be read. */
export function parseVolumeMl(s: string): number | null {
  const ml = s.match(/(\d+(?:\.\d+)?)\s*m\.?\s*l\.?\b/i)
  if (ml) return Number(ml[1])

  const flOz = s.match(/(\d+(?:\.\d+)?)\s*fl\.?\s*\.?\s*oz\.?/i)
  if (flOz) return Number(flOz[1]) * ML_PER_FL_OZ

  const litres = s.match(/(\d+(?:\.\d+)?)\s*l\.?\b/i)
  if (litres) return Number(litres[1]) * 1000

  return null
}
```

Note the ordering: millilitres and fluid ounces are matched before litres, because the litre pattern would otherwise match the `l` in `mL` and the `oz` string has no `l`. Keep this order.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run src/lib/units.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/units.ts src/lib/units.test.ts
git commit -m "feat: add ABV and volume unit parsing"
```

---

### Task 5: Field comparison

**Files:**
- Create: `src/lib/compare.ts`
- Test: `src/lib/compare.test.ts`

**Interfaces:**
- Consumes: `Verdict`, `LabelField`, `FIELD_LABELS`, `FieldVerdict` from `@/lib/types`; `normalize`, `isNearMatch` from `@/lib/normalize`; `parseAbv`, `parseVolumeMl` from `@/lib/units`
- Produces: `compareField(field: LabelField, expected: string | null | undefined, found: string | null): FieldVerdict`

- [ ] **Step 1: Write the failing tests**

Create `src/lib/compare.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { compareField } from './compare'

describe('compareField, text fields', () => {
  it('matches identical strings', () => {
    const r = compareField('brandName', 'Old Tom Distillery', 'Old Tom Distillery')
    expect(r.verdict).toBe('match')
  })

  it('matches when only case differs', () => {
    const r = compareField('brandName', "Stone's Throw", "STONE'S THROW")
    expect(r.verdict).toBe('match')
    expect(r.reason).toContain('Capitalization')
  })

  it('matches when only punctuation differs', () => {
    const r = compareField('brandName', 'Stones Throw', "Stone's Throw")
    expect(r.verdict).toBe('match')
  })

  it('flags a near miss for review', () => {
    const r = compareField(
      'classType',
      'Kentucky Straight Bourbon Whiskey',
      'Kentucky Straght Bourbon Whiskey',
    )
    expect(r.verdict).toBe('review')
  })

  it('reports a real conflict as a mismatch', () => {
    const r = compareField('classType', 'Bourbon Whiskey', 'Rye Whiskey')
    expect(r.verdict).toBe('mismatch')
    expect(r.reason).toContain('Bourbon Whiskey')
    expect(r.reason).toContain('Rye Whiskey')
  })

  it('reports a missing label value as a mismatch', () => {
    const r = compareField('brandName', 'Old Tom Distillery', null)
    expect(r.verdict).toBe('mismatch')
    expect(r.reason).toBe('Not found on the label.')
  })

  it('skips the field when nothing was expected', () => {
    const r = compareField('countryOfOrigin', undefined, null)
    expect(r.verdict).toBe('match')
    expect(r.reason).toBe('Not checked.')
  })
})

describe('compareField, alcohol content', () => {
  it('treats 45% and 45.0% Alc./Vol. as equal', () => {
    const r = compareField('alcoholContent', '45%', '45.0% Alc./Vol.')
    expect(r.verdict).toBe('match')
  })

  it('treats 45% and 90 Proof as equal', () => {
    const r = compareField('alcoholContent', '45%', '90 Proof')
    expect(r.verdict).toBe('match')
  })

  it('reports a real ABV difference as a mismatch', () => {
    const r = compareField('alcoholContent', '40% Alc./Vol.', '45% Alc./Vol.')
    expect(r.verdict).toBe('mismatch')
  })

  it('falls back to text comparison when no number can be read', () => {
    const r = compareField('alcoholContent', 'Alc./Vol.', 'Alc./Vol.')
    expect(r.verdict).toBe('match')
  })
})

describe('compareField, net contents', () => {
  it('treats 750 mL and 0.75 L as equal', () => {
    const r = compareField('netContents', '750 mL', '0.75 L')
    expect(r.verdict).toBe('match')
  })

  it('reports a real volume difference as a mismatch', () => {
    const r = compareField('netContents', '750 mL', '1 L')
    expect(r.verdict).toBe('mismatch')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run src/lib/compare.test.ts`
Expected: FAIL, cannot find module `./compare`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/compare.ts`:

```ts
import { FIELD_LABELS, type FieldVerdict, type LabelField } from '@/lib/types'
import { isNearMatch, normalize } from '@/lib/normalize'
import { parseAbv, parseVolumeMl } from '@/lib/units'

const ABV_TOLERANCE = 0.1
const VOLUME_TOLERANCE_ML = 1

export function compareField(
  field: LabelField,
  expected: string | null | undefined,
  found: string | null,
): FieldVerdict {
  const label = FIELD_LABELS[field]
  const exp = expected?.trim() ? expected.trim() : null
  const got = found?.trim() ? found.trim() : null

  const base = { field, label, expected: exp, found: got }

  if (exp === null) {
    return { ...base, verdict: 'match', reason: 'Not checked.' }
  }

  if (got === null) {
    return { ...base, verdict: 'mismatch', reason: 'Not found on the label.' }
  }

  if (field === 'alcoholContent') {
    const a = parseAbv(exp)
    const b = parseAbv(got)
    if (a !== null && b !== null) {
      return Math.abs(a - b) <= ABV_TOLERANCE
        ? { ...base, verdict: 'match', reason: `Both are ${a}% alcohol by volume.` }
        : {
            ...base,
            verdict: 'mismatch',
            reason: `Label says ${b}% alcohol by volume, application says ${a}%.`,
          }
    }
  }

  if (field === 'netContents') {
    const a = parseVolumeMl(exp)
    const b = parseVolumeMl(got)
    if (a !== null && b !== null) {
      return Math.abs(a - b) <= VOLUME_TOLERANCE_ML
        ? { ...base, verdict: 'match', reason: `Both are ${a} mL.` }
        : {
            ...base,
            verdict: 'mismatch',
            reason: `Label says ${b} mL, application says ${a} mL.`,
          }
    }
  }

  if (exp === got) {
    return { ...base, verdict: 'match', reason: 'Exact match.' }
  }

  if (normalize(exp) === normalize(got)) {
    return {
      ...base,
      verdict: 'match',
      reason: 'Capitalization or punctuation differs, wording is the same.',
    }
  }

  if (isNearMatch(exp, got)) {
    return {
      ...base,
      verdict: 'review',
      reason: `Close but not identical. Label says "${got}", application says "${exp}".`,
    }
  }

  return {
    ...base,
    verdict: 'mismatch',
    reason: `Label says "${got}", application says "${exp}".`,
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run src/lib/compare.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/compare.ts src/lib/compare.test.ts
git commit -m "feat: add three-state field comparison"
```

---

### Task 6: Government warning and required element rules

**Files:**
- Create: `src/lib/rules.ts`
- Test: `src/lib/rules.test.ts`

**Interfaces:**
- Consumes: `ExtractedLabel`, `ExpectedValues`, `RuleVerdict` from `@/lib/types`; `collapseWhitespace`, `normalize` from `@/lib/normalize`
- Produces:
  - `STATUTORY_WARNING: string` — the verbatim 27 CFR 16.21 text from Global Constraints.
  - `checkWarning(label: ExtractedLabel): RuleVerdict[]`
  - `checkRequiredElements(label: ExtractedLabel, expected: ExpectedValues): RuleVerdict[]`

- [ ] **Step 1: Write the failing tests**

Create `src/lib/rules.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { STATUTORY_WARNING, checkWarning, checkRequiredElements } from './rules'
import type { ExtractedLabel } from '@/lib/types'

function label(overrides: Partial<ExtractedLabel> = {}): ExtractedLabel {
  return {
    brandName: 'Old Tom Distillery',
    classType: 'Kentucky Straight Bourbon Whiskey',
    alcoholContent: '45% Alc./Vol. (90 Proof)',
    netContents: '750 mL',
    bottlerNameAddress: 'Bottled by Old Tom Distillery, Bardstown, KY',
    countryOfOrigin: null,
    governmentWarning: STATUTORY_WARNING,
    warningIsAllCapsHeader: true,
    warningLegibilityConcern: false,
    imageLegible: true,
    ...overrides,
  }
}

describe('checkWarning', () => {
  it('passes the verbatim statutory text', () => {
    const rules = checkWarning(label())
    expect(rules.every((r) => r.verdict === 'match')).toBe(true)
  })

  it('passes when only whitespace differs', () => {
    const spaced = STATUTORY_WARNING.replace(/ /g, '  ')
    const rules = checkWarning(label({ governmentWarning: spaced }))
    expect(rules.every((r) => r.verdict === 'match')).toBe(true)
  })

  it('reports altered wording as a mismatch', () => {
    const altered = STATUTORY_WARNING.replace('birth defects', 'birth problems')
    const rules = checkWarning(label({ governmentWarning: altered }))
    const wording = rules.find((r) => r.rule === 'Warning wording')
    expect(wording?.verdict).toBe('mismatch')
  })

  it('reports a missing warning as a mismatch', () => {
    const rules = checkWarning(label({ governmentWarning: null }))
    expect(rules[0].verdict).toBe('mismatch')
    expect(rules[0].reason).toContain('No government warning')
  })

  it('reports a title case header as a mismatch', () => {
    const rules = checkWarning(label({ warningIsAllCapsHeader: false }))
    const caps = rules.find((r) => r.rule === 'Warning header capitalization')
    expect(caps?.verdict).toBe('mismatch')
  })

  it('raises a review when the warning is hard to read', () => {
    const rules = checkWarning(label({ warningLegibilityConcern: true }))
    const legibility = rules.find((r) => r.rule === 'Warning legibility')
    expect(legibility?.verdict).toBe('review')
  })
})

describe('checkRequiredElements', () => {
  it('passes a complete domestic label', () => {
    const rules = checkRequiredElements(label(), {})
    expect(rules.every((r) => r.verdict === 'match')).toBe(true)
  })

  it('reports a missing required element', () => {
    const rules = checkRequiredElements(label({ netContents: null }), {})
    const missing = rules.find((r) => r.verdict === 'mismatch')
    expect(missing?.reason).toContain('Net contents')
  })

  it('requires country of origin only for imports', () => {
    const domestic = checkRequiredElements(label(), { isImported: false })
    expect(domestic.every((r) => r.verdict === 'match')).toBe(true)

    const imported = checkRequiredElements(label(), { isImported: true })
    const country = imported.find((r) => r.rule === 'Country of origin present')
    expect(country?.verdict).toBe('mismatch')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run src/lib/rules.test.ts`
Expected: FAIL, cannot find module `./rules`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/rules.ts`:

```ts
import type { ExpectedValues, ExtractedLabel, RuleVerdict } from '@/lib/types'
import { collapseWhitespace, normalize } from '@/lib/normalize'

/** 27 CFR 16.21. This exact wording is required on every alcohol beverage label. */
export const STATUTORY_WARNING =
  'GOVERNMENT WARNING: (1) According to the Surgeon General, women should not drink alcoholic beverages during pregnancy because of the risk of birth defects. (2) Consumption of alcoholic beverages impairs your ability to drive a car or operate machinery, and may cause health problems.'

export function checkWarning(labelData: ExtractedLabel): RuleVerdict[] {
  const found = labelData.governmentWarning?.trim()

  if (!found) {
    return [
      {
        rule: 'Warning present',
        verdict: 'mismatch',
        reason: 'No government warning was found on the label.',
      },
    ]
  }

  const rules: RuleVerdict[] = [
    {
      rule: 'Warning present',
      verdict: 'match',
      reason: 'The government warning is on the label.',
    },
  ]

  const wordingMatches =
    normalize(collapseWhitespace(found)) === normalize(STATUTORY_WARNING)

  rules.push(
    wordingMatches
      ? {
          rule: 'Warning wording',
          verdict: 'match',
          reason: 'The wording matches the required statement exactly.',
        }
      : {
          rule: 'Warning wording',
          verdict: 'mismatch',
          reason:
            'The wording does not match the required statement. It must be word for word.',
        },
  )

  rules.push(
    labelData.warningIsAllCapsHeader
      ? {
          rule: 'Warning header capitalization',
          verdict: 'match',
          reason: 'GOVERNMENT WARNING is in all capitals.',
        }
      : {
          rule: 'Warning header capitalization',
          verdict: 'mismatch',
          reason: 'GOVERNMENT WARNING must be in all capitals.',
        },
  )

  if (labelData.warningLegibilityConcern) {
    rules.push({
      rule: 'Warning legibility',
      verdict: 'review',
      reason:
        'The warning looks small or hard to read. Check that it meets the size requirement.',
    })
  }

  return rules
}

const REQUIRED: Array<{ key: keyof ExtractedLabel; label: string }> = [
  { key: 'brandName', label: 'Brand name' },
  { key: 'classType', label: 'Class or type' },
  { key: 'alcoholContent', label: 'Alcohol content' },
  { key: 'netContents', label: 'Net contents' },
  { key: 'bottlerNameAddress', label: 'Bottler name and address' },
]

export function checkRequiredElements(
  labelData: ExtractedLabel,
  expected: ExpectedValues,
): RuleVerdict[] {
  const rules: RuleVerdict[] = REQUIRED.map(({ key, label }) => {
    const value = labelData[key]
    const present = typeof value === 'string' && value.trim().length > 0
    return {
      rule: `${label} present`,
      verdict: present ? 'match' : 'mismatch',
      reason: present
        ? `${label} is on the label.`
        : `${label} is required and was not found on the label.`,
    }
  })

  if (expected.isImported) {
    const present = Boolean(labelData.countryOfOrigin?.trim())
    rules.push({
      rule: 'Country of origin present',
      verdict: present ? 'match' : 'mismatch',
      reason: present
        ? 'Country of origin is on the label.'
        : 'This is an import, so country of origin is required and was not found on the label.',
    })
  }

  return rules
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run src/lib/rules.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/rules.ts src/lib/rules.test.ts
git commit -m "feat: add government warning and required element rules"
```

---

### Task 7: Result assembly and overall verdict

**Files:**
- Create: `src/lib/verify.ts`
- Test: `src/lib/verify.test.ts`

**Interfaces:**
- Consumes: everything from `@/lib/types`, `compareField` from `@/lib/compare`, `checkWarning` and `checkRequiredElements` from `@/lib/rules`
- Produces: `buildResult(labelData: ExtractedLabel, expected: ExpectedValues): VerificationResult`

- [ ] **Step 1: Write the failing tests**

Create `src/lib/verify.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { buildResult } from './verify'
import { STATUTORY_WARNING } from './rules'
import type { ExtractedLabel } from '@/lib/types'

function label(overrides: Partial<ExtractedLabel> = {}): ExtractedLabel {
  return {
    brandName: 'Old Tom Distillery',
    classType: 'Kentucky Straight Bourbon Whiskey',
    alcoholContent: '45% Alc./Vol. (90 Proof)',
    netContents: '750 mL',
    bottlerNameAddress: 'Bottled by Old Tom Distillery, Bardstown, KY',
    countryOfOrigin: null,
    governmentWarning: STATUTORY_WARNING,
    warningIsAllCapsHeader: true,
    warningLegibilityConcern: false,
    imageLegible: true,
    ...overrides,
  }
}

const expected = {
  brandName: 'Old Tom Distillery',
  classType: 'Kentucky Straight Bourbon Whiskey',
  alcoholContent: '45%',
  netContents: '750 mL',
}

describe('buildResult', () => {
  it('returns match when everything agrees', () => {
    expect(buildResult(label(), expected).overall).toBe('match')
  })

  it('returns mismatch when any field conflicts', () => {
    const r = buildResult(label(), { ...expected, alcoholContent: '40%' })
    expect(r.overall).toBe('mismatch')
  })

  it('returns review when the worst result is a review', () => {
    const r = buildResult(label(), { ...expected, brandName: 'Old Tomm Distillery' })
    expect(r.overall).toBe('review')
  })

  it('lets a rule failure drive the overall verdict', () => {
    const r = buildResult(label({ warningIsAllCapsHeader: false }), expected)
    expect(r.overall).toBe('mismatch')
  })

  it('short circuits to mismatch when the image is not legible', () => {
    const r = buildResult(label({ imageLegible: false }), expected)
    expect(r.overall).toBe('mismatch')
    expect(r.imageLegible).toBe(false)
    expect(r.fields).toHaveLength(0)
    expect(r.rules[0].reason).toContain('could not be read')
  })

  it('includes one entry per label field', () => {
    expect(buildResult(label(), expected).fields).toHaveLength(6)
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run src/lib/verify.test.ts`
Expected: FAIL, cannot find module `./verify`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/verify.ts`:

```ts
import {
  LABEL_FIELDS,
  type ExpectedValues,
  type ExtractedLabel,
  type VerificationResult,
  type Verdict,
} from '@/lib/types'
import { compareField } from '@/lib/compare'
import { checkRequiredElements, checkWarning } from '@/lib/rules'

/** Worst result wins: any mismatch beats any review, which beats match. */
function worst(verdicts: Verdict[]): Verdict {
  if (verdicts.includes('mismatch')) return 'mismatch'
  if (verdicts.includes('review')) return 'review'
  return 'match'
}

export function buildResult(
  labelData: ExtractedLabel,
  expected: ExpectedValues,
): VerificationResult {
  if (!labelData.imageLegible) {
    return {
      imageLegible: false,
      overall: 'mismatch',
      fields: [],
      rules: [
        {
          rule: 'Image quality',
          verdict: 'mismatch',
          reason:
            'The label image could not be read. Ask the applicant for a clearer photo.',
        },
      ],
    }
  }

  const fields = LABEL_FIELDS.map((field) =>
    compareField(field, expected[field], labelData[field]),
  )

  const rules = [
    ...checkWarning(labelData),
    ...checkRequiredElements(labelData, expected),
  ]

  return {
    imageLegible: true,
    overall: worst([...fields.map((f) => f.verdict), ...rules.map((r) => r.verdict)]),
    fields,
    rules,
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run src/lib/verify.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Run the whole suite**

Run: `npm test`
Expected: PASS, all files.

- [ ] **Step 6: Commit**

```bash
git add src/lib/verify.ts src/lib/verify.test.ts
git commit -m "feat: assemble verification result with overall verdict"
```

---

### Task 8: Claude vision extraction

**Files:**
- Create: `src/lib/extract.ts`

**Interfaces:**
- Consumes: `ExtractedLabel` from `@/lib/types`
- Produces:
  - `extractLabel(imageBase64: string, mediaType: string): Promise<ExtractedLabel>`
  - `ExtractionError` — thrown with a plain English `message` on API failure or missing key.

This task has no unit test. It is a thin wrapper over a non-deterministic API call; asserting on model output in CI would be flaky. It is exercised by the manual fixture pass in Task 13.

- [ ] **Step 1: Write the extraction module**

Create `src/lib/extract.ts`:

```ts
import Anthropic from '@anthropic-ai/sdk'
import { z } from 'zod'
import { zodOutputFormat } from '@anthropic-ai/sdk/helpers/zod'
import type { ExtractedLabel } from '@/lib/types'
import { STATUTORY_WARNING } from '@/lib/rules'

export class ExtractionError extends Error {}

const ExtractedLabelSchema = z.object({
  brandName: z.string().nullable(),
  classType: z.string().nullable(),
  alcoholContent: z.string().nullable(),
  netContents: z.string().nullable(),
  bottlerNameAddress: z.string().nullable(),
  countryOfOrigin: z.string().nullable(),
  governmentWarning: z.string().nullable(),
  warningIsAllCapsHeader: z.boolean(),
  warningLegibilityConcern: z.boolean(),
  imageLegible: z.boolean(),
})

const SYSTEM = `You read alcohol beverage labels for the US Alcohol and Tobacco Tax and Trade Bureau.

Transcribe what is printed on the label. Do not judge compliance, do not correct errors, and do not fill in values that are not visible. Copy the text exactly as printed, including capitalization and punctuation.

Field guidance:
- brandName: the brand as printed, not the bottler name unless they are the same.
- classType: the class or type designation, for example "Kentucky Straight Bourbon Whiskey".
- alcoholContent: the full alcohol statement as printed, for example "45% Alc./Vol. (90 Proof)".
- netContents: the volume statement as printed, for example "750 mL".
- bottlerNameAddress: the bottler or producer name and address line.
- countryOfOrigin: only if a country of origin statement appears. Otherwise null.
- governmentWarning: the full warning text as printed. If any of it is unreadable, transcribe what you can read.
- warningIsAllCapsHeader: true only if the words GOVERNMENT WARNING appear in all capital letters.
- warningLegibilityConcern: true if the warning is noticeably smaller than surrounding text, low contrast, or buried.
- imageLegible: false only if the image is too blurry, dark, or obstructed to read the main label text at all. Angled or imperfect photos that you can still read are legible.

For reference, the required warning reads: ${STATUTORY_WARNING}
Do not copy that reference text into governmentWarning. Report only what you actually see on the label.

Use null for any field that is not present on the label.`

export async function extractLabel(
  imageBase64: string,
  mediaType: string,
): Promise<ExtractedLabel> {
  if (!process.env.ANTHROPIC_API_KEY) {
    throw new ExtractionError(
      'The server is missing its ANTHROPIC_API_KEY setting. Contact whoever set up this app.',
    )
  }

  const client = new Anthropic()

  try {
    const response = await client.messages.parse({
      model: 'claude-opus-4-8',
      max_tokens: 2048,
      system: SYSTEM,
      output_config: {
        format: zodOutputFormat(ExtractedLabelSchema),
        effort: 'low',
      },
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'image',
              source: {
                type: 'base64',
                media_type: mediaType as 'image/png' | 'image/jpeg' | 'image/webp',
                data: imageBase64,
              },
            },
            { type: 'text', text: 'Read this alcohol label.' },
          ],
        },
      ],
    })

    if (!response.parsed_output) {
      throw new ExtractionError('The label could not be read. Try a clearer photo.')
    }

    return response.parsed_output
  } catch (error) {
    if (error instanceof ExtractionError) throw error
    if (error instanceof Anthropic.RateLimitError) {
      throw new ExtractionError('Too many labels at once. Wait a moment and try again.')
    }
    if (error instanceof Anthropic.APIConnectionError) {
      throw new ExtractionError('Could not reach the reading service. Check the connection and try again.')
    }
    throw new ExtractionError('Something went wrong reading the label. Try again.')
  }
}
```

Note: `effort: 'low'` sits inside `output_config` alongside `format`, and the `thinking` parameter is deliberately absent. Both choices serve the 5 second target.

- [ ] **Step 2: Verify it compiles**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add src/lib/extract.ts
git commit -m "feat: add Claude vision label extraction"
```

---

### Task 9: Verification API route

**Files:**
- Create: `src/app/api/verify/route.ts`

**Interfaces:**
- Consumes: `extractLabel` and `ExtractionError` from `@/lib/extract`, `buildResult` from `@/lib/verify`, `ExpectedValues` and `VerificationResult` from `@/lib/types`
- Produces: `POST /api/verify` accepting `multipart/form-data` with an `image` file part and an `expected` JSON string part. Returns `200` with a `VerificationResult`, or a non-200 with `{ error: string }` where `error` is plain English.

- [ ] **Step 1: Write the route**

Create `src/app/api/verify/route.ts`:

```ts
import { NextResponse } from 'next/server'
import { extractLabel, ExtractionError } from '@/lib/extract'
import { buildResult } from '@/lib/verify'
import type { ExpectedValues } from '@/lib/types'

export const maxDuration = 60

const ACCEPTED = ['image/png', 'image/jpeg', 'image/webp']
const MAX_BYTES = 10 * 1024 * 1024

export async function POST(request: Request) {
  let form: FormData
  try {
    form = await request.formData()
  } catch {
    return NextResponse.json({ error: 'The upload was not readable. Try again.' }, { status: 400 })
  }

  const image = form.get('image')
  if (!(image instanceof File)) {
    return NextResponse.json({ error: 'No label image was uploaded.' }, { status: 400 })
  }

  if (!ACCEPTED.includes(image.type)) {
    return NextResponse.json(
      { error: 'That file type is not supported. Upload a PNG, JPEG, or WEBP image.' },
      { status: 400 },
    )
  }

  if (image.size > MAX_BYTES) {
    return NextResponse.json(
      { error: 'That image is larger than 10 MB. Upload a smaller file.' },
      { status: 400 },
    )
  }

  let expected: ExpectedValues = {}
  const raw = form.get('expected')
  if (typeof raw === 'string' && raw.trim()) {
    try {
      expected = JSON.parse(raw) as ExpectedValues
    } catch {
      return NextResponse.json(
        { error: 'The application values were not readable. Reload the page and try again.' },
        { status: 400 },
      )
    }
  }

  try {
    const base64 = Buffer.from(await image.arrayBuffer()).toString('base64')
    const labelData = await extractLabel(base64, image.type)
    return NextResponse.json(buildResult(labelData, expected))
  } catch (error) {
    if (error instanceof ExtractionError) {
      return NextResponse.json({ error: error.message }, { status: 502 })
    }
    return NextResponse.json(
      { error: 'Something went wrong checking that label. Try again.' },
      { status: 500 },
    )
  }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add src/app/api/verify/route.ts
git commit -m "feat: add label verification API route"
```

---

### Task 10: Client-side image downscaling

**Files:**
- Create: `src/lib/downscale.ts`
- Test: `src/lib/downscale.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `MAX_DIMENSION = 1500`
  - `scaleToFit(width: number, height: number, max?: number): { width: number; height: number }`
  - `downscaleImage(file: File): Promise<File>` — browser only. Returns the original file unchanged when it is already small enough or when canvas encoding fails.

Only `scaleToFit` is unit tested. `downscaleImage` depends on real canvas encoding, which jsdom does not implement.

- [ ] **Step 1: Write the failing test**

Create `src/lib/downscale.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { scaleToFit, MAX_DIMENSION } from './downscale'

describe('scaleToFit', () => {
  it('leaves a small image alone', () => {
    expect(scaleToFit(800, 600)).toEqual({ width: 800, height: 600 })
  })

  it('scales down by the long edge when it is the width', () => {
    expect(scaleToFit(3000, 1500)).toEqual({ width: 1500, height: 750 })
  })

  it('scales down by the long edge when it is the height', () => {
    expect(scaleToFit(1500, 3000)).toEqual({ width: 750, height: 1500 })
  })

  it('respects a custom maximum', () => {
    expect(scaleToFit(2000, 1000, 500)).toEqual({ width: 500, height: 250 })
  })

  it('exposes the default maximum', () => {
    expect(MAX_DIMENSION).toBe(1500)
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run src/lib/downscale.test.ts`
Expected: FAIL, cannot find module `./downscale`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/downscale.ts`:

```ts
export const MAX_DIMENSION = 1500

export function scaleToFit(
  width: number,
  height: number,
  max: number = MAX_DIMENSION,
): { width: number; height: number } {
  const longest = Math.max(width, height)
  if (longest <= max) return { width, height }
  const ratio = max / longest
  return { width: Math.round(width * ratio), height: Math.round(height * ratio) }
}

/**
 * Shrink an image to MAX_DIMENSION on its long edge before upload.
 * Returns the original file if it is already small enough or if anything fails.
 */
export async function downscaleImage(file: File): Promise<File> {
  try {
    const bitmap = await createImageBitmap(file)
    const { width, height } = scaleToFit(bitmap.width, bitmap.height)

    if (width === bitmap.width && height === bitmap.height) {
      bitmap.close()
      return file
    }

    const canvas = document.createElement('canvas')
    canvas.width = width
    canvas.height = height
    const context = canvas.getContext('2d')
    if (!context) return file
    context.drawImage(bitmap, 0, 0, width, height)
    bitmap.close()

    const blob = await new Promise<Blob | null>((resolve) =>
      canvas.toBlob(resolve, 'image/jpeg', 0.9),
    )
    if (!blob) return file

    return new File([blob], file.name, { type: 'image/jpeg' })
  } catch {
    return file
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run src/lib/downscale.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/downscale.ts src/lib/downscale.test.ts
git commit -m "feat: downscale images before upload"
```

---

### Task 11: Single review page

**Files:**
- Create: `src/components/VerdictBadge.tsx`
- Create: `src/components/ResultPanel.tsx`
- Create: `src/app/page.tsx`
- Modify: `src/app/layout.tsx`

**Interfaces:**
- Consumes: `VerificationResult`, `Verdict`, `ExpectedValues`, `LABEL_FIELDS`, `FIELD_LABELS` from `@/lib/types`; `downscaleImage` from `@/lib/downscale`
- Produces:
  - `VerdictBadge({ verdict }: { verdict: Verdict })`
  - `ResultPanel({ result }: { result: VerificationResult })` — reused by the batch page in Task 13.

This task has no unit test. It is presentation; it is verified by running the app in Task 14.

- [ ] **Step 1: Write the verdict badge**

Create `src/components/VerdictBadge.tsx`:

```tsx
import type { Verdict } from '@/lib/types'

const STYLES: Record<Verdict, { text: string; className: string }> = {
  match: { text: 'Match', className: 'bg-green-100 text-green-900 border-green-300' },
  review: { text: 'Review', className: 'bg-amber-100 text-amber-900 border-amber-300' },
  mismatch: { text: 'Mismatch', className: 'bg-red-100 text-red-900 border-red-300' },
}

export function VerdictBadge({ verdict }: { verdict: Verdict }) {
  const style = STYLES[verdict]
  return (
    <span
      className={`inline-block rounded border px-3 py-1 text-base font-semibold ${style.className}`}
    >
      {style.text}
    </span>
  )
}
```

- [ ] **Step 2: Write the result panel**

Create `src/components/ResultPanel.tsx`:

```tsx
import type { VerificationResult } from '@/lib/types'
import { VerdictBadge } from './VerdictBadge'

const HEADLINE = {
  match: 'Everything matches.',
  review: 'Some items need your judgment.',
  mismatch: 'Problems found.',
} as const

export function ResultPanel({ result }: { result: VerificationResult }) {
  return (
    <div className="space-y-6">
      <div className="flex items-center gap-4">
        <VerdictBadge verdict={result.overall} />
        <h2 className="text-2xl font-semibold">{HEADLINE[result.overall]}</h2>
      </div>

      {result.fields.length > 0 && (
        <section>
          <h3 className="mb-2 text-lg font-semibold">Application values</h3>
          <ul className="divide-y rounded border">
            {result.fields.map((f) => (
              <li key={f.field} className="flex items-start gap-4 p-4">
                <div className="w-28 shrink-0">
                  <VerdictBadge verdict={f.verdict} />
                </div>
                <div>
                  <div className="text-lg font-medium">{f.label}</div>
                  <div className="text-lg text-gray-700">{f.reason}</div>
                </div>
              </li>
            ))}
          </ul>
        </section>
      )}

      <section>
        <h3 className="mb-2 text-lg font-semibold">Label requirements</h3>
        <ul className="divide-y rounded border">
          {result.rules.map((r) => (
            <li key={r.rule} className="flex items-start gap-4 p-4">
              <div className="w-28 shrink-0">
                <VerdictBadge verdict={r.verdict} />
              </div>
              <div>
                <div className="text-lg font-medium">{r.rule}</div>
                <div className="text-lg text-gray-700">{r.reason}</div>
              </div>
            </li>
          ))}
        </ul>
      </section>
    </div>
  )
}
```

- [ ] **Step 3: Write the single review page**

Create `src/app/page.tsx`:

```tsx
'use client'

import { useState } from 'react'
import Link from 'next/link'
import {
  LABEL_FIELDS,
  FIELD_LABELS,
  type ExpectedValues,
  type VerificationResult,
} from '@/lib/types'
import { downscaleImage } from '@/lib/downscale'
import { ResultPanel } from '@/components/ResultPanel'

export default function SingleReviewPage() {
  const [file, setFile] = useState<File | null>(null)
  const [preview, setPreview] = useState<string | null>(null)
  const [expected, setExpected] = useState<ExpectedValues>({})
  const [result, setResult] = useState<VerificationResult | null>(null)
  const [error, setError] = useState<string | null>(null)
  const [busy, setBusy] = useState(false)
  const [seconds, setSeconds] = useState<number | null>(null)

  function pickFile(selected: File | null) {
    setFile(selected)
    setResult(null)
    setError(null)
    setPreview(selected ? URL.createObjectURL(selected) : null)
  }

  async function check() {
    if (!file) {
      setError('Choose a label image first.')
      return
    }
    setBusy(true)
    setError(null)
    setResult(null)
    const started = performance.now()

    try {
      const form = new FormData()
      form.append('image', await downscaleImage(file))
      form.append('expected', JSON.stringify(expected))

      const response = await fetch('/api/verify', { method: 'POST', body: form })
      const body = await response.json()

      if (!response.ok) {
        setError(body.error ?? 'Something went wrong. Try again.')
      } else {
        setResult(body as VerificationResult)
        setSeconds((performance.now() - started) / 1000)
      }
    } catch {
      setError('Could not reach the server. Check your connection and try again.')
    } finally {
      setBusy(false)
    }
  }

  return (
    <main className="mx-auto max-w-6xl p-8">
      <header className="mb-8 flex items-baseline justify-between">
        <h1 className="text-3xl font-bold">Check a label</h1>
        <Link href="/batch" className="text-lg text-blue-700 underline">
          Check many labels at once
        </Link>
      </header>

      <div className="grid gap-8 lg:grid-cols-2">
        <div className="space-y-6">
          <div>
            <label className="mb-2 block text-lg font-semibold" htmlFor="label-image">
              Label image
            </label>
            <input
              id="label-image"
              type="file"
              accept="image/png,image/jpeg,image/webp"
              className="block w-full rounded border p-3 text-lg"
              onChange={(e) => pickFile(e.target.files?.[0] ?? null)}
            />
          </div>

          <fieldset className="space-y-4">
            <legend className="text-lg font-semibold">What the application says</legend>
            {LABEL_FIELDS.map((field) => (
              <div key={field}>
                <label className="mb-1 block text-base" htmlFor={field}>
                  {FIELD_LABELS[field]}
                </label>
                <input
                  id={field}
                  type="text"
                  className="w-full rounded border p-3 text-lg"
                  value={expected[field] ?? ''}
                  onChange={(e) =>
                    setExpected({ ...expected, [field]: e.target.value })
                  }
                />
              </div>
            ))}
            <label className="flex items-center gap-3 text-lg">
              <input
                type="checkbox"
                className="h-5 w-5"
                checked={expected.isImported ?? false}
                onChange={(e) => setExpected({ ...expected, isImported: e.target.checked })}
              />
              This product is imported
            </label>
          </fieldset>

          <button
            type="button"
            onClick={check}
            disabled={busy}
            className="w-full rounded bg-blue-700 p-4 text-xl font-semibold text-white disabled:bg-gray-400"
          >
            {busy ? 'Checking...' : 'Check this label'}
          </button>

          {error && (
            <p className="rounded border border-red-300 bg-red-50 p-4 text-lg text-red-900">
              {error}
            </p>
          )}
        </div>

        <div className="space-y-6">
          {preview && (
            /* eslint-disable-next-line @next/next/no-img-element */
            <img src={preview} alt="The label you uploaded" className="max-h-96 rounded border" />
          )}
          {result && (
            <>
              <ResultPanel result={result} />
              {seconds !== null && (
                <p className="text-base text-gray-600">Checked in {seconds.toFixed(1)} seconds.</p>
              )}
            </>
          )}
        </div>
      </div>
    </main>
  )
}
```

- [ ] **Step 4: Set the page title**

In `src/app/layout.tsx`, replace the exported `metadata` object with:

```ts
export const metadata = {
  title: 'TTB Label Check',
  description: 'Check an alcohol label against its application.',
}
```

- [ ] **Step 5: Verify it compiles and lints**

Run: `npx tsc --noEmit && npm run lint`
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add src/app/page.tsx src/app/layout.tsx src/components
git commit -m "feat: add single label review page"
```

---

### Task 12: CSV parsing and filename matching

**Files:**
- Create: `src/lib/csv.ts`
- Test: `src/lib/csv.test.ts`

**Interfaces:**
- Consumes: `ExpectedValues` from `@/lib/types`
- Produces:
  - `CsvError` — thrown with a plain English message naming the offending row.
  - `parseExpectedCsv(text: string): Map<string, ExpectedValues>` — keyed by lowercased filename.
  - `matchFiles(names: string[], rows: Map<string, ExpectedValues>): { matched: Array<{ name: string; expected: ExpectedValues }>; imagesWithoutRow: string[]; rowsWithoutImage: string[] }`

- [ ] **Step 1: Write the failing tests**

Create `src/lib/csv.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { parseExpectedCsv, matchFiles, CsvError } from './csv'

const CSV = `filename,brandName,classType,alcoholContent,netContents,bottlerNameAddress,countryOfOrigin,isImported
a.png,Old Tom Distillery,Kentucky Straight Bourbon Whiskey,45%,750 mL,Bardstown KY,,false
b.png,Stone's Throw,Gin,40%,750 mL,Louisville KY,Scotland,true
`

describe('parseExpectedCsv', () => {
  it('reads one entry per row', () => {
    const rows = parseExpectedCsv(CSV)
    expect(rows.size).toBe(2)
    expect(rows.get('a.png')?.brandName).toBe('Old Tom Distillery')
  })

  it('parses the imported flag', () => {
    const rows = parseExpectedCsv(CSV)
    expect(rows.get('b.png')?.isImported).toBe(true)
    expect(rows.get('a.png')?.isImported).toBe(false)
  })

  it('handles quoted fields containing commas', () => {
    const csv = 'filename,brandName\nx.png,"Smith, Jones and Co"\n'
    expect(parseExpectedCsv(csv).get('x.png')?.brandName).toBe('Smith, Jones and Co')
  })

  it('lowercases the filename key', () => {
    const csv = 'filename,brandName\nMixedCase.PNG,Test\n'
    expect(parseExpectedCsv(csv).has('mixedcase.png')).toBe(true)
  })

  it('rejects a file with no filename column', () => {
    expect(() => parseExpectedCsv('brandName\nTest\n')).toThrow(CsvError)
  })

  it('names the offending row when a row is short', () => {
    const csv = 'filename,brandName,classType\na.png,Test\n'
    expect(() => parseExpectedCsv(csv)).toThrow(/row 2/)
  })

  it('rejects an empty file', () => {
    expect(() => parseExpectedCsv('')).toThrow(CsvError)
  })
})

describe('matchFiles', () => {
  it('pairs images with rows', () => {
    const rows = parseExpectedCsv(CSV)
    const out = matchFiles(['a.png', 'b.png'], rows)
    expect(out.matched).toHaveLength(2)
    expect(out.imagesWithoutRow).toEqual([])
    expect(out.rowsWithoutImage).toEqual([])
  })

  it('reports an image with no row', () => {
    const rows = parseExpectedCsv(CSV)
    const out = matchFiles(['a.png', 'c.png'], rows)
    expect(out.imagesWithoutRow).toEqual(['c.png'])
    expect(out.rowsWithoutImage).toEqual(['b.png'])
  })

  it('ignores case when pairing', () => {
    const rows = parseExpectedCsv(CSV)
    const out = matchFiles(['A.PNG', 'b.png'], rows)
    expect(out.matched).toHaveLength(2)
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run src/lib/csv.test.ts`
Expected: FAIL, cannot find module `./csv`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/csv.ts`:

```ts
import { LABEL_FIELDS, type ExpectedValues, type LabelField } from '@/lib/types'

export class CsvError extends Error {}

/** Split one CSV line, honoring double quotes and doubled escape quotes. */
function splitLine(line: string): string[] {
  const cells: string[] = []
  let cell = ''
  let inQuotes = false

  for (let i = 0; i < line.length; i++) {
    const char = line[i]
    if (inQuotes) {
      if (char === '"') {
        if (line[i + 1] === '"') {
          cell += '"'
          i++
        } else {
          inQuotes = false
        }
      } else {
        cell += char
      }
    } else if (char === '"') {
      inQuotes = true
    } else if (char === ',') {
      cells.push(cell)
      cell = ''
    } else {
      cell += char
    }
  }
  cells.push(cell)
  return cells.map((c) => c.trim())
}

export function parseExpectedCsv(text: string): Map<string, ExpectedValues> {
  const lines = text
    .split(/\r?\n/)
    .filter((line) => line.trim().length > 0)

  if (lines.length === 0) {
    throw new CsvError('The CSV file is empty.')
  }

  const header = splitLine(lines[0]).map((h) => h.toLowerCase())
  const filenameIndex = header.indexOf('filename')
  if (filenameIndex === -1) {
    throw new CsvError('The CSV file needs a column named "filename".')
  }

  const rows = new Map<string, ExpectedValues>()

  for (let i = 1; i < lines.length; i++) {
    const cells = splitLine(lines[i])
    if (cells.length !== header.length) {
      throw new CsvError(
        `CSV row ${i + 1} has ${cells.length} values but the header has ${header.length}. Fix that row and upload again.`,
      )
    }

    const filename = cells[filenameIndex]
    if (!filename) {
      throw new CsvError(`CSV row ${i + 1} has no filename.`)
    }

    const expected: ExpectedValues = {}
    for (const field of LABEL_FIELDS) {
      const index = header.indexOf(field.toLowerCase())
      if (index !== -1 && cells[index]) {
        expected[field as LabelField] = cells[index]
      }
    }

    const importedIndex = header.indexOf('isimported')
    if (importedIndex !== -1) {
      expected.isImported = /^(true|yes|y|1)$/i.test(cells[importedIndex])
    }

    rows.set(filename.toLowerCase(), expected)
  }

  return rows
}

export function matchFiles(
  names: string[],
  rows: Map<string, ExpectedValues>,
): {
  matched: Array<{ name: string; expected: ExpectedValues }>
  imagesWithoutRow: string[]
  rowsWithoutImage: string[]
} {
  const matched: Array<{ name: string; expected: ExpectedValues }> = []
  const imagesWithoutRow: string[] = []
  const used = new Set<string>()

  for (const name of names) {
    const key = name.toLowerCase()
    const expected = rows.get(key)
    if (expected) {
      matched.push({ name, expected })
      used.add(key)
    } else {
      imagesWithoutRow.push(name)
    }
  }

  const rowsWithoutImage = [...rows.keys()].filter((key) => !used.has(key))

  return { matched, imagesWithoutRow, rowsWithoutImage }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run src/lib/csv.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/csv.ts src/lib/csv.test.ts
git commit -m "feat: add CSV parsing and filename matching"
```

---

### Task 13: Batch review page

**Files:**
- Create: `src/lib/pool.ts`
- Test: `src/lib/pool.test.ts`
- Create: `src/app/batch/page.tsx`

**Interfaces:**
- Consumes: `parseExpectedCsv`, `matchFiles`, `CsvError` from `@/lib/csv`; `downscaleImage` from `@/lib/downscale`; `ResultPanel` from `@/components/ResultPanel`; `VerdictBadge` from `@/components/VerdictBadge`; `VerificationResult` from `@/lib/types`; `JSZip` from `jszip`
- Produces: `runPool<T, R>(items: T[], limit: number, worker: (item: T, index: number) => Promise<R>, onDone: (index: number, result: R) => void): Promise<void>`

- [ ] **Step 1: Write the failing pool test**

Create `src/lib/pool.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { runPool } from './pool'

describe('runPool', () => {
  it('processes every item', async () => {
    const seen: number[] = []
    await runPool([1, 2, 3, 4, 5], 2, async (n) => n * 2, (_, r) => seen.push(r))
    expect(seen.sort((a, b) => a - b)).toEqual([2, 4, 6, 8, 10])
  })

  it('reports results with the right index', async () => {
    const out: Array<[number, string]> = []
    await runPool(['a', 'b', 'c'], 2, async (s) => s.toUpperCase(), (i, r) => out.push([i, r]))
    expect(out.sort((x, y) => x[0] - y[0])).toEqual([
      [0, 'A'],
      [1, 'B'],
      [2, 'C'],
    ])
  })

  it('never exceeds the concurrency limit', async () => {
    let active = 0
    let peak = 0
    await runPool(
      Array.from({ length: 20 }, (_, i) => i),
      3,
      async () => {
        active++
        peak = Math.max(peak, active)
        await new Promise((r) => setTimeout(r, 5))
        active--
        return null
      },
      () => {},
    )
    expect(peak).toBeLessThanOrEqual(3)
  })

  it('handles an empty list', async () => {
    await expect(runPool([], 3, async () => null, () => {})).resolves.toBeUndefined()
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run src/lib/pool.test.ts`
Expected: FAIL, cannot find module `./pool`.

- [ ] **Step 3: Write the pool**

Create `src/lib/pool.ts`:

```ts
/**
 * Run `worker` over `items` with at most `limit` in flight.
 * `onDone` fires as each item finishes, so the UI can update incrementally.
 * The worker is expected to catch its own errors and return a result value.
 */
export async function runPool<T, R>(
  items: T[],
  limit: number,
  worker: (item: T, index: number) => Promise<R>,
  onDone: (index: number, result: R) => void,
): Promise<void> {
  let next = 0

  async function drain(): Promise<void> {
    while (next < items.length) {
      const index = next++
      const result = await worker(items[index], index)
      onDone(index, result)
    }
  }

  await Promise.all(
    Array.from({ length: Math.min(limit, items.length) }, () => drain()),
  )
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run src/lib/pool.test.ts`
Expected: PASS, all tests.

- [ ] **Step 5: Write the batch page**

Create `src/app/batch/page.tsx`:

```tsx
'use client'

import { useState } from 'react'
import Link from 'next/link'
import JSZip from 'jszip'
import { CsvError, matchFiles, parseExpectedCsv } from '@/lib/csv'
import { downscaleImage } from '@/lib/downscale'
import { runPool } from '@/lib/pool'
import { ResultPanel } from '@/components/ResultPanel'
import { VerdictBadge } from '@/components/VerdictBadge'
import type { ExpectedValues, VerificationResult } from '@/lib/types'

const CONCURRENCY = 8
const IMAGE_TYPES: Record<string, string> = {
  png: 'image/png',
  jpg: 'image/jpeg',
  jpeg: 'image/jpeg',
  webp: 'image/webp',
}

type Row = {
  name: string
  status: 'waiting' | 'checking' | 'done' | 'failed'
  result?: VerificationResult
  error?: string
}

const RANK = { mismatch: 0, review: 1, match: 2 } as const

export default function BatchPage() {
  const [zipFile, setZipFile] = useState<File | null>(null)
  const [csvFile, setCsvFile] = useState<File | null>(null)
  const [rows, setRows] = useState<Row[]>([])
  const [skipped, setSkipped] = useState<{ images: string[]; csvRows: string[] }>({
    images: [],
    csvRows: [],
  })
  const [error, setError] = useState<string | null>(null)
  const [busy, setBusy] = useState(false)
  const [openRow, setOpenRow] = useState<number | null>(null)

  async function run() {
    if (!zipFile || !csvFile) {
      setError('Choose both a ZIP of label images and a CSV of application values.')
      return
    }

    setBusy(true)
    setError(null)
    setRows([])
    setOpenRow(null)

    let expectedByName: Map<string, ExpectedValues>
    try {
      expectedByName = parseExpectedCsv(await csvFile.text())
    } catch (e) {
      setError(e instanceof CsvError ? e.message : 'The CSV file could not be read.')
      setBusy(false)
      return
    }

    let entries: Array<{ name: string; file: File }>
    try {
      const zip = await JSZip.loadAsync(zipFile)
      entries = []
      for (const entry of Object.values(zip.files)) {
        if (entry.dir) continue
        const name = entry.name.split('/').pop() ?? entry.name
        if (name.startsWith('.')) continue
        const ext = name.split('.').pop()?.toLowerCase() ?? ''
        const type = IMAGE_TYPES[ext]
        if (!type) continue
        const blob = await entry.async('blob')
        entries.push({ name, file: new File([blob], name, { type }) })
      }
    } catch {
      setError('That ZIP file could not be opened. Check the file and try again.')
      setBusy(false)
      return
    }

    if (entries.length === 0) {
      setError('The ZIP file has no PNG, JPEG, or WEBP images in it.')
      setBusy(false)
      return
    }

    const { matched, imagesWithoutRow, rowsWithoutImage } = matchFiles(
      entries.map((e) => e.name),
      expectedByName,
    )
    setSkipped({ images: imagesWithoutRow, csvRows: rowsWithoutImage })

    const byName = new Map(entries.map((e) => [e.name, e.file]))
    const work = matched.map((m) => ({ ...m, file: byName.get(m.name)! }))

    setRows(work.map((w) => ({ name: w.name, status: 'waiting' as const })))

    await runPool(
      work,
      CONCURRENCY,
      async (item, index) => {
        setRows((prev) => {
          const copy = [...prev]
          copy[index] = { ...copy[index], status: 'checking' }
          return copy
        })
        try {
          const form = new FormData()
          form.append('image', await downscaleImage(item.file))
          form.append('expected', JSON.stringify(item.expected))
          const response = await fetch('/api/verify', { method: 'POST', body: form })
          const body = await response.json()
          if (!response.ok) return { error: body.error as string }
          return { result: body as VerificationResult }
        } catch {
          return { error: 'Could not reach the server for this label.' }
        }
      },
      (index, outcome) => {
        setRows((prev) => {
          const copy = [...prev]
          copy[index] =
            'result' in outcome && outcome.result
              ? { ...copy[index], status: 'done', result: outcome.result }
              : { ...copy[index], status: 'failed', error: outcome.error }
          return copy
        })
      },
    )

    setBusy(false)
  }

  const done = rows.filter((r) => r.status === 'done' || r.status === 'failed').length
  const ordered = rows
    .map((row, index) => ({ row, index }))
    .sort((a, b) => {
      const rank = (r: Row) => (r.result ? RANK[r.result.overall] : r.error ? -1 : 3)
      return rank(a.row) - rank(b.row)
    })

  return (
    <main className="mx-auto max-w-6xl p-8">
      <header className="mb-8 flex items-baseline justify-between">
        <h1 className="text-3xl font-bold">Check many labels</h1>
        <Link href="/" className="text-lg text-blue-700 underline">
          Check one label
        </Link>
      </header>

      <div className="mb-8 space-y-6">
        <div>
          <label className="mb-2 block text-lg font-semibold" htmlFor="zip">
            ZIP file of label images
          </label>
          <input
            id="zip"
            type="file"
            accept=".zip"
            className="block w-full rounded border p-3 text-lg"
            onChange={(e) => setZipFile(e.target.files?.[0] ?? null)}
          />
        </div>

        <div>
          <label className="mb-2 block text-lg font-semibold" htmlFor="csv">
            CSV file of application values
          </label>
          <input
            id="csv"
            type="file"
            accept=".csv,text/csv"
            className="block w-full rounded border p-3 text-lg"
            onChange={(e) => setCsvFile(e.target.files?.[0] ?? null)}
          />
          <p className="mt-2 text-base text-gray-600">
            The CSV needs a column named filename that matches the image file names in the ZIP.
          </p>
        </div>

        <button
          type="button"
          onClick={run}
          disabled={busy}
          className="w-full rounded bg-blue-700 p-4 text-xl font-semibold text-white disabled:bg-gray-400"
        >
          {busy ? `Checking ${done} of ${rows.length}...` : 'Check all labels'}
        </button>

        {error && (
          <p className="rounded border border-red-300 bg-red-50 p-4 text-lg text-red-900">
            {error}
          </p>
        )}

        {skipped.images.length > 0 && (
          <p className="rounded border border-amber-300 bg-amber-50 p-4 text-lg">
            Skipped, no matching CSV row: {skipped.images.join(', ')}
          </p>
        )}
        {skipped.csvRows.length > 0 && (
          <p className="rounded border border-amber-300 bg-amber-50 p-4 text-lg">
            Skipped, no matching image: {skipped.csvRows.join(', ')}
          </p>
        )}
      </div>

      {rows.length > 0 && (
        <ul className="divide-y rounded border">
          {ordered.map(({ row, index }) => (
            <li key={row.name}>
              <button
                type="button"
                className="flex w-full items-center gap-4 p-4 text-left"
                onClick={() => setOpenRow(openRow === index ? null : index)}
              >
                <span className="w-28 shrink-0">
                  {row.result ? (
                    <VerdictBadge verdict={row.result.overall} />
                  ) : (
                    <span className="text-base text-gray-600">
                      {row.status === 'failed' ? 'Failed' : row.status === 'checking' ? 'Checking' : 'Waiting'}
                    </span>
                  )}
                </span>
                <span className="text-lg font-medium">{row.name}</span>
              </button>
              {openRow === index && (
                <div className="border-t bg-gray-50 p-6">
                  {row.result && <ResultPanel result={row.result} />}
                  {row.error && <p className="text-lg text-red-900">{row.error}</p>}
                </div>
              )}
            </li>
          ))}
        </ul>
      )}
    </main>
  )
}
```

- [ ] **Step 6: Verify it compiles and lints**

Run: `npx tsc --noEmit && npm run lint`
Expected: no errors.

- [ ] **Step 7: Commit**

```bash
git add src/lib/pool.ts src/lib/pool.test.ts src/app/batch/page.tsx
git commit -m "feat: add batch label review page"
```

---

### Task 14: Fixtures, manual verification, README, deploy

**Files:**
- Create: `fixtures/README.md`
- Create: `fixtures/expected.csv`
- Create: `README.md` (replace the create-next-app default)

**Interfaces:**
- Consumes: the whole app
- Produces: the two deliverables from the spec, a public repo with documentation and a live URL.

- [ ] **Step 1: Run the whole test suite**

Run: `npm test`
Expected: PASS, every test file. Record the total count for the README.

- [ ] **Step 2: Create the fixture set**

Create `fixtures/README.md`:

```markdown
# Test labels

Generate six label images with any AI image tool and save them here as PNG:

1. `clean.png` — a straight, well lit "OLD TOM DISTILLERY" bourbon label with all
   required elements and the correct government warning.
2. `angled.png` — the same label photographed at an angle.
3. `glare.png` — the same label with glare across part of the text.
4. `wrong-abv.png` — the same label but printed at 40% Alc./Vol.
5. `altered-warning.png` — the same label with the warning reworded.
6. `titlecase-warning.png` — the same label with "Government Warning:" in title case.

`expected.csv` in this folder holds the application values for all six. Every row
states 45% alcohol and the correct warning, so labels 4, 5, and 6 are expected to
come back as mismatches.
```

Create `fixtures/expected.csv`:

```csv
filename,brandName,classType,alcoholContent,netContents,bottlerNameAddress,countryOfOrigin,isImported
clean.png,OLD TOM DISTILLERY,Kentucky Straight Bourbon Whiskey,45% Alc./Vol.,750 mL,"Bottled by Old Tom Distillery, Bardstown, KY",,false
angled.png,OLD TOM DISTILLERY,Kentucky Straight Bourbon Whiskey,45% Alc./Vol.,750 mL,"Bottled by Old Tom Distillery, Bardstown, KY",,false
glare.png,OLD TOM DISTILLERY,Kentucky Straight Bourbon Whiskey,45% Alc./Vol.,750 mL,"Bottled by Old Tom Distillery, Bardstown, KY",,false
wrong-abv.png,OLD TOM DISTILLERY,Kentucky Straight Bourbon Whiskey,45% Alc./Vol.,750 mL,"Bottled by Old Tom Distillery, Bardstown, KY",,false
altered-warning.png,OLD TOM DISTILLERY,Kentucky Straight Bourbon Whiskey,45% Alc./Vol.,750 mL,"Bottled by Old Tom Distillery, Bardstown, KY",,false
titlecase-warning.png,OLD TOM DISTILLERY,Kentucky Straight Bourbon Whiskey,45% Alc./Vol.,750 mL,"Bottled by Old Tom Distillery, Bardstown, KY",,false
```

- [ ] **Step 3: Run the app and verify against the fixtures**

```bash
echo "ANTHROPIC_API_KEY=<the key>" > .env.local
npm run dev
```

Open `http://localhost:3000` and check each fixture image, entering the values from `expected.csv`. Record for the README:

- Whether each of the six produced the expected overall verdict.
- The reported "Checked in N seconds" value for `clean.png` across at least five runs. Note the median and the slowest.

Then open `http://localhost:3000/batch`, zip the six fixtures, upload the zip and `fixtures/expected.csv`, and confirm all six rows resolve with problems sorted to the top.

If any fixture produces an unexpected verdict, fix the cause before continuing and note it in the README limitations if it cannot be fixed.

- [ ] **Step 4: Write the README**

Replace `README.md` with the following, filling in the bracketed values from Step 1 and Step 3:

````markdown
# TTB Label Verification

A prototype that checks an alcohol beverage label image against the values on its
COLA application, and against the TTB rules that apply to every label.

Live: [deployed URL]

## What it does

- **Check one label.** Upload a label image, type the values from the application,
  get a per-field verdict beside the image.
- **Check many labels.** Upload a ZIP of label images plus a CSV of application
  values, matched by filename. Results stream in as they finish, with problems
  sorted to the top.
- **Three verdicts, not two.** Match, Review, Mismatch. A brand name that reads
  `STONE'S THROW` on the label and `Stone's Throw` on the application is a Match,
  not a rejection. Something close but not identical is a Review with the reason
  stated. Only a real conflict is a Mismatch.
- **Exact warning check.** The government warning is compared word for word against
  the 27 CFR 16.21 text, and `GOVERNMENT WARNING:` must be in all capitals.

## Setup

Requires Node 20 or later and an Anthropic API key.

```bash
npm install
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env.local
npm run dev
```

Open http://localhost:3000.

```bash
npm test          # run the unit tests
npm run build     # production build
```

## Approach

The Claude vision call does one job: read the label and return typed fields. It
never decides pass or fail. Every verdict comes from deterministic TypeScript in
`src/lib/`, which means the decision logic is unit tested and the same extracted
values always produce the same verdict.

| File | Responsibility |
|---|---|
| `src/lib/extract.ts` | The one Claude vision call, with a schema-constrained response |
| `src/lib/compare.ts` | Per-field Match / Review / Mismatch |
| `src/lib/normalize.ts` | Casefolding, punctuation, edit distance |
| `src/lib/units.ts` | ABV and volume parsing so 45%, 45.0%, and 90 Proof agree |
| `src/lib/rules.ts` | Government warning and required element checks |
| `src/lib/verify.ts` | Assembles the result and picks the overall verdict |
| `src/lib/csv.ts` | Batch CSV parsing and filename matching |
| `src/lib/pool.ts` | Bounded concurrency for batch runs |

## Measured latency

Single label, end to end, on the clean fixture: median [N] seconds, slowest of
five runs [N] seconds. The target was under 5 seconds.

This comes from three choices: one model call per label, client-side downscaling
to 1500px on the long edge before upload, and running the extraction without
extended thinking at low effort. The reading task does not need deliberation.

Batch mode runs 8 labels in parallel with per-row results appearing as they land.

## Tools used

Next.js 15, TypeScript, Tailwind CSS, Claude Opus 4.8 for vision extraction with
structured outputs, Zod for the response schema, Vitest for tests, JSZip for batch
uploads, Vercel for hosting.

## Assumptions

- No authentication and no persistence. Uploaded images live in memory for the
  duration of one request and are never written to disk or a database.
- The statutory warning text used is the standard 27 CFR 16.21 wording.
- Beverage type specific rules are out of scope. The common required elements
  apply to all types.
- Country of origin is only required when the application marks the product as
  imported.

## Trade-offs and limitations

- **Extraction is not deterministic.** The comparison logic is, so the same
  extracted values always yield the same verdict, but extraction itself can vary
  between runs on a hard image.
- **No history and no audit trail.** A 300 label batch that is interrupted has to
  start over. Persistence was traded away to keep the prototype free of any data
  retention question.
- **The firewall constraint is documented, not designed around.** Marcus described
  a network that blocks outbound calls to ML endpoints. A cloud vision API cannot
  satisfy that. Designing for it would mean local OCR, which fails the requirement
  to handle angled and glare-affected photos. An on-premises version would need a
  self-hosted vision model, which is a procurement decision, not a prototype one.
- **Batch throughput is bounded by API rate limits**, not by application code.
- **Rules cover common required elements only.** A class specific requirement such
  as a wine appellation rule is not enforced.

## Test results against the fixture set

[Table of the six fixtures and their actual verdicts from Step 3.]
````

- [ ] **Step 5: Commit and push**

```bash
git add -A
git commit -m "docs: add README, fixtures, and measured latency"
git push
```

- [ ] **Step 6: Deploy to Vercel**

```bash
npx vercel --prod
```

When prompted, link to a new project. Then set the API key and redeploy:

```bash
npx vercel env add ANTHROPIC_API_KEY production
npx vercel --prod
```

- [ ] **Step 7: Verify the deployment**

Open the production URL. Check `clean.png` through the single review page and
confirm it returns a Match. Run one batch of the six fixtures.

- [ ] **Step 8: Record the URL and push**

Replace `[deployed URL]` in `README.md` with the real URL.

```bash
git add README.md
git commit -m "docs: add deployed URL"
git push
```
