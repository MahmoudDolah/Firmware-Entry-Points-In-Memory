# Re-examining "Firmware Entry Points in Memory" (2026)

Ten years on, with modern tooling (Python 3, `numpy`, `binwalk` v3, `radare2`, and an
agentic RE CLI called `rea`) sitting a `pip`/`cargo install` away, this is a technical
re-test of the paper's central claims against the *exact* four files logged in `notes.txt`:
`openwrt-ar71xx-generic-nbg460n_550n_550nh-u-boot.bin`, `vmlinux-lzma.elf`,
`uImage-lzma.bin`, and `vmlinux.elf`, all under `openwrt/openwrt-mips-ar71xx/bin/ar71xx/`
(§1–§6), extended in §7 to a file the paper never tested: the x86 kernel image under
`openwrt/OpenWrt-ImageBuilder-15.05.1-x86-generic.Linux-x86_64/bin/x86/`. (The third
directory in `openwrt/`, `openwrt-generic-x86/`, turns out to be just an extracted
root filesystem — no kernel or bootloader binary in it — so it isn't part of this test.)

Headline: the paper's *data* holds up better than its *diagnosis*. The numbers in
`notes.txt` and the paper's tables are almost entirely reproducible byte-for-byte. But the
paper's explanation for why `basefind.py` failed ("may be relying on ARM-specific
structures") is a guess that turns out to be wrong — the real causes are precise,
measurable, and worse than that guess in one case, much more fixable in another.

## Method

`basefind.py` is Python 2 (`print` statements, `xrange`), so it can't run as-is on this
machine. I wrote a byte-for-byte-faithful Python 3 port of its two extraction functions
(`get_strings`, `get_pointers` — only mechanical syntax changes) and its scoring loop, then
empirically cross-validated a vectorized `numpy` reimplementation against the faithful port
on a 300-base sample per file (exact match, all four files) before trusting the vectorized
version for a full-range run. The equivalence isn't just empirical: `score(base)` is only
nonzero when some pointer value `p` and string offset `o` satisfy `p - base == o`, i.e.
`base == p - o` — so instead of testing every candidate base against every pointer
(`O(bases × pointers)`), you can loop over `(pointer, string-offset)` pairs and bucket the
result directly (`O(pointers × strings)`), with provably identical output. The original's
`del ptr_table[ptr]` pruning when `ptr < base` is a pure optimization with zero effect on
results, since a negative offset can never be in `str_table` anyway.

Ground truth, independently derived, not assumed:
- `file(1)` and `binwalk` for the uImage header (it's a documented 64-byte struct).
- `readelf -h` / `readelf -l` for the two ELF files.
- Direct MIPS disassembly (`radare2`, big-endian, raw binary mode) for `u-boot.bin`, which has no header at all.

## 1. Reproducibility audit: the raw data is solid; two typos crept into the typeset tables

All four of `basefind.py`'s reported top-20 tables reproduce **exactly** by rerunning the
real algorithm against the real files. Comparing `notes.txt` (the raw lab notes) against
`firmwareEntryPointsInMemory.tex` (the typeset tables) line-by-line turns up two single-digit
transcription drops, both in the *last* row of a table, both confirmed wrong by the rerun:

| File | `notes.txt` (correct) | `.tex` table (as published) |
|---|---|---|
| u-boot.bin, row 20 | `0x212dd000  44` | `0x212dd000 & 4` |
| vmlinux.elf, row 20 | `0x20dc0000  5533` | `0x20dc0000 & 553` |

Separately, `vmlinux-lzma.elf`'s published table only has 18 rows where it should have 20 —
not a transcription error (both `notes.txt` and the `.tex` agree on 18), just an incomplete
capture of the original run; the rerun fills in the missing two: `0x1a38000 → 1` and
`0x23c3000 → 1`.

None of this changes any conclusion, but it's worth knowing the source data is trustworthy —
the problems are in the method and the diagnosis, not the numbers.

## 2. The reported multi-hour runtimes are an implementation defect, not a property of the technique

This is the biggest actionable gap. The paper's Limitations section proposes "us[ing]
heuristics to determine attributes of the binary" to speed things up. That's solving the
wrong problem. The real issue is the loop nesting: iterating `bases × pointers` when you
could iterate `(pointers or strings, whichever is smaller) × the other`, which is
mathematically guaranteed to produce the same scores (proof above) while doing far less work:

| File | reported original runtime | original op-count (bases×ptrs) | reformulated op-count | op-count reduction | measured reformulated wall-clock | speedup vs. reported |
|---|---|---|---|---|---|---|
| u-boot.bin | 2.3 hrs | 1.63×10¹⁰ | 8.28×10⁶ | ~1,970× | 0.06 s | ~140,000× |
| vmlinux-lzma.elf | 27.5 hrs | 3.30×10¹¹ | 5.71×10⁶ | ~57,850× | 0.06 s | ~1,770,000× |
| uImage-lzma.bin | 13.4 hrs | 3.29×10¹¹ | 1.67×10⁶ | ~196,630× | 0.03 s | ~1,855,000× |
| vmlinux.elf | 11.3 hrs | 1.71×10¹¹ | 4.28×10⁹ | ~40× | 27.5 s | ~1,480× |

The "op-count reduction" column is hardware-independent — it's just how much less
arithmetic the reformulated algorithm has to do, provably, for identical output. The
"speedup vs. reported" column is bigger still because it also folds in a decade of
hardware/software improvement and `numpy`'s vectorization — those two effects can't be
fully separated from the paper's numbers alone, but the op-count math by itself already
proves the original was doing 40×–196,630× more work than necessary. Note `vmlinux.elf`
is the one file where the two counts (strings vs. pointers) are closest in magnitude, which
is exactly why it's the only one that still takes measurable time (27.5s, not milliseconds)
even after the fix.

## 3. The accuracy failures have a precise, measurable cause — a different one than the paper guessed

The paper's Discussion attributes the wrong-answer results to architecture ("basefind.py may
be relying on specific structures ... found within binaries for ARM that simply do not exist
in other formats"). That's untested speculation. Measured directly:

| File | qualifying strings found | distinct pointer values | score at the *true* base | rank among nonzero candidates |
|---|---|---|---|---|
| vmlinux-lzma.elf (base `0x80050000`) | **17** | 335,592 | **0** | never appears (1,356 nonzero bases total) |
| uImage-lzma.bin (base `0x80060000`) | **5** | 334,592 | **0** | never appears (335 nonzero bases total) |
| vmlinux.elf (base `0x8005f000`) | 24,580 | 174,147 | **1** | 256,073rd out of 387,828 |

Two distinct, now-provable causes, neither of which is "ARM vs. MIPS":

- **For the two LZMA-compressed files, the technique's core signal is structurally absent.**
  `get_strings()`'s whole premise is that a pointer table will correlate with runs of ≥10
  printable ASCII characters. Compressed data is close to maximum-entropy by construction —
  there are almost no such runs (17 and 5, in 1.4MB and 1.3MB files respectively). This has
  nothing to do with instruction set; it would fail identically on a compressed ARM image.
- **For the one uncompressed, string-rich file, the signal exists but is statistically
  worthless.** `vmlinux.elf` has a healthy 24,580 candidate strings, and the true base *does*
  register a nonzero score — but score = 1 is the floor (a single coincidental pointer/string
  alignment), and it ranks below roughly two-thirds of all 387,828 candidates that scored
  *something* purely by chance. The correct answer isn't merely outside the top 20; it's
  indistinguishable from noise under this file's own pointer/string statistics.

## 4. Two of the four files didn't need `basefind.py` at all — and the paper's own citation says so

```
$ file openwrt-ar71xx-generic-uImage-lzma.bin
... u-boot legacy uImage ... Load Address: 0X80060000, Entry Point: 0X80060000 ...
```
```
$ binwalk openwrt-ar71xx-generic-uImage-lzma.bin
0  0x0  uImage firmware image, ... load address: 0x80060000, entry point: 0x80060000 ...
```

Both tools return the answer in milliseconds, with no ambiguity, because uImage is a
documented 64-byte header format with the load address and entry point as literal fields —
there is nothing to search for. `binwalk` on `vmlinux-lzma.elf` goes further, for free:

```
71404  0x116EC  Copyright text: "Copyright (C) 2011 Gabor Juhos <juhosg@openwrt.org>"
71632  0x117D0  LZMA compressed data, ... uncompressed size: 4266300 bytes
```

Binwalk is the paper's own reference **[7]**, cited in Related Work as an existing tool
that "uses known patterns to locate structures within executables" — but it's never actually
run anywhere in the Implementation or Evaluation. Running it now shows the paper's own
Evaluation attempted to heuristically solve, for half its test files, a problem a tool it had
already cited would have solved outright.

(I also tried `rea`'s `extract-firmware` command — a modern agent-oriented RE CLI, thematically
on point — but it delegates to an external `unblob` binary via `REA_UNBLOB_COMMAND`, which
isn't configured in this environment, so it declined rather than running. Not pursued further
since `binwalk`/`file` already answered the same question directly.)

## 5. The one header-less file: direct disassembly beats both approaches

`u-boot.bin` has no uImage header and isn't an ELF (`file` just says "data"; `binwalk` finds
only a generic CRC32-table false-positive). This is the one case where the paper's own
`basefind.py` numbers looked the most promising ("more closely resembled Coppola's results")
and the one case the paper admits it couldn't verify ("not in ELF format ... not able to
easily verify the validity of any of the candidate addresses").

Disassembling the start of the file as big-endian MIPS (`radare2`, raw binary, base 0) shows
a standard MIPS exception-vector table at offset 0 (branches to `0x400`/`0x570`, the
TLB-refill/general-exception handler offsets) — consistent with offset 0 in the file being
the hardware reset vector. A few instructions into the handler:

```
0x468   lui  t0, 0x8000
0x46c   ori  t0, t0, 0x1000
0x470   addiu sp, t0, 0        ; sp = 0x80001000, hand-placed, absolute
```

Position-dependent MIPS code has no choice but to embed absolute 32-bit addresses as literal
`lui`/`ori` immediate pairs wherever it needs one — there's no PC-relative addressing for
large constants. That means the file's *real* link address is sitting in plain sight in the
instruction stream, deterministically, no statistical scan required. This one instruction
pair already places the code in the same `0x8000_0000`+ RAM window as both kernel images
(`0x80050000`/`0x80060000`), which is circumstantial but consistent, not a fully pinned-down
`CONFIG_SYS_TEXT_BASE` citation — I looked for the board's historical OpenWrt/u-boot build
config to nail the exact value and didn't find it in the time available, so I'm flagging this
as the one open item rather than overclaiming it. The point stands regardless: the paper
never considers "read the absolute constants the compiler already baked into the code" as an
alternative to blind statistical scanning, and for exactly the one file-type (header-less,
position-dependent machine code) where that alternative is most directly applicable.

## 6. A conceptual gap: "base address" and "entry point" are conflated

The paper is titled "Firmware Entry Points in Memory," but both `basefind.py` and the
Evaluation only ever compute a **base address** — the address file-offset-0 would occupy in
memory — never the actual entry point (where execution starts). These are not the same
number: `readelf` reports `vmlinux.elf`'s entry point as `0x80064f00`, 0x4f00 bytes past its
base of `0x80060000`. For a raw boot image like `u-boot.bin` the two likely coincide (CPU
reset jumps straight to file-offset 0), but the paper never states or verifies this
assumption. Also worth naming explicitly, since the paper never does: its "readelf base
address" figures (`0x80050000`, `0x8005f000`) use the convention `p_vaddr − p_offset` of the
first `LOAD` segment — i.e., "what address would file-offset-0 map to" — which is exactly the
right ground truth for comparing against `basefind.py`'s own definition of base (since its
`str_table` holds *file* offsets), but a reader can't tell that from the paper, and a naive
read of `readelf -h`'s "Entry point address" field gives a different, equally plausible-looking
number (`0x80064f00`) that isn't what's being reported.

## What this changes about the paper's Conclusion

The Conclusion's core claim — that `basefind.py` doesn't generalize past Coppola's original
ARM binary — needs revising, not just re-evidencing. The paper frames the failure as
architectural ("poor solution as a general tool," attributed to ARM-specific structures
"that simply do not exist" elsewhere). §7's x86 result directly contradicts that: the same
technique, unmodified, ranks the correct answer **#1** out of 633,000 candidates on an
uncompressed x86 kernel — a third architecture, neither ARM nor MIPS. The real fault line
running through all three architectures tested (ARM, MIPS, x86) is **compression, not
instruction set**: every compressed payload tested (MIPS `vmlinux-lzma.elf`, MIPS
`uImage-lzma.bin`, x86 `vmlinuz`) starves the string-correlation signal to near zero;
every uncompressed, string-rich file tested either works outright (x86) or at least
registers a nonzero signal (MIPS `vmlinux.elf`, albeit a weak one).

That reframes what's actually recoverable: two of the original four formats have a trivial,
deterministic answer already available (uImage headers, ELF program headers); a third has a
strong deterministic lead via direct disassembly; the x86 case shows the heuristic itself
can be excellent, not just a last resort, when given uncompressed input. Only genuinely
compressed payloads are hard, and they're hard for a specific, provable reason (near-zero
string signal) rather than an architectural one. A revised Future Work could say,
concretely: detect container format first (uImage/ELF/compressed) and only fall back to
statistical scanning — with the loop reformulated, and using the pointer addressing
convention the compiled code actually uses (§7's physical-vs-virtual comparison shows this
matters a great deal) — when no header is present and the payload isn't compressed.

## 7. A third architecture, not in the original paper: x86 overturns the "it's an architecture thing" theory

The paper only ever tested ARM (Coppola's original) and MIPS (ar71xx). The repo's other
OpenWrt build — `OpenWrt-ImageBuilder-15.05.1-x86-generic.Linux-x86_64/bin/x86/openwrt-15.05.1-x86-generic-vmlinuz`,
a real x86 Linux kernel image — gives a third, cleanly independent data point, and it
changes the conclusion.

**Ground truth, cross-confirmed from two independent sources.** `vmlinuz` is a standard
bzImage: a small real-mode/protected-mode "setup" stub followed by a compressed kernel body
(`binwalk` finds an XZ stream at offset `0x3C62`). Parsing the documented boot header
(`Documentation/x86/boot.rst` field offsets, confirmed via the `0xAA55` boot signature and
`"HdrS"` magic) gives `pref_address = 0x01000000` — the one true load address, since
`relocatable_kernel = 0`. Carving out and decompressing the XZ stream yields a full ELF
(`ELF 32-bit LSB, Intel 80386, stripped`), and `readelf` independently reports
`PhysAddr = 0x01000000`, `VirtAddr = 0xc1000000` (the standard 3G/1G split,
`0xC0000000 + physical`) for the first `LOAD` segment, with entry point `0x01000000` — an
exact match to the header field, from a completely different parsing path. (The header's
*other* address field, `code32_start = 0x00100000`, is a third, legitimately different
number — where the setup stub itself runs, 1MB, before it decompresses and jumps to the
real kernel at 16MB. Three different, all-correct "address" fields in one file is a sharper
illustration of the base-vs-entry-point conflation in §6 than the MIPS case gave.)

**basefind.py was re-run the same way** (faithful port, cross-validated against the fast
reformulation on a 300-base sample — exact match for both files, same as the MIPS cases):

| File | strings | distinct pointers | score at true base | rank | nonzero candidates |
|---|---|---|---|---|---|
| `vmlinuz` (full bzImage, mostly compressed) | 58 | 612,240 | 0 (at `code32_start=0x00100000`) | never appears | 8,209 |
| decompressed kernel ELF, **virtual-address convention** (`0xc1000000 − 0x1000 = 0xc0fff000`) | 19,595 | 584,093 | **11,399** | **#1** | 632,989 |
| decompressed kernel ELF, physical-address convention (`0x01000000 − 0x1000 = 0x00fff000`) | 19,595 | 584,093 | 18 | 55,463rd | 632,989 |

Two things this settles:

- **The compressed `vmlinuz` result is the same story as the MIPS compressed files** (near-zero
  strings → zero signal at the true address), now confirmed on a third architecture. This is
  the strongest evidence yet that compression, not CPU architecture, is the actual
  determining factor.
- **The uncompressed x86 kernel is the opposite of the MIPS result — and directly falsifies
  the paper's hypothesis.** The paper's Discussion guessed `basefind.py` "may be relying on
  specific structures ... found within binaries for ARM that simply do not exist" in other
  architectures. Here, on x86 — neither ARM nor MIPS — the technique works about as well as
  it possibly could: the true base isn't just in the top 20, it's **rank #1** by a wide
  margin (11,399 vs. the next-best candidate's 5,074), out of nearly 633,000 candidates that
  scored something. Architecture isn't the variable that matters; whether the input is
  compressed is. The MIPS kernel's poor showing (§3) is a property of that specific
  uncompressed-but-still-weak-signal binary, not of "MIPS" as a category — x86 shows an
  uncompressed kernel absolutely can produce a dominant, unambiguous top-1 result.
- **Addressing convention matters a lot, concretely.** Scoring the *same* correct kernel
  against the wrong (but equally plausible-looking) address convention — physical instead of
  virtual — drops the result from rank #1 with a dominant score to rank 55,463rd with a score
  of 18. The compiled kernel's embedded pointers are virtual addresses (as expected for code
  written to run post-paging), so that's the convention that has to be used; get it wrong and
  a technique that actually works looks like it's failing.

Algorithmic note, same reformulation as §2, same proof of equivalence: `vmlinuz` sped up
~16,949× in operation count (0.29s vs. an untested but clearly multi-hour naive-loop
projection); the decompressed ELF's reformulated run took 80.8s (only a ~50× operation-count
reduction, since its string and pointer counts are closer in magnitude — the same pattern
seen with `vmlinux.elf` in §2).

## Tooling used

- Python 3 (faithful port of `basefind.py`'s extraction/scoring logic) + `numpy` (vectorized,
  cross-validated reformulation) — no new system packages.
- `binwalk` 3.1.0, via `cargo install binwalk` (the PyPI `binwalk` 2.1.0 package is a dead
  stub that `ModuleNotFoundError`s on import; the real tool is now a Rust rewrite distributed
  via `crates.io`, not `pip`).
- `radare2` 6.2.4, extracted from the official `.deb` with `dpkg-deb -x` (no root available in
  this environment; this avoids needing `sudo apt install`).
- `readelf`/`objdump`/`file`/`xz` — already present. `xz -d` carved the compressed kernel
  body out of the x86 `vmlinuz` (binwalk located the stream offset; the stream itself is
  self-delimiting) for the §7 comparison.
- `rea` (agentic RE CLI, installed earlier this session) — `extract-firmware` is wired for
  this exact use case but needs an external `unblob` binary not present here; noted, not
  pursued.

All exploratory scripts live outside the repo (session scratch directory) — nothing here
required editing `basefind.py` itself.
