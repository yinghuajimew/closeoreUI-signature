<div align="center">

<h1 align="center">closeoreUI-signature</h1>

<p>Derive and verify <b>byte signatures</b> for Minecraft Bedrock's OreUI config function<br>
from a <code>libminecraftpe.so</code> — pure static analysis, no IDA, no decompiler, no root.<br>
One <code>.so</code> takes tens of seconds.</p>

<p>
<b>English</b> · <a href="./README.zh-CN.md">中文</a>
</p>

</div>

<br/>

> [!NOTE]
> This repository contains **tools and a workflow only**. It ships no Minecraft binary
> files and no derivatives of them. You supply the `.so` yourself.

## The problem

Mods like `ForceCloseOreUI` need a byte signature to locate the OreUI config function:

```
? ? ? D1 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? 91 ? ? ? B9 ? ? ? D5 F9 03 03 2A
```

Every game update reshuffles the register allocation in the function prologue, and the old
signature stops matching. This tool **derives** the new one from the updated
`libminecraftpe.so` automatically, instead of waiting for someone to guess it.

## Quick start

### 1. Run it on GitHub Actions (recommended — no local setup)

1. Fork this repository.
2. Upload your `libminecraftpe.so` to **your fork's Releases** (create a tag, e.g. `1.26.62.1`).
   - The `.so` is 240–340 MB, over Git's 100 MB per-file limit, so it **must** go in a Release asset.
3. Go to Actions → `derive-signature` → Run workflow, and fill in:
   - `target`: `libminecraftpe.so` (the asset name)
   - `from_release`: `1.26.62.1` (your tag; `latest` picks the newest)
   - `derive`: ✅
4. Tens of seconds later the new signature is printed at the end of the log, and the report is
   downloadable as an artifact.

### 2. Run it locally

Requires **Python 3.10+** (uses `X | None` annotations). **No third-party packages** — the
tools import only `json`, `struct`, `sys`, `zipfile` and friends from the standard library.

```bash
python3 tools/derive_sig.py libminecraftpe.so
python3 tools/derive_sig.py libminecraftpe.so --json out.json
```

### 3. Check which versions your existing signatures still hit

```bash
python3 tools/scan.py libminecraftpe.so          # pass the .so directly
python3 tools/scan.py "Minecraft PE x.xx.apk"    # or pass an APK; the .so is extracted
python3 tools/scan.py --json report.json *.so
```

## How it works

### Step 1 — Find the function by its strings, not by a signature

Search `.rodata` for OreUI route strings:

```
/oreui-marketplace/search   /oreui-marketplace/wishlist
/start-from-template        /create-new-world
/player-permissions         /send-invites
/gameplay/bedtime           /pre-start
```

Then find the `ADRP+ADD` pairs in `.text` that reference them, and resolve the address:

```
ADRP Xn, page        ; page address, 4 KB aligned
ADD  Xn, Xn, #off    ; add the low 12 bits → (PC & ~0xFFF) + page + off
```

### Step 2 — Walk back to the function entry

Scan backwards from the reference for the nearest thing that actually looks like a function
start. The criteria:

- The instruction is `sub sp,sp` or `stp`
- **The preceding instruction is `ret` / `br` / unconditional `b`**
- The first instruction is **not** `ADRP`

> The last rule matters. `ADRP` and `sub sp,sp` collide under the mask
> (`912EFC00` is `add x0,x0,#0xBBF` but gets misread as `sub sp,sp`). An early version
> omitted this check and picked the wrong function.

### Step 3 — Collect the prologue and wildcard it by instruction class

AArch64 is little-endian, so **the opcode lands in the highest byte**. The convention is
therefore "blank the low 3 bytes, keep the highest byte":

| Class | Mask | Kept byte |
|---|---|---|
| `sub sp,sp,#imm` | `0x003FFFE0` | `D1` |
| `stp` / `ldp` | `0x003FFFE0` | `A9` |
| `add xN,xM,#imm` | `0x003FFFE0` | `91` |
| `str` / `ldr` | `0x003FFFE0` | `B9` / `F9` |
| `mrs xN,<sysreg>` | `0x00FFFFFF` | `D5` |
| `mov xN,xM` | `0x00FFFFFF` | `2A` / `AA` |

**Version signature:** when the `mov` source is `x0`–`x7` (an argument), the whole
instruction is kept **fixed**. Different builds move different arguments into different
callee-saved registers — that is exactly what distinguishes one version from another.

### Step 4 — Score the candidate functions

Dozens of functions reference the same route strings (one registration function per screen).
Score them by prologue shape:

```
+10  first instruction is sub sp,sp
+ 4  ×  number of stp            (more saved registers = larger function)
+ 3  contains add_imm
+ 3  contains mrs                (stack-protector canary)
+ 6  contains mov_reg            (argument shuffle = where the version imprint lives)
```

**Do not rank by reference count.** Measured on 1.21.132.1, the most-referenced function was
a small neighbouring helper, while the real target had a longer prologue.

## Verification

Deriving against 1.26.62.1 produces:

```
? ? ? D1 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? 91 ? ? ? B9 ? ? ? D5 F9 03 03 2A
```

- **Byte-for-byte identical** to the hand-maintained `1.26.60` entry in the mod's source.
- Hits `0x9AF937C`, and that address is exactly a function entry (preceded by `ret`).
- The function body references `/oreui-marketplace/search`, `/send-invites`, `/main-menu`.
- Running on GitHub Actions produced **byte-identical** output to running locally.

`derive_sig.py` now self-verifies: it re-scans its own output and reports whether every hit
lands on a function entry.

## Important limitations

### 1. OreUI grew incrementally — it is not a switch

| Version | State |
|---|---|
| 1.20.10.01 | HBUI/OreUI **runtime core** already present, UI still mostly JSON-UI |
| 1.21.x | OreUI screens appear one by one |
| 1.26.x | Almost every screen is OreUI |

On 1.20.10.01 the signature hits `0x56DBB0C` (a real function entry), but that function
references `/hbui/gameplay.html`, `core:routing:not-found`, `core:gui:resize-hack`,
`core:telemetry:firstContentfulPaint` and `JsException` — **runtime core, not a screen**.
It is a different function from the 1.26 hook target and cannot be reused as-is.

### 2. Route strings change every version

Screens disappear in one release and come back in the next. A fixed signature table will
always rot, so **re-derive for each new version** rather than expecting the table to keep up.

### 3. A hit is not the same as usable

| Version | Hit address | Previous insn | Function entry? |
|---|---|---|---|
| 1.20.10.01 | `0x56DBB0C` | `ret` | ✅ |
| 1.21.0.03 | `0x5F85100` | `ret` | ✅ |
| 1.21.90.3 | `0x5EC9928` | `ret` | ✅ |
| 1.21.132.1 | `0x907E4BC` | `bl` | ❌ real entry is `0x907E3D4` |
| 1.26.0.2 | `0x913DF54` | `bl` | ❌ |
| 1.26.62.1 | `0x9AF937C` | `ret` | ✅ |

Hooking the middle of a function overwrites a called instruction or corrupts the stack frame.
**Always re-verify with `scan.py` that the hit lands on a function entry.**

See [`tools/SIG_MATRIX.md`](./tools/SIG_MATRIX.md) for the full 6-sample × 10-signature matrix.

## Repository layout

```
tools/derive_sig.py        derive a new signature
tools/scan.py              check which versions existing signatures hit (accepts APK too)
tools/README.md            algorithm details (English)
tools/README.zh-CN.md      algorithm details (Chinese)
tools/SIG_MATRIX.md        measured hit matrix across samples (English)
tools/SIG_MATRIX.zh-CN.md  measured hit matrix across samples (Chinese)
.github/workflows/derive-signature.yml   manually triggered lightweight workflow
```

## Licence

**GNU Affero General Public License v3.0** (AGPL-3.0).

The reason for AGPL specifically: its section 13 requires that **providers of a network
service also release their modifications**. If someone wires this derivation logic into an
online service — say, a site where you upload a `.so` and get a signature back — without
publishing their changes, AGPL obliges them to ship the source too.

See [LICENSE](./LICENSE) for the full terms.

## Disclaimer

This tool performs ELF static analysis only. It does not modify or redistribute any game
file. You must supply your own `.so`, and you are responsible for how you obtained it.
