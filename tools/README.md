<div align="center">

<h1 align="center">Signature derivation and verification</h1>

<p>For Minecraft Bedrock's <code>libminecraftpe.so</code> — locate the OreUI config function<br>
and generate a signature using <b>pure byte analysis</b>. No IDA, no decompiler.</p>

<p>
<b>English</b> · <a href="./README.zh-CN.md">中文</a>
</p>

</div>

<br/>

> [!NOTE]
> **Why not IDA?** `libminecraftpe.so` is 240–340 MB. A full IDA decompilation needs several GB
> of RAM and takes minutes to hours. Finding a signature only needs the ELF header, the anchor
> strings and a `.text` scan — tens of seconds, well within a standard GitHub Actions runner.

## 1. What a signature looks like

```
? ? ? D1 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? 91 ? ? ? B9 ? ? ? D5 F9 03 03 2A
```

- `?` is a wildcard byte; everything else is fixed.
- **AArch64 is little-endian, so the opcode lands in the highest byte.** The convention here is
  "blank the low 3 bytes, keep the highest byte": `D1` is `sub sp,sp,#imm`, `A9` is `stp`,
  `91` is `add imm`, `B9` is `str`, `D5` is `mrs`.
- The trailing 1–2 `mov` instructions are the **version imprint**
  (`F9 03 03 2A` = `mov w25, w3`). Each build shuffles a different argument into a different
  callee-saved register, and that is precisely what tells versions apart.

## 2. The algorithm (`derive_sig.py`)

### Step 1 — Locate the function by strings, not by a signature

```
/oreui-marketplace/search   /oreui-marketplace/wishlist
/start-from-template        /create-new-world
/player-permissions         /send-invites
/gameplay/bedtime           /pre-start
```

These OreUI route strings sit at stable offsets in `.rodata` and exist across versions (though
they do come and go — see section 4). Find the `ADRP+ADD` pairs in `.text` that reference them:

```
ADRP Xn, page          ; page address, 4 KB aligned
ADD  Xn, Xn, #off      ; add the low 12 bits → address = (PC & ~0xFFF) + page + off
```

Resolving that address tells you where the string lives, which in turn tells you where the code
that uses that route lives.

### Step 2 — Walk back to the function entry

Scan backwards from the reference for the nearest location that genuinely looks like a function
start. The criteria:

- The instruction is `sub sp,sp` or `stp`
- **The preceding instruction is `ret` / `br` / unconditional `b`** (a function-end marker)
- The first instruction is **not** `ADRP`

> The last rule matters. `ADRP` and `sub sp,sp` collide under the mask
> (`912EFC00` is `add x0,x0,#0xBBF`, misread as `sub sp,sp`). An early version omitted this
> check and picked the wrong function.

### Step 3 — Collect the prologue and wildcard it by class

Walking down from the function entry, collect `sub sp,sp` / `stp` / `add imm` / `str` / `mrs` /
`mov`, and stop at the first unrecognised class. The variable-bit mask per class:

| Class | Mask | Kept highest byte |
|---|---|---|
| `sub_sp` | `0x003FFFE0` | `D1` |
| `stp` | `0x003FFFE0` | `A9` |
| `add_imm` | `0x003FFFE0` | `91` |
| `str` | `0x003FFFE0` | `B9` / `F9` |
| `mrs` | `0x00FFFFFF` | `D5` |
| `mov_reg` | `0x00FFFFFF` | `2A` / `AA` |

When the `mov` source is `x0`–`x7` (an argument), the **whole instruction is kept fixed** —
that is the version imprint.

### Step 4 — Choose among candidate functions

Dozens of functions can reference the same route string (one registration function per screen).
Score by prologue shape:

```
+10  first instruction is sub sp,sp
+ 4  ×  number of stp            (more saved registers = larger function)
+ 3  contains add_imm
+ 3  contains mrs                (stack-protector canary)
+ 6  contains mov_reg            (argument shuffle = where the imprint lives)
```

**Do not rank by reference count.** Measured on 1.21.132.1, the most-referenced function was a
small neighbouring helper; the real target simply had a longer prologue.

## 3. Cross-validation

Deriving against 1.26.62.1 yields:

```
? ? ? D1 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? 91 ? ? ? B9 ? ? ? D5 F9 03 03 2A
```

That is **byte-for-byte identical** to the hand-maintained `1.26.60` entry in the mod's source —
direct evidence the algorithm is correct.

The same signature hits `0x9AF937C` on 1.26.62.1, and the hit is exactly a function entry
(preceded by `ret`). The function body references `/oreui-marketplace/search`, `/send-invites`
and `/main-menu`, confirming it is the OreUI config registration function.

`derive_sig.py` now re-scans its own output and reports whether every hit lands on a function
entry, so this check is no longer something you have to do by hand.

## 4. Limitations you must know

### 4.1 OreUI grew incrementally — it is not a switch

Since 1.20, screens have been migrating from JSON-UI to OreUI one at a time:

| Version | State |
|---|---|
| 1.20.10.01 | HBUI/OreUI **runtime core** already present, UI still mostly JSON-UI |
| 1.21.x | OreUI screens appear one by one |
| 1.26.x | Almost every screen is OreUI |

On 1.20.10.01 the signature hits `0x56DBB0C` (a real function entry), but that function
references:

```
/hbui/gameplay.html   core:routing:not-found   core:gui:resize-hack
core:telemetry:firstContentfulPaint   core:exception   JsException
core:keyboard:dismissed   ScreenLoadFailed
```

— runtime core, not a specific screen. In other words **1.20 and 1.26 do not share a hook
target**, so a 1.26 signature cannot be reused for 1.20.

### 4.2 Route strings come and go

Every version has a different route-string set; some screens vanish for a release and return in
the next. A fixed signature table will therefore always rot — **re-derive for each new version**
rather than expecting a hand-written table to keep up.

### 4.3 A hit is not the same as usable

| Version | Hit address | Previous insn | Function entry? |
|---|---|---|---|
| 1.20.10.01 | `0x56DBB0C` | `ret` | ✅ |
| 1.21.0.03 | `0x5F85100` | `ret` | ✅ |
| 1.21.90.3 | `0x5EC9928` | `ret` | ✅ |
| 1.21.132.1 | `0x907E4BC` | `bl` | ❌ real entry is `0x907E3D4` |
| 1.26.0.2 | `0x913DF54` | `bl` | ❌ |
| 1.26.62.1 | `0x9AF937C` | `ret` | ✅ |

Hooking the middle of a function overwrites a called instruction or corrupts the stack frame.
**Always verify that the preceding instruction is `ret`** before trusting a signature.

## 5. Usage

### Derive a signature for a new version

```bash
python3 tools/derive_sig.py libminecraftpe.so
python3 tools/derive_sig.py libminecraftpe.so --json out.json
```

Output includes the located function address, the prologue instruction list, and the generated
signature.

### Check which versions your existing signatures hit

```bash
python3 tools/scan.py libminecraftpe.so            # pass the .so directly
python3 tools/scan.py "Minecraft PE x.xx.apk"      # or an APK; the .so is extracted
python3 tools/scan.py --json report.json *.so
```

### Extract the .so from an APK

`scan.py` handles this automatically. To extract only:

```bash
python3 tools/scan.py --dump-so libminecraftpe.so game.apk
```

## 6. Using it from GitHub Actions

`libminecraftpe.so` is 240–340 MB, so do not bundle this with a full IDA decompilation job.
Use a separate lightweight job instead: upload the `.so` → run `derive_sig.py` → print the
signature. The whole thing takes tens of seconds and needs no IDA, no patching and no virtual
desktop.

See [`SIG_MATRIX.md`](./SIG_MATRIX.md) for the measured hit matrix.
