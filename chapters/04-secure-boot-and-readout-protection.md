# Bypassing Secure Boot and Readout Protection: Published Research

Cryptographic algorithms are not the only fault targets. The checks *around* cryptography are often easier: the branch that decides whether a signature verified, the bit that enables secure boot, the register that says whether flash may be read. This chapter surveys published research on these targets, abstracted into patterns. It names vendors only where the research did and stays at the level of the published papers and talks; it is not a recipe for attacking any product.

## Why boot is a favourite target

Timmers and Spruyt (Black Hat Europe 2016) argue that a small boot ROM has a small logical attack surface, so implementations without software vulnerabilities likely exist; when that is the case, other attacks must be used. Fault injection needs no logical flaw, and typical boot targets were not properly protected. Boot is also when valuable assets are present that may not be available later: boot ROM code itself, keys used to decrypt the next stage, and crypto engines in privileged modes.

## Pattern 1: the verification decision

Timmers and Spruyt list the "classic" bypasses:

| Target | What the fault does | Notes from the talk |
|---|---|---|
| Hash comparison | `memcmp` of computed vs expected hash returns "equal" or its result is ignored | Multiple instructions in a typical comparison can each be faulted |
| Signature check call | The branch after `verify()` takes the "boot" path | Bypasses can happen inside the function or in its caller |
| Failure loop | `while(1);` on failure is exited | Timing is easy because the target waits forever; better to reset or wipe keys |
| Secure boot enable bit | The OTP bit read after reset returns "disabled" | Often read early, just after reset |

The infinite-loop item connects to a long history: smartcard "unloopers" were devices that glitched cards out of lockout loops. Datko, Quartier and Belyayev (DEF CON 25, 2017) demonstrated loop and password-check glitches on an STM32F205-based training board, concluding that the chip was susceptible to fault attacks.

## Pattern 2: combined attacks

Timmers and Spruyt also showed faults that create *software* vulnerabilities:

- **Corrupting copy arguments.** Faulting the size argument of a `memcpy` (their example changed `0x1000` into a larger value) creates a buffer overflow in boot code that has no exploitation mitigations.
- **Loading the program counter.** On 32-bit ARM, flipping one bit in an `LDMIA` encoding adds `PC` to the loaded registers, so attacker-controlled data from the image becomes the next instruction address.
- **"Wild jungle jump".** Glitching while an image is loaded but before it is decrypted, hoping a corrupted pointer lands in attacker-controlled code; more pointers near the code means higher probability.

The lesson: even if every authenticity check is hardened, faults elsewhere can hand control to the attacker before checks matter.

## Pattern 3: architecture-level fault windows

Adam Zabrocki and Alex Matrosov's talk "Glitching RISC-V chips: MTVEC corruption for hardening ISA" described a design issue rather than a code bug. The RISC-V specification does not define the reset value of `MTVEC`, the machine-mode trap vector register. They observed that many implementations reset it to 0; if an exception occurs before boot code initialises `MTVEC`, the core jumps to address 0, faults again, and spins in a stable exception loop at the highest privilege level, repeatedly reading a register an attacker can try to glitch. Their slides describe how a glitched `MTVEC` value pointing into attacker-filled memory yields code execution, and record that a CVE (CVE-2021-1104) was allocated for the affected implementation. The countermeasure is architectural: define safe reset values and initialise trap vectors as the very first instructions.

## Pattern 4: readout protection

Microcontrollers offer readout protection (RDP, CRP or similar) to stop debuggers reading flash. Published work shows these mechanisms often depend on configuration bits read at power-up, making them fault and logic targets:

- **STM32F0, 2017.** Obermaier and Tatschner, "Shedding too much Light on a Microcontroller's Firmware Protection" (USENIX WOOT 2017), presented three weaknesses with proof-of-concept attacks, including Cold-Boot Stepping, downgrading RDP Level 2 to Level 1, and exploiting the debug interface, all with low-cost equipment. The findings were assigned CVE-2017-18347.
- **Multiple MCUs, 2019.** Bozzato, Focardi and Palmarini (IACR TCHES 2019) reported firmware extraction from six microcontrollers by three vendors using shaped voltage glitches, with responsible disclosure.
- **ESP32, 2019 and 2022.** As recounted by Abdellatif, Hériveaux and Thillard (Black Hat USA 2022), LimitedResults showed in 2019 that power glitching during power-up could disturb eFuse protection bits on ESP32-V1, breaking secure boot and flash encryption. The vendor's ESP32-V3 added fault-injection hardening; the 2022 work then used EMFI and side-channel analysis against the V3's flash encryption key handling.

## Pattern 5: trusted execution environments

CLKSCREW (Tang, Sethumadhavan and Stolfo, 2017) showed that TrustZone on a commodity phone could be faulted from the normal world through power management, leaking AES keys and loading self-signed code into the secure world. This removes the physical-access assumption and is relevant to any SoC that exposes fine-grained voltage and frequency control to less trusted software. See **tee-attack-engineering** for TEE attack surfaces more broadly.

## What defenders should take away

| Pattern | Defensive question |
|---|---|
| Verification decision | Does a single skipped instruction change the boot decision? |
| Combined attacks | Are lengths, pointers and copy sizes bounded and re-checked? |
| Fault windows | Is there any stable loop or early state where a privileged register is read repeatedly? |
| Readout protection | Are protection bits read redundantly and from fault-resistant storage? |
| TEE | Can less trusted software change voltage or clock of secure code? |

Chapter 5 turns these questions into countermeasures.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally. Uses your own lab board (for example a ChipWhisperer training target) running your own bootloader code.

Build a mini secure-boot on your lab target and attack your own check.

1. Write a tiny "bootloader" for the training target that computes SHA-256 over a fixed "application" buffer (use a small public-domain implementation), compares it with a stored hash using `memcmp`, prints `BOOT` on match or `FAIL` and loops on mismatch.
2. Modify the application buffer so the hash no longer matches; confirm `FAIL`.
3. Add a GPIO trigger just before the comparison.
4. Use your Chapter 2 parameter sweep to glitch around the trigger and count how often `BOOT` appears with the wrong image.
5. Record the success rate and the glitch parameters. Do not move on to any commercial device; Chapter 5 hardens this code and you re-measure.

## Common pitfalls

- **Hardening the crypto but not the decision.** The branch after verification is often the weakest point.
- **Infinite loops on failure.** They give attackers unlimited, perfectly timed attempts.
- **Single reads of protection bits.** One fault at power-up flips the outcome.
- **Undefined reset states.** Architectural gaps create stable fault windows.
- **Ignoring software-controlled power management.** It can act as a remote glitcher.

## Exercises

1. Run the worked example and report the bypass rate with the plain `memcmp` check.
2. Replace the failure loop with an immediate reset and measure how the attack changes.
3. Read the Obermaier and Tatschner paper and summarise, in your own words, why RDP state stored in flash option bytes was a weak point.

## Further reading

- Timmers and Spruyt, "Bypassing Secure Boot using Fault Injection", Black Hat Europe 2016
- Obermaier and Tatschner, "Shedding too much Light on a Microcontroller's Firmware Protection", USENIX WOOT 2017: <https://www.usenix.org/conference/woot17/workshop-program/presentation/obermaier>
- Bozzato, Focardi and Palmarini, "Shaping the Glitch", IACR TCHES 2019: <https://tches.iacr.org/index.php/TCHES/article/view/7390>
- Abdellatif, Hériveaux and Thillard, "Unlimited Results: Breaking Firmware Encryption of ESP32-V3", Black Hat USA 2022
- Zabrocki and Matrosov, "Glitching RISC-V chips: MTVEC corruption for hardening ISA" (conference talk slides)
- Datko, Quartier and Belyayev, "Breaking Bitcoin Hardware Wallets", DEF CON 25 (2017)
