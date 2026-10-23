# Fault Models and Threat Model

Fault injection is the deliberate disturbance of a chip so that it computes the wrong thing at the right moment. Timmers and Spruyt's "Bypassing Secure Boot using Fault Injection" (Black Hat Europe 2016) define it as "introducing faults in a target to alter its intended behavior", and illustrate it with a single line of code: `if (key_is_correct)` with a glitch arriving exactly as that branch is evaluated. If the processor skips or corrupts the comparison, the door opens without the key.

For cryptography the stakes are higher still. A single wrong intermediate value in some algorithms is enough to recover the whole private key from the faulty output. This book explains how faults are induced, which classic cryptographic fault attacks the literature describes, how secure boot and readout protection have been bypassed in published research, which countermeasures hold, and how to test your own designs. It is written for designers, evaluators and students working on their own lab targets.

## A short history

Ross Anderson's *Security Engineering* (2nd edition, 2008) recounts that in 1996 he and Markus Kuhn reported that many smartcards could be broken by inserting transients, or glitches, into their power or clock lines. The same book describes **fault induction** more broadly: probing needles that stop a processor's branch logic so it executes instructions linearly ("memory linearisation"), bridging a blown test fuse so factory self-test routines can dump memory, and Adi Shamir's observation that a CPU with a single consistently wrong multiplication result can leak an RSA key when it computes a signature correctly modulo one prime but not the other. Anderson's conclusion is the first countermeasure in this book: "a careful programmer will check the results of critical computations."

In parallel, Boneh, DeMillo and Lipton's 1997 paper "On the Importance of Checking Cryptographic Protocols for Faults" (the "Bellcore attack") made RSA-CRT fault attacks famous, and Biham and Shamir introduced differential fault analysis against symmetric ciphers the same year. Jean-Philippe Aumasson's *Serious Cryptography* (2017) still presents the Bellcore attack as one of the most important attacks in RSA's history.

## Fault models

A **fault model** describes what the attacker can do to the computation, abstracted from the physics. Timmers and Spruyt split models into faults affecting hardware (registers, buses) and faults affecting the software running on it (instruction corruption, data corruption), and warn that the true fault model is hard to predict or prove. Common models:

| Model | Description | Typical cause |
|---|---|---|
| Instruction skip | One or more instructions not executed | Voltage or clock glitch during fetch/decode |
| Instruction corruption | Opcode bits flipped, turning one instruction into another | Glitch on instruction bus or cache |
| Data bit flip | Single or few bits flipped in a register or memory word | Laser, EM pulse |
| Byte or word randomisation | A value replaced by an unknown random value | Voltage glitch, EM |
| Stuck-at / set-reset | Bits forced to 0 or 1 | Laser on SRAM cells, under-voltage |
| Timing violation | A value latched too early or late | Over-clocking, under-voltage |
| Persistent fault | Corrupted constant (for example an S-box table in RAM) affects all later computations | Any method that hits stored data |

Daniel Nemiroff and Carlos Tokunaga's Black Hat USA 2022 briefing on Intel's fault-injection detection circuit explains the underlying physics in timing terms: under-voltage slows logic, over-clocking shortens the cycle, and either way data is latched before it settles. A partially settled bus often reads as zeros, which is why a `JMP` can become a `NOP` on some architectures. Timmers and Spruyt show the same effect at the instruction level: corrupting a few bits of an ARM `LDMIA` instruction can add the program counter to its register list, giving the attacker control of execution.

## Threat model

**Assets.** Secret keys in embedded devices; boot-chain integrity (only signed code runs); readout protection on firmware and secrets; anti-rollback counters; PIN retry counters.

**Attacker capabilities by class.**

| Class | Access | Typical budget | Methods |
|---|---|---|---|
| Opportunistic | Device in hand for minutes | Low; hobbyist tools | Voltage and clock glitching through accessible pins |
| Skilled hobbyist / researcher | Days to weeks, multiple samples | Moderate | Board modification, crowbar glitches, EM probes |
| Professional lab | Weeks, many samples, decapping | High | Laser, precise EM, combined side-channel and fault |
| Software-only (privileged) | Kernel or firmware access, no physical access | Low equipment cost | Abusing DVFS controls (CLKSCREW) |

The software-only row matters. Tang, Sethumadhavan and Stolfo's CLKSCREW work (USENIX Security 2017, presented at Black Hat Europe 2017) showed that a malicious kernel driver on an ARM Android phone could use software-controlled frequency and voltage scaling to induce faults inside TrustZone, extracting AES keys and loading self-signed code, without physical access or special equipment.

**Out of scope.** Recipes against named commercial products, attacks on devices you do not own, and side-channel analysis in depth (see **deep-learning-side-channel-analysis**). Hardware wallets as products are covered in **hardware-wallet-security**.

## Ethics and safety

Work only on targets you own or are authorised to test, and preferably on dedicated training targets. Glitching "too hard", as Timmers and Spruyt note, can permanently damage a chip. High-voltage EM injectors and lasers are personal-safety hazards: follow equipment manuals, use interlocks and eye protection, and never work alone with high-voltage pulse generators. Report vulnerabilities in products to vendors under coordinated disclosure.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally. Runs entirely in software; no hardware needed.

Model instruction-skip faults on a password check before touching hardware. Compile a tiny ARM Cortex-M function and enumerate which single skipped instruction breaks it.

```c
// check.c - target function for fault modelling
int check(const char *in, const char *pw) {
    int ok = 1;
    for (int i = 0; i < 8; i++) if (in[i] != pw[i]) ok = 0;
    return ok;
}
```

1. Build: `arm-none-eabi-gcc -mcpu=cortex-m3 -mthumb -O2 -c check.c && arm-none-eabi-objdump -d check.o`.
2. List each instruction in `check`. For each, ask: if this instruction is skipped, does the function return 1 for a wrong password?
3. Automate it with an emulator: write a short Unicorn Engine (Python) script that loads the object code, runs `check` with a wrong password, and for each instruction address skips it once (advance PC by the instruction length) and records the return value.
4. Rebuild with `-O0` and compare the number of vulnerable instructions.
5. Keep the script; Chapter 5 uses it to measure countermeasures.

## Common pitfalls

- **Assuming one fault model.** Real glitches produce skips, corruptions and data errors mixed together.
- **Trusting source-level reasoning.** Faults act on machine code; the compiler may remove your checks.
- **Treating physical access as rare.** Lost, stolen, returned and second-hand devices give attackers time.
- **Ignoring software-induced faults.** Power-management interfaces can be fault injectors.
- **Unsafe lab practice.** High voltage and lasers need training and interlocks.

## Exercises

1. Run the worked example and report how many single-instruction skips defeat `check` at `-O0` and `-O2`.
2. Extend the emulator script to the "bit flip in a register after instruction N" model and compare results.
3. Write a one-page threat model for a microcontroller-based device you own, assigning each attacker class from the table a realistic goal.

## Further reading

- Timmers and Spruyt, "Bypassing Secure Boot using Fault Injection", Black Hat Europe 2016
- Anderson, *Security Engineering*, 2nd edition (2008), chapters on smartcards and emission security
- Bar-El, Choukri, Naccache, Tunstall and Whelan, "The Sorcerer's Apprentice Guide to Fault Attacks" (IACR ePrint 2004/100): <https://eprint.iacr.org/2004/100>
- Tang, Sethumadhavan and Stolfo, "CLKSCREW: Exposing the Perils of Security-Oblivious Energy Management", USENIX Security 2017
- Nemiroff and Tokunaga, "Fault-Injection Detection Circuits: Design, Calibration, Validation and Tuning", Black Hat USA 2022
- Unicorn Engine: <https://www.unicorn-engine.org/>
