+++
date = '2026-04-27T00:36:00-07:00'
title = "Breaking a Lightweight Cipher's Security Proof: A 3-Round Trail That Shouldn't Exist"
summary = "On a critical flaw in the security analysis of NDN, an IoT-targeted block cipher published in IJCNA 2025"
+++

*On a critical flaw in the security analysis of NDN, an IoT-targeted block cipher published in IJCNA 2025*

---

## The world of lightweight crypto

Your smart thermostat needs to encrypt data. So does your insulin pump, your car's tire pressure sensor, the LED bulb in your living room, and the badge reader on the office door. Billions of these tiny devices ship every year, and each one needs to talk securely to something — a phone, a hub, a cloud service.

The problem is that AES — the standard encryption algorithm your laptop and phone use — is too heavy for the cheapest of these chips. AES needs a few thousand "gate equivalents" of silicon, kilobytes of memory, and energy budget that an IoT device on a coin-cell battery just doesn't have.

So researchers design *lightweight block ciphers*: simpler, smaller encryption algorithms that try to provide reasonable security while fitting on a microcontroller the size of a sesame seed. PRESENT, SLIM, LBlock, Piccolo, RECTANGLE, HIGHT — there are dozens of them, each with different design choices and trade-offs. New ones get proposed every year.

Earlier this year, a paper in the *International Journal of Computer Networks and Applications* introduced **NDN** ("Neural-Network Driven"), pitching it as the next-generation answer for IoT cryptography. It's a 64-bit block cipher with 80- and 128-bit keys, 12 or 18 rounds, and a structure that combines two classical designs (Feistel networks and substitution-permutation networks). The paper claims robust resistance to all known cryptanalytic techniques.

I've been reading the paper carefully, and I want to walk through one specific finding: **the central security theorem is false, and the cipher is much weaker than advertised.** The trail I'm about to show you takes about 100 lines of Python to find. The paper claims it's mathematically impossible.

## Why "active S-boxes" matter

Before I can show you the bug, I need to explain the metric the paper uses to argue NDN is secure.

Every block cipher worth its salt has a **non-linear component** — usually a small lookup table called an **S-box** (substitution box). The S-box is the part that prevents attackers from solving simple equations to recover the key. Everything else in the cipher (XORs, bit shuffles, permutations) is linear, which is fast and cheap but not, by itself, secure.

When cryptographers want to argue that a cipher resists *differential cryptanalysis* — the most basic and historically dangerous attack family, dating to the 1980s — they count something called **active S-boxes**.

Here's the idea. A differential attacker doesn't try to recover the key directly. Instead, they encrypt many *pairs* of plaintexts that differ in some specific way (say, only the first bit is different) and observe how that difference *propagates* through the cipher's rounds. If the attacker can find a difference pattern that survives many rounds with surprisingly high probability, they can use it to recover key bits.

An S-box is "active" in a particular attack trail if its input has a nonzero difference between the two encryptions in the pair. Each active S-box reduces the attacker's success probability by a factor — for NDN's S-box, by 2⁻². So if a 3-round trail has, say, 15 active S-boxes, the attacker's best probability across those rounds is 2⁻³⁰. To attack a 64-bit cipher, the attacker needs probability much larger than 2⁻⁶⁴; otherwise the attack costs more than just guessing the key.

**More active S-boxes → harder to attack.** That's the whole game.

## The paper's central theorem

In Section 4.1, the NDN paper states this as **Theorem 3**:

> *Any three consecutive rounds of the NDN cipher activate at least 15 S-boxes.*

From this, the paper derives all of its differential security claims:

- 3 rounds → MDP ≤ (2⁻²)¹⁵ = 2⁻³⁰
- 12 rounds → MDP ≤ (2⁻²)⁶⁰ = 2⁻¹²⁰
- 18 rounds → MDP ≤ (2⁻²)⁹⁰ = 2⁻¹⁸⁰

The claimed 2⁻¹²⁰ bound for 12 rounds is comfortably below 2⁻⁶⁴, so the paper concludes that NDN-80 is differentially secure with a wide margin.

This single theorem is doing enormous work. If it's wrong, every downstream security claim in the paper falls with it.

## How the paper "proves" it

Here's the part that should have raised an eyebrow at peer review. The paper's proof of Theorem 3 doesn't use any of the standard tools for this kind of result — no MILP solver, no SAT-based search, no explicit case analysis covering all input differences. It checks **four hand-picked input differences**:

1. `(0x1000, 0, 0, 0)`
2. `(0x1001, 0, 0, 0)`
3. `(0x0001, 0, 0, 0x0001)`
4. `(0, 0x0001, 0, 0)`

For each, the authors trace the difference through three rounds, count active S-boxes, and confirm they get at least 15. Then the paper states:

> *Based on test conditions and results, if ΔP1 contains more active nibbles, the minimum number of active S-boxes after three rounds remains at least 15, as observed in experiments. Consequently, any three-round differential characteristic of NDN will have at least 15 active S-boxes under all possible input conditions.*

That's the entire argument. Four examples and an "as observed in experiments" hand-wave.

This is not how active S-box bounds are proved in modern lightweight cryptography. The standard approach (Mouha et al. 2011, Sun et al. 2014) is to encode the cipher's structure as a Mixed-Integer Linear Program (MILP) and let the solver search the *entire* space of differential trails. PRESENT, SKINNY, GIFT, and every credible recent design ships with MILP-verified active S-box bounds. NDN does not.

So I tested it.

## A trail with 4 active S-boxes

Pick the input difference `(dP1, dP2, dP3, dP4) = (0, 0x0008, 0, 0)`. Just one bit flipped, and crucially, that bit is in the **second 16-bit chunk** — not the first or fourth.

This matters because of how NDN's round function works. Each round applies two F-functions: one to `dP1` (the left side) and one to `dP4` (the right side). Then it XORs and swaps things around so that on the next round, different chunks get processed.

If we put our difference in `dP2`, then in round 1, both F-functions get **zero input**. Zero input means zero S-boxes are activated. The difference passes through round 1 untouched, sliding from `dP2` to `dP4` thanks to the cipher's swap pattern.

Let me trace it precisely:

| Round | State `(dP1, dP2, dP3, dP4)` | Active S-boxes called |
|---|---|---|
| Start | `(0, 0x0008, 0, 0)` | — |
| After R1 | `(0, 0, 0, 0x0008)` | **0** (both F-inputs were zero) |
| After R2 | `(0x910a, 0, 0x0008, 0)` | **1** (F_r processed `0x0008`) |
| Round 3 F-calls | F_l processes `0x910a`, F_r processes `0` | **3** (`0x910a` has 3 active nibbles) |

**Total active S-boxes over 3 rounds: 0 + 1 + 3 = 4.**

Not 15. Four.

The paper's main theorem — and every security bound that depends on it — is wrong by a factor of nearly four.

## Why this works

The paper's proof failed for a structural reason worth understanding. The four hand-picked test cases all placed the difference on `dP1` or `dP4` — exactly the chunks that get fed into F-functions in round 1. So in those test cases, an active S-box gets called immediately, and the count starts climbing right away.

But NDN's swap pattern is symmetric across all four chunks. An attacker who places their difference in `dP2` or `dP3` instead gets a free ride through round 1, because the difference doesn't reach an F-function until *after* the swap. The paper's proof never checks this case. It's the cryptographic equivalent of testing a lock by trying four specific keys and concluding it's pickproof.

The miracle of the cipher's design — that it always activates many S-boxes — turns out to be conditional on the attacker cooperating by attacking the obvious chunks. They won't.

## What this means for security

Let me put numbers to the impact.

**The paper's claim:** 12-round NDN-80 has differential probability ≤ 2⁻¹²⁰. Differential cryptanalysis is infeasible.

**What the trail above implies:** if 4 active S-boxes is achievable in 3 rounds, then by extending the same idea, roughly 16 active S-boxes is achievable in 12 rounds. That gives differential probability ≥ 2⁻³². For a 64-bit block cipher, this is well above the 2⁻⁶⁴ threshold where differential attacks become practical.

I want to be careful here: the trail I've shown is a *characteristic* — an upper bound on what's achievable, not a fully verified attack. Turning it into a working key-recovery attack requires more work: you'd need to verify the actual differential probability across all keys, build a key-recovery strategy on top of it, and analyze the data complexity. That's the paper a cryptanalyst would write next.

But the design margin the original paper claims — a comfortable 2⁻¹²⁰ versus the 2⁻⁶⁴ threshold — has evaporated. NDN-80 may not be differentially secure at all. The 18-round NDN-128 has more rounds and more margin to play with, but its security argument depends on the *same* broken theorem.

## How this got published

I don't want to be too harsh on the authors, but I also don't want to soften this: the "proof" of Theorem 3 should not have survived peer review. Four hand-picked examples plus "as observed in experiments" is not a proof; it's a vibe. Any reviewer familiar with how active S-box bounds are established in 2025 would have asked for an MILP-verified result, which is now standard practice.

IJCNA isn't a top-tier venue for cryptography. The strongest cryptanalysis appears at venues like CRYPTO, EUROCRYPT, and FSE/ToSC, where reviewers would catch this kind of issue immediately. But papers like this still get cited, and worse, occasionally get implemented. If you're looking at lightweight cipher proposals and considering one for a real product, the existence of an MILP-verified active S-box bound — done by the designers, not just claimed in prose — is the bare minimum hygiene check.

## The lesson

Here's what I take away from this.

A cipher's security argument is a chain. If any link breaks, the whole argument fails. NDN's chain is roughly: "S-box DDT max is 2⁻²" → "≥15 active S-boxes per 3 rounds" → "differential probability ≤ 2⁻¹²⁰" → "differential cryptanalysis infeasible." Break the second link and everything after it collapses.

The middle link — the active S-box bound — is also the *hardest* link to verify. It's where designers cut corners. It requires either deep manual case analysis or a serious tool like MILP. If you read a lightweight cipher paper and the authors prove this bound by checking a few examples and waving their hands, that's a red flag. If they used MILP, they'll say so prominently.

NDN's authors didn't. And what they claimed, isn't.

---

*Code reproducing the 3-round trail is straightforward — about 100 lines of Python implementing NDN's S-box, bit transformation tables (BT1 and BT2 from the paper), and the XOR-shift diffusion layer. The trail above is the best one found by exhaustive search over 1-active-S-box starting differences placed in each of the four chunk positions.*
