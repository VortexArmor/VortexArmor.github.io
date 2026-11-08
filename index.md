---
title: "Anatomy of an .ex5 RAM dump: why wrapping your MT5 bot protects nothing"
date: 2026-11-08
---

## 1. The $10 copy of a $1,500 bot

I sell a commercial MT5 trading bot (MT5 = MetaTrader 5, the retail
forex/CFD terminal — if you've never touched it, think "a niche
ecosystem of paid expert-assistant programs that run inside a trading
platform", with its own file format, .ex5, and its own marketplace
culture). One afternoon a buyer sent me a screenshot: my bot — the one
I price at $1,500 — listed on a Telegram
channel for $10, inside a "group buy" bundle of forty-plus bots.
Crypto payments, instant delivery, a support channel of its own. The
pirate operation looked more professional than my sales page.

The uncomfortable part was not the piracy itself. I had made peace
with the idea that piracy exists. The uncomfortable part was *how*
it worked. My bot had not been decompiled. It had not been cracked in
any sense I understood the word at the time. Someone had bought one
legitimate license, run the bot once, and read it out of the memory of
a live process. Total effort, as far as I could reconstruct later:
under two minutes.

This post is the full write-up of that rabbit hole: how the attack
actually works, why every defense I had in place — and every defense
most bot vendors have in place — is irrelevant to it, what I ended up
rebuilding, and the measurements I took along the way. It's written
from the perspective of a bot vendor who got burned, not a security
company selling you something. That perspective turned out to matter:
almost all the standard advice comes from people who have never had
to live with the consequences of it failing.

## 2. Group-buy economics: why bot piracy is an industry, not a hobby

Before the technical part, it's worth understanding the economics,
because the economics dictate what a defense must accomplish.

Trading bots are a peculiar product:

- High perceived value. A bot that trades gold on M5 and claims
  double-digit monthly returns sells for $500–$3,000.
- Zero marginal cost of copying. One working binary serves unlimited
  buyers.
- A buyer population that is, statistically, price-sensitive and
  trust-limited. Many would prefer a $10 "group buy" copy from a
  Telegram channel over a $1,500 purchase from an unknown vendor.

Group-buy channels aggregate exactly that demand: forty people chip in
$10 each, one of them buys (or cracks) a single legit copy, everyone
gets the bot. The channel operator takes a cut of every bundle. This
is not one bored teenager with IDA Pro; it's a distribution business
with customers, support and repeat purchase behavior.

The consequence for defenders is brutal and clarifying: **you do not
need to stop a motivated reverse engineer.** If your security posture
assumes a nation-state adversary, you will spend unlimited effort and
still lose. What you need is to make the cost of producing one working
resale copy exceed the revenue a $10 copy generates. Break the unit
economics of the group-buy bundle and the industry moves on to the
next bot whose economics are still intact.

Keep that framing in mind for everything below. "Unhackable" is not a
goal, not a claim, and not what I built.

## 3. Threat model, stated plainly

Who attacks a commercial MT5 bot, and what can they actually do?

- **The bundle operator.** Funds one legit license, wants a copyable
  artifact out of it. Skills: competent scripter, not a
  vulnerability researcher. Uses public tools. This is 95% of the
  threat.
- **The eager customer.** Buys a $10 copy, wants to "fix" it when the
  license dies. Skills: low. Copies folders between terminals, edits
  configs.
- **The curious expert.** A skilled developer who buys one license
  out of professional curiosity. Rare, but the most technically
  capable. Usually not malicious — but if protection is trivially
  bypassed, they will talk about it publicly, and that talk is what
  the bundle operators read.

What they can all do: run the bot with a valid license on a machine
they fully control. That last clause is the whole problem statement.
**Any protection scheme whose guarantees depend on the client machine
being friendly is protecting nothing**, because the pirate's machine
is maximally unfriendly by construction.

What we are not defending against, up front:

- A kernel debugger attached from process start by a full-time
  professional. Expensive, rare, and — crucially — it doesn't scale
  to a $10 bundle business.
- A camera pointed at the monitor. If the value is in what the bot
  *displays*, no process-level protection helps; that's a different
  product problem.
- The legitimate buyer simply sharing account credentials where the
  licensing model allows it. That's a licensing-policy problem, not a
  memory-protection one.

## 4. The standard defenses, and the specific way each one fails

I had most of this list in place when my bot got resold. Here is the
honest post-mortem of each.

### 4.1 Wrappers and packers

The classic advice for .ex5 distribution: wrap the file, encrypt the
payload, unpack at load time. The failure mode is structural. A
wrapper is a self-decrypting archive. Whatever key unlocks the payload
has to be available to the unpacker on the client machine — so the
key is on the machine, and the plaintext is in memory one instruction
after unpacking. The wrapper protects the file on disk, and the file
on disk was never the thing being stolen.

There's a subtler point that took me a while to internalize: a packed
binary makes the *static* analysis harder and the *runtime* capture
identical. You have relocated the secret from "find it in the file"
to "find it in memory", and memory is exactly where the attacker is
looking. You've done the pirate a favor by consolidating everything
into one decrypt-at-once event.

### 4.2 Obfuscation

String encryption, control-flow flattening, opaque predicates,
symbol stripping. I shipped a build with all of it. The post-mortem:

- Obfuscation transforms **code**. The pirate's attack targets
  **data**. A virtualized function still loads its license key from a
  contiguous 2048-byte blob; a flattened loop still holds the neural
  weights as a findable float array.
- Obfuscation raises the cost of *understanding* the program. The
  group-buy pipeline doesn't require understanding. It requires
  *copying*. Nobody in the bundle chain ever reads your control flow.
- Symbol stripping is real but minor — it raises the cost of the
  "curious expert" path, which was never the main threat.

### 4.3 Hardware-ID (HWID) locks and online license checks

Both are good ideas that fail alone. A HWID lock is a string
comparison inside the process; patch it out, or read a valid HWID
response from a proxy. An online check phones home once and then
caches its result in — you guessed it — process memory, next to
everything else worth stealing. The check itself becomes another
artifact in the dump.

### 4.4 Integrity checks and anti-debug

Self-hash checks fail to the same runtime argument: whatever they
verify, the verified result sits in memory next to everything else.
Anti-debug is an arms race the defender loses by default at user
level — public tooling iterates faster than vendor checks — and it
has an ugly side effect: aggressive anti-debug behavior is exactly
the behavioral profile that makes antivirus engines flag your binary,
which produces a failure mode worse than piracy (below).

### 4.5 The failure mode worse than piracy

One thing I got wrong for months: my early hardening attempts made
the binary *look like malware* to heuristic antivirus (AV) engines.
When your
customers' terminals start quarantine-popping your product, you get
chargebacks instead of piracy. Any defense in this space must end a
scans clean across all major engines — not as a nice-to-have, as a
release gate. I'll return to the numbers in section 8.

## 5. The attack that beats all of them: runtime memory

### 5.1 Where the dump comes from

A running process leaks its memory from several directions, all
public, all unremarkable:

- Full process dumps via standard operating-system facilities — one
  API call away in any tool that has existed for twenty years.
- Crash-style minidumps, which include heap regions and are trivially
  requested.
- Plugin/injected readers that walk the address space from inside a
  helper process with sufficient access rights.
- On locked-down setups, even pagefile artifacts from a session where
  the bot ran.

None of these require exotic privilege, and — the part that matters —
**none of them are detectable-and-blockable from inside the target
process in any reliable way.** The read happens outside your code's
control flow. This is the asymmetry that kills client-side trust
models: you cannot prevent the read, so the only remaining move is to
make what gets read worthless.

### 5.2 What's actually in the dump

A dump of a running, licensed trading bot is embarrassingly rich:

- The license key material, in plaintext, in the heap.
- Serialized strategy parameters, often including things the vendor
  considers proprietary.
- Neural weights or lookup tables, if the bot uses them — large float
  arrays with very distinctive value distributions.
- Interface strings, error messages, sometimes the vendor's own
  comments.
- The decrypted image of any packed layer, sitting there post-unpack.

A pirate does not need to "understand" any of this. Pattern-matching
is enough, and the patterns are strong.

### 5.3 Finding secrets in a dump

This is where the attack goes from "possible" to "minutes". The
standard repertoire:

- **Known-plaintext search.** If you know any fragment of the secret —
  a key header, a magic constant, a fixed field — search for it
  directly. Works on every naive scheme.
- **Entropy-guided search.** High-entropy regions in an otherwise
  ordinary heap stand out statistically. Encrypted blobs, keys and
  compressed data light up like beacons. Ironic: the better your
  encryption, the more visible your key.
- **Structure search.** Float arrays with gaussian-ish distributions
  (weights), pointer-array patterns (vtables, allocation tables),
  length-prefixed strings. Each is a signature, and signatures are
  scriptable.
- **Diffing.** Run the bot twice with different license inputs, diff
  the dumps, and the deltas localize the interesting regions. This
  defeats "hide the key among megabytes of noise" schemes almost
  embarrassingly.

The result of this stage is not understanding — it's a *shopping
list* of memory regions, each with a known meaning.

### 5.4 Reassembly

With the shopping list, the pirate has options in ascending effort:
repackage the extracted weights/logic into their own harness; patch a
copy of the binary to skip licensing entirely (now that the dump
shows exactly which checks exist); or replay/clone whatever the
licensing accepts. The first option is the bundle operator's favorite
because it needs zero understanding of the vendor's code — and because
the extracted assets are exactly the parts customers pay for.

### 5.5 Cost accounting

Run the numbers on the full pipeline: one legit license (~$1,500, or
less on promotion), minutes of tool time, and a distribution channel
that already exists. Against that: forty sales at $10 each, minus
channel fees. When I did this arithmetic about *my own* bot, the
conclusion was unavoidable — at the prevailing level of protection,
pirating my product was a *rational business*. No amount of
obfuscation changes that; only changing what a dump yields does.

## 6. Rebuilding from first principles

Given the asymmetry in 5.1 — you cannot prevent the read, only change
what a read yields — the design constraints write themselves:

1. No secret may exist as a contiguous plaintext region in memory at
   any moment.
2. No credential may be valid outside the session that acquired it.
3. Compromise must be *containable*: a leaked session must be
   endable, fast, and leave nothing behind.
4. The hardened binary must look boring — no AV heuristics, no
   packed-bloat profile.

### 6.1 No contiguous plaintext: the masked arena

The core mechanism. All sensitive material lives in a dedicated
memory region — the arena — under a regime where no single address
ever exposes usable plaintext:

- **Fragmentation.** Each secret is split into fragments stored at
  unrelated offsets. There is no layout table in the clear that says
  which fragment belongs to what.
- **Masking.** Every fragment is stored masked — combined with a
  rolling mask value that itself never sits still. A dump is a
  photograph; the mask makes every photograph different, so diffing
  (5.3) produces noise instead of signal.
- **Register-only reassembly.** Fragments are combined into usable
  plaintext only in registers, only at the point of use, and only
  the piece needed for that use. The full plaintext of any secret
  exists in exactly one place for exactly the duration of one
  operation. Vectorized instructions (the AVX2 family) make the
  mask-and-recombine cheap enough that this is viable even on hot
  paths — with the discipline that genuinely latency-critical loops
  stay out of the arena entirely and carry no secrets.

A diagram of the difference, because it's the whole post in one
picture:

```
Naive:                          Masked arena:

heap @ 0x7F3A00                 arena fragment layout (differs per run)
  [ 2048-byte key, plaintext ]    0x...40: 9A F2 11 ..  (masked frag)
  [ weights array, plaintext ]    0x...71: 04 5C 8D ..  (unrelated frag)
  [ strings, plaintext ]          0x...B3: 77 10 E2 ..  (mask state)
                                  ...no contiguous plaintext anywhere
```

In the left picture, one dump is the whole theft. In the right
picture, a dump yields fragments whose meaning and assembly change
between runs. The attacker's reassembly work goes from "read the
shopping list" to "reverse-engineer the mask protocol" — and the mask
protocol is code, which changes with every build, which is what
finally puts the *curious expert* back in the cost equation too.

### 6.2 Licenses as one-time tokens

Static keys are standing targets: copy once, valid forever. The
licensing model moved to **one-time tokens**, each an Ed25519-signed
grant valid for exactly one session:

- Handshake verifies the token signature and binds it to the session
  — measured at ~0.01s, i.e. free at bot startup scale.
- A commit roundtrip confirms both sides observed the same handshake
  (the roundtrip is byte-exact by construction; a man-in-the-middle
  or a replay diverges at the first mismatched byte).
- A captured token is already spent. Copying a *token* from memory
  gains the pirate nothing, because the thing worth copying was never
  a reusable secret.

This also cleans up the licensing business logic that used to live in
the dump: there is no long-lived key to find, so the dump no longer
contains one.

### 6.3 Revocation that fails closed

The piece I initially resisted building, and now consider the
centerpiece. When a license is revoked, an operator-signed kill
verdict (0xFF) reaches the session and the arena **burns**: masks are
discarded, fragments are zeroized, the region is overwritten — in
about one second, end to end.

The design test for revocation is not "can I turn a license off". It
is: **after the off-switch, what does a dump of the process return?**
If the answer is "the secrets, still there in some form", the
revocation is decorative. After a burn, the answer is zeros — the
resale copy that a bundle operator was assembling dies mid-assembly,
and the $10 listing becomes a refund queue.

Fail-closed matters doubly because revocation is the one lever you
pull *after* abuse is detected. Detection will always be reactive —
someone reports the Telegram listing, you check the token ledger,
you kill the session. The entire defense end-to-end is: tokens
leak at most one session each, and each leaked session burns inside
a second of being noticed.

### 6.4 Look boring

The hardened build must present the same profile as an ordinary
native binary: no packer signatures, no section anomalies, no
behavioral tells. Hence the release gate — every build is scanned,
and 0 detections across all 60 engines is a ship/no-ship criterion,
not a marketing stat. (The full numbers are in the next section.)

## 7. Integration reality: MQL5, C#, Python

A design that can't be adopted doesn't protect anyone, so the
boundary matters as much as the mechanism. The engine exposes
bindings for MQL5, C# and Python, with an MQL5 Integration Kit that
covers the .ex5 workflow end to end. The practical shape: the bot's
secret-bearing parts are delegated to the protected engine; the MQL5
side orchestrates and trades; the arena, tokens and kill switch live
below the .ex5 boundary where the dump attacker lives.

The vendor-facing consequence is the interesting part: the *visible*
binary shrinks to a stub. What a reverse engineer sees of a protected
build — symbols, strings, size — collapses, because the substance
isn't in the file anymore at all.

## 8. Measurements, and how they were taken

Numbers from the final build. Methodology note: "before" is the
shipping v1 binary with wrapper + obfuscation (the standard stack);
"after" is the protected build. "What a reverse engineer gets" means:
material recoverable from a full memory dump plus static analysis of
the shipped file.

| What a reverse engineer gets | Before | After |
|---|---|---|
| Visible binary size | 16.0 MB | ~1.3 MB |
| Symbols | 49,237 | 0 |
| Readable strings | many | 0 |
| Contiguous key/weights in dump | present | absent |
| Token replay value | permanent | one session |
| Revocation burn time | — | ~1 s |
| License handshake | — | ~0.01 s |
| VirusTotal detections | — | 0/60 |

Two readings of this table:

- The "before" column is not a strawman; it is the standard stack,
  and I shipped every line of it believing it was protection.
- The single row that changes the business is "contiguous
  key/weights": every other number is downstream of that one.

## 9. What this does not protect against

Stated plainly, because the credibility of every claim above depends
on the honesty of this section:

- **Kernel-resident attackers from boot.** A professional with a
  kernel debugger attached before the process exists operates outside
  the model. Mitigations exist but the cost/benefit against a $10
  bundle adversary has never penciled out for me.
- **The display problem.** If your alpha is visible on a chart or in
  a log, it's exfiltrated by a phone camera. Process protection is
  orthogonal to information presented to humans.
- **Collusive legits.** A buyer who screen-shares their terminal with
  a community is a distribution channel no binary can close. Token
  revocation contains it (burn on report); it cannot prevent it.
- **The trusted-machine fantasy.** Any claim of protection that
  assumes the client machine cooperates is, as established in section
  3, protecting nothing. Everything above is designed inside that
  constraint, which is why some of it looks paranoid. It is exactly
  as paranoid as the adversary.

## 10. Open questions

Things I have not settled, offered to anyone working on the same
problem:

- The right cadence for mask-state rotation: too slow and diffing
  across a long session recovers structure; too fast and the
  recombine overhead shows up on the strategy-testing path.
- Whether dump-diffing *by the vendor* (i.e., shipping honeypot
  regions that change per build) is worth the false-positive cost in
  telemetry.
- Where the honest line is between "masked beyond casual recovery"
  and "masked beyond a funded professional" — and whether the second
  category is even meaningfully reachable at user-level privilege.

## 11. Footnote

I ended up building the above into a small protection service for
other bot authors — that's context, not the point of this write-up,
and this is the only sentence in which it appears. If you work on
memory protection or licensing design and can poke a hole in the
threat model, I want to hear it. Section 10 is an honest list of
where I think the remaining holes are.
