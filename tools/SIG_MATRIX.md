<div align="center">

<h1 align="center">OreUI signature matrix</h1>

<p>6 samples × 10 signatures, measured</p>

<p>
<b>English</b> · <a href="./SIG_MATRIX.zh-CN.md">中文</a>
</p>

</div>

<br/>

All samples are official, unmodified arm64-v8a `libminecraftpe.so`, scanned byte-wise with
`scan.py`; each hit was then walked back to check whether it lands on a function entry.

| Version → | 1.20.10.01 | 1.21.0.03 | 1.21.90.3 | 1.21.132.1 | 1.26.0.2 | 1.26.62.1 |
|---|---|---|---|---|---|---|
| `1.26.60` | — | — | — | — | — | ✅ **0x9AF937C** (entry) |
| `1.26.50` | — | — | — | — | — | — |
| `1.26.40` | — | — | — | — | — | — |
| `1.26.30` | — | — | — | — | — | — |
| `H8 1.26.20` | — | — | — | ✅ 0x907E4BC (mid) | ✅ 0x913DF54 (mid) | — |
| `H4 1.21.130~1.26.0` | — | — | — | — | ✅ 0x8F7452C (mid) | — |
| `H5 1.26.0~1.26.10` | — | — | — | ✅ 0x8C496D8 (mid) | — | — |
| `H2 1.21.90~1.21.120` | ✅ **0x56DBB0C** (entry) | ✅ 0x5F85100 (entry) | ✅ 0x5EC9928 (entry) | — | — | — |
| `H3 1.21.120` | — | — | — | — | — | — |
| `H7 1.26.10` | — | — | — | — | ✅ 0x8F7452C (mid) | — |

> **This table is historical.** It records what the original 10 signatures did across the six
> samples. `scan.py` deliberately keeps all 10 so you can reproduce these results. The shipped
> mod currently carries only 5 — see "Changes to the shipped signature table" below.

## Key findings

### 1. H2 is the most stable signature across versions

H2 hits on **1.20.10.01, 1.21.0.03 and 1.21.90.3** — all three, and **all on a function entry**
(preceded by `ret`). It is the only one of the 10 that spans 1.20→1.21 *and* lands on a function
entry every time.

### 2. 1.20.10.01 already has an OreUI runtime, but a different one

H2 hits `0x56DBB0C` there, and that function body references:

```
/gameplay/death      /hbui/gameplay.html      JsException      ScreenLoadFailed
core:routing:not-found      core:gui:resize-hack      core:exception
core:telemetry:firstContentfulPaint      core:telemetry:firstMeaningfulPaint
core:keyboard:dismissed      Play      Edit World
```

Whereas the 1.26.62.1 target references specific screen routes like
`/oreui-marketplace/search` and `/send-invites`. That means:

- **1.20 era:** only the HBUI **runtime core** (routing table, exceptions, telemetry, keyboard);
  screens were still JSON-UI.
- **1.26 era:** a large number of **OreUI screen registrations** on top of that.

So "removing OreUI on 1.20" would mean handling a different function; the 1.26 hook target
cannot simply be reused.

### 3. Whether the hit lands on the entry decides whether the hook works at all

| On function entry? | Versions |
|---|---|
| ✅ yes | 1.20.10.01, 1.21.0.03, 1.21.90.3, 1.26.62.1 |
| ❌ no (mid-function, previous insn is `bl`) | 1.21.132.1, 1.26.0.2 |

`DobbyHook` / `pl::hook` on a mid-function address behaves unpredictably — it overwrites a
called instruction or corrupts the stack frame. **On those 2 versions the existing hook is
most likely outright broken**, not merely "works but poorly".

### 4. Several signatures hitting the same address on one version = redundant

On 1.26.0.2, `H8`, `H4` and `H7` all point at `0x8F7452C`. Keeping three just wastes scan time.

## Changes to the shipped signature table

**Dropped** (with reasons):

- `H3` — zero hits across all 6 samples; the claimed "1.21.120" is actually covered by H2.
- `H5` — claimed "1.26.0~1.26.10" but missed 1.26.0.2, and hit 1.21.132.1 instead. The label
  is not trustworthy.
- `H7` — on 1.26.0.2 it hits the same address as `H4`; fully equivalent.
- `1.26.50` / `1.26.40` / `1.26.30` — zero hits across all 6 samples, added early per-version
  with no measured evidence. Removed in a later pass; see below.

**Demoted:**

- `H8` — the hit is not on a function entry, so it is kept only as a last resort.

**Added:**

- A dedicated `1.21.0.03` signature, derived with `derive_sig.py`
  (tail `F4 03 04 AA E4 03 03 AA … E1 03 00 AA`).

**Shipped order after the later cleanup (5 entries):**

`1.26.60 → H8 → H4 → H2 → 1.21.0.03`

> The three per-version entries (`1.26.50` / `1.26.40` / `1.26.30`) were removed from the mod's
> table because they never hit any of the six samples. They remain in `scan.py` so this matrix
> stays reproducible. Note that the versions they *claim* to target were never part of the
> sample set — their validity for those versions is genuinely unknown, not disproven. If those
> builds ever show up, re-derive with `derive_sig.py` rather than resurrecting them.

## Missing samples

| Gap | Impact |
|---|---|
| Before 1.20.10 (1.19.x) | Cannot confirm where OreUI actually begins |
| 1.21.120.4 | Whether H3 was truly redundant can only be inferred indirectly |
| 1.26.10.4 | H7's label cannot be verified directly |
| 1.26.30.5 / 1.26.40.5 / 1.26.50.4 | These three never hit any sample at all |

## Note: OreUI grew incrementally, it is not a switch

Since 1.20, screens have been migrating from JSON-UI to OreUI one at a time. So "removing
OreUI" is not a one-shot toggle — it has to be handled **per version, per screen**:

| Version | State |
|---|---|
| 1.20.10.01 | HBUI/OreUI runtime core present, UI still mostly JSON-UI |
| 1.21.x | OreUI screens appear one by one |
| 1.26.x | Almost every screen is OreUI |

On 1.20.10.01 the signature hits `0x56DBB0C` (a function entry), but that function references
runtime-core strings such as `/hbui/gameplay.html`, `core:routing:not-found`,
`core:gui:resize-hack`, `core:telemetry:firstContentfulPaint` and `JsException` — not a
specific screen. It is not the same function as the 1.26 hook target.

## Note: route strings come and go every version

Every version has a different route-string set; some screens disappear and return in the next
release. A fixed signature table will always rot, so **re-derive with `derive_sig.py` for each
new version** instead of expecting a hand-written table to cover everything.
