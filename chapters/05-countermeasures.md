# Countermeasures That Hold

Timmers and Spruyt (Black Hat Europe 2016) separate countermeasures into two kinds: hardware countermeasures that **detect** the glitch or fault, and software countermeasures that **lower the probability** of a successful fault without addressing the root cause. Their summary is the honest starting point for this chapter: "You can lower the probability but not rule it out!" Good designs therefore layer detection, redundancy and response, and measure the result on real hardware.

## Layer 1: hardware sensors and detectors

| Countermeasure | Detects | Notes |
|---|---|---|
| Voltage monitors (brown-out, glitch detectors) | Supply excursions | Analog detectors may miss very short or shaped glitches |
| Clock monitors and internal oscillators | Missing, extra or fast clock edges | Datko et al. (DEF CON 25, 2017) found wallets not enabling an MCU's clock security feature |
| Light sensors, active shields | Decapping, laser | Standard in secure elements |
| Temperature sensors | Thermal attacks | Usually combined with others |
| Timing-replica circuits | Any timing violation | Intel's tunable replica circuit (below) |
| Hardware duplication / lockstep | Computation divergence | Two cores or datapaths compare results |

Nemiroff and Tokunaga's Black Hat USA 2022 briefing describes Intel's **tunable replica circuit (TRC)**: a launch flip-flop, a tunable delay chain and a capture flip-flop. Because fault attacks typically aim to cause timing violations, a delay chain calibrated to fail just before real logic fails will flag voltage, clock, EM or temperature attacks. Intel chose it because it is a digital design that ports to new process nodes, covers multiple attack types, and is small. They integrated it to protect the Converged Security and Manageability Engine (CSME), which supports a TPM 2.0 implementation, and calibrate each part during manufacturing by fusing a delay setting derived from the part's own voltage/delay slope, then testing for false positives. The talk notes analog sensors offer precision and independence but may not trigger if, for example, voltage and frequency both increase. The general lesson: **calibrate detectors per part and test for false positives**, or they will either miss attacks or brick legitimate devices.

## Layer 2: algorithmic countermeasures for cryptography

**Verify before release.** For RSA-CRT, check `s^e mod n == m` before outputting a signature (Chapter 3). The *Handbook of Elliptic and Hyperelliptic Curve Cryptography* (Cohen, Frey et al., 2005) notes that the obvious countermeasure to fault attacks is a check computation, but that for curve-based cryptography checking amounts to doing the whole computation twice, unlike RSA where the public operation is cheap. That cost difference drives ECC-specific defences:

- **Point validation.** Check that input points, intermediate points and output points lie on the correct curve, which defeats invalid-curve faults in the style of Biehl, Meyer and Müller (CRYPTO 2000).
- **Coherence checks.** Montgomery ladders maintain a known relation between two registers (`R1 − R0 = P`); checking it at the end detects many faults.
- **Randomised nonces in deterministic schemes.** Adding randomness to RFC 6979 or EdDSA nonces breaks the "same message, same nonce" relation exploited by fault attacks, at the cost of determinism.

**Infective computation.** Instead of checking and branching (a branch that can itself be faulted), infective countermeasures arrange that any fault randomises the output so it carries no exploitable information. For AES, a common construction interleaves real rounds with dummy rounds and redundant computations, and multiplies any difference between redundant results into the output. Infective schemes have been broken and repaired several times in the literature, so treat any specific scheme as needing review.

**Redundant computation.** Compute twice (same or different methods, for example encrypt then decrypt) and compare. Temporal redundancy (same hardware twice) is defeated by identical repeated faults; spatial or algorithmic diversity helps.

## Layer 3: software hardening of control flow

These patterns appear in Timmers and Spruyt and in Datko et al.'s designer advice:

- **Double-check decisions, in a way the compiler preserves.** Timmers and Spruyt show that a duplicated `memcmp` check is removed by an optimising compiler. Use `volatile`, separate functions, or compiler barriers, and **read the generated assembly**.
- **Use values with large Hamming distance.** Avoid `0`/non-`0` booleans for security decisions; use constants such as `0x3CA5965A` and `0xC35A69A5`, so a single bit flip or zeroed register cannot produce the "success" value.
- **Count and verify control flow.** Increment a counter at each step and verify the final count before acting (Datko et al.: "Count your functions!", "Check for complete loop completion").
- **Default to failure.** Initialise result variables to failure; only an explicit, verified path sets success.
- **Random delays.** Make trigger timing harder; they slow attackers but do not stop them.
- **Fail hard.** On detection, reset and, after repeated detections, wipe keys. Replace infinite failure loops with resets (Chapter 4).
- **Initialise trap vectors and exception handlers first.** Zabrocki and Matrosov's RISC-V MTVEC work shows why.

```c
// Hardened decision sketch (illustrative; validate on your toolchain and hardware)
#define OK  0x3CA5965Au
#define BAD 0xC35A69A5u
static volatile uint32_t steps;
uint32_t verify_image(const uint8_t *img, size_t len, const uint8_t *expected) {
    volatile uint32_t r1 = BAD, r2 = BAD;
    steps = 0;
    if (ct_memcmp(hash(img, len), expected, 32) == 0) { r1 = OK; } steps++;
    random_delay();
    if (ct_memcmp(hash(img, len), expected, 32) == 0) { r2 = OK; } steps++;
    if (r1 == OK && r2 == OK && steps == 2) return OK;
    return BAD;
}
// Caller: if (verify_image(...) != OK) { reset_and_count_failure(); }  // never loop forever
```

The sketch is not proof of security. It moves the attacker from "one fault" to "several correlated faults", which is the realistic goal.

## Layer 4: system design

- **Do not expose power management to untrusted software.** CLKSCREW (Tang et al., 2017) exploited exactly that.
- **Keep secrets out of general MCUs** when physical attackers are in scope; use certified secure elements for key storage and signing. See **hardware-wallet-security**.
- **Limit attempts.** Monotonic counters in protected storage for PIN tries and boot failures.
- **Bind keys to measured state.** If boot measurements feed key derivation, a glitch that skips verification yields the wrong keys.

## Measuring countermeasures

Report effectiveness as numbers from your own lab:

| Metric | Meaning |
|---|---|
| Success rate per attempt | Faults that produce the attacker's goal / total attempts |
| Detection rate | Attempts that trigger a sensor or software check |
| Mute/reset rate | Attempts that crash or reset the target |
| Time to first success | Wall-clock time in your best parameter region |
| False-positive rate | Detector triggers under normal operation across temperature and voltage corners |

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally on your Chapter 4 lab bootloader and the Chapter 1 emulator script.

1. **Emulator first.** Apply the hardening patterns to the Chapter 1 `check` function. Rerun the Unicorn single-skip script and count vulnerable instructions before and after. Then add a two-skip mode and count pairs.
2. **Assembly review.** Compile with your production flags and confirm the duplicated checks survive in the disassembly.
3. **Hardware.** Flash the hardened bootloader to your lab target. Rerun the exact Chapter 4 glitch campaign (same parameters, same number of attempts).
4. **Compare.** Fill the metrics table for both versions. Expect the success rate to fall; note any rise in resets, which tells you the detectors are firing.
5. **Iterate.** If successes remain, find the faulted instruction by narrowing the delay window and inspecting the code there.

## Common pitfalls

- **Writing double checks the compiler deletes.** Always inspect the binary.
- **Branch-based checks with a single decision point.** The check's own branch is a target.
- **Uncalibrated sensors.** They miss attacks or reset legitimate devices.
- **Temporal redundancy only.** Identical repeated faults defeat compute-twice-and-compare.
- **Claiming immunity.** Countermeasures change success rates; report numbers, not guarantees.

## Exercises

1. Measure single-skip and double-skip vulnerability counts for the plain and hardened `check` in the emulator.
2. Implement RSA-CRT signing with verification in C on your lab target and attempt to glitch both the signature and the verification step.
3. Design a calibration test plan for a voltage-glitch detector covering temperature and supply-voltage corners.

## Further reading

- Timmers and Spruyt, "Bypassing Secure Boot using Fault Injection", Black Hat Europe 2016
- Nemiroff and Tokunaga, "Fault-Injection Detection Circuits: Design, Calibration, Validation and Tuning", Black Hat USA 2022
- Cohen, Frey et al., *Handbook of Elliptic and Hyperelliptic Curve Cryptography* (CRC, 2005), section on countermeasures against fault attacks
- Datko, Quartier and Belyayev, "Breaking Bitcoin Hardware Wallets", DEF CON 25 (2017)
- Bar-El et al., "The Sorcerer's Apprentice Guide to Fault Attacks" (IACR ePrint 2004/100): <https://eprint.iacr.org/2004/100>
