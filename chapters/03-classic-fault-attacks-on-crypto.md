# Classic Fault Attacks on Cryptography

Fault attacks on cryptography share a pattern: the attacker obtains a faulty output, often together with a correct one, and uses the algebraic structure of the algorithm to turn the difference into key information. This chapter walks through the classic attacks from the literature: the Bellcore attack on RSA-CRT, attacks on square-and-multiply exponentiation, safe-error attacks, differential fault analysis on AES, and fault attacks on elliptic curves. The goal is to understand why each works so that countermeasures (Chapter 5) make sense.

## RSA-CRT: the Bellcore attack

RSA signing computes `s = m^d mod n`. Implementations use the Chinese Remainder Theorem (CRT) for speed: compute `s_p = m^{d_p} mod p` and `s_q = m^{d_q} mod q`, then recombine. Aumasson's *Serious Cryptography* (2017) explains the attack attributed to Boneh, DeMillo and Lipton (1997): if a fault corrupts the computation of `s_q` but not `s_p`, the faulty signature `s'` is still correct modulo `p` and wrong modulo `q`. The difference `s − s'` is then a multiple of `p`, so

```text
p = gcd(s - s', n)
```

and the key is broken. A variant (credited to Lenstra) needs only the faulty signature and the message: `gcd(s'^e − m, n) = p`. Aumasson notes that the attack applies to deterministic RSA signatures such as full-domain-hash schemes; randomised padding like PSS prevents the attacker from knowing the exact message representative, which blocks the simple form. Anderson's *Security Engineering* (2008) adds Shamir's observation that even a CPU with a single consistently wrong multiplication result can produce such a split-correctness signature.

What makes the attack devastating is that **one** fault, of almost any kind, in one half of the CRT computation suffices. No precise bit-flip control is needed.

Countermeasure in one line: verify the signature (`s^e mod n == m`) before releasing it, and design that verification to resist faults itself (Chapter 5).

## Attacks on square-and-multiply

Without CRT, RSA exponentiation is usually left-to-right square-and-multiply. Schmidt and Herbst's "A Practical Fault Attack on Square and Multiply" (FDTC 2008, presented by IAIK Graz) review earlier models: bit flips in the exponent or intermediate values (Boneh et al. 1997; Bao et al. 1997; Joye et al. 1997), safe errors (Yen and Joye 2000), and random faults in intermediate values (Boreale 2006), and present a practical attack in which faults hit the square-and-multiply loop on a real device. The common idea: a fault at loop iteration *i* produces an output whose relationship to the correct one depends on the exponent bits processed after *i*, so the attacker can recover the exponent piece by piece by guessing and checking.

## Safe-error attacks

Safe-error attacks exploit operations whose result is sometimes discarded. In "square-and-multiply-always", a dummy multiplication runs when the exponent bit is 0 to equalise timing. If the attacker faults the multiplication and the output is still correct, the multiplication was a dummy, so the bit was 0. The fault does not need to produce a useful value; the attacker learns from **whether the output changed**. This is why "balanced" side-channel countermeasures can create fault vulnerabilities, and why Yen and Joye's work is cited in both literatures.

## Differential fault analysis on AES

Biham and Shamir introduced differential fault analysis (DFA) against DES in 1997. For AES, the best-known attack is Piret and Quisquater's (CHES 2003): a single random byte fault injected in the state between the MixColumns operations of rounds 8 and 9 (of AES-128) spreads through MixColumns into a predictable pattern of four byte differences per column at the output. Comparing correct and faulty ciphertexts for the same plaintext constrains the last round key; a small number of faulty ciphertexts reduces the key space to something brute-forceable. Later work refined this to fewer faults and to other fault positions.

| Fault position (AES-128) | What the attacker compares | Rough idea |
|---|---|---|
| Last round, before SubBytes | Correct vs faulty ciphertext | Each faulty byte constrains one key byte |
| Round 9 state byte | Correct vs faulty ciphertext | Fault spreads to four bytes; solve per column |
| Round 8 state byte | Correct vs faulty ciphertext | Fault spreads to all 16 bytes; very few pairs needed |
| Key schedule | Faulty round keys | Recover key schedule relations |
| Persistent S-box fault | Many faulty ciphertexts, no correct ones needed | Statistical bias reveals key |

Abdellatif, Hériveaux and Thillard's ESP32-V3 talk (Black Hat USA 2022) cites Yifan Lu's "Attacking Hardware AES of PlayStation with DFA" (2019) as an example of DFA against hardware AES engines; fixed-function crypto engines are as exposed as software implementations if they lack internal checks.

## Elliptic curve fault attacks

Elliptic curve scalar multiplication has its own fault attacks:

- **Invalid-point / invalid-curve faults.** Biehl, Meyer and Müller, "Differential Fault Attacks on Elliptic Curve Cryptosystems" (CRYPTO 2000), showed that faulting the base point or curve parameters can move the computation onto a weaker curve where discrete logarithms are easy. Countermeasure: check that input and output points lie on the correct curve.
- **Sign-change and twist faults.** Faults that move points to the quadratic twist.
- **Nonce faults in ECDSA.** Faulting nonce generation or the nonce's use can produce biased nonces; lattice techniques recover the key from enough biased signatures.
- **Deterministic signature faults.** With deterministic nonces (RFC 6979, EdDSA), signing the same message twice yields the same nonce. A fault in one signing produces two signatures with the same nonce but different results, which can reveal the key. Implementations add verification or randomisation to defend.

The *Handbook of Elliptic and Hyperelliptic Curve Cryptography* (Cohen, Frey et al., 2005) treats side-channel and fault countermeasures for curve arithmetic in detail.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally with Python 3. Pure software simulation of a fault; no hardware involved.

```python
# Simulate the Bellcore attack on textbook RSA-CRT (toy key; never use for real crypto).
from math import gcd
from Crypto.Util.number import getPrime, inverse  # pip install pycryptodome
p, q = getPrime(512), getPrime(512); n = p*q; e = 65537
d = inverse(e, (p-1)*(q-1)); dp, dq, qinv = d % (p-1), d % (q-1), inverse(q, p)
def sign_crt(m, fault=False):
    sp = pow(m, dp, p)
    sq = pow(m, dq, q)
    if fault: sq ^= 1                     # simulated single-bit fault in the q half
    h = (qinv * (sp - sq)) % p
    return sq + h*q
m = 0x1234567890abcdef
s, s_bad = sign_crt(m), sign_crt(m, fault=True)
print("recovered p:", gcd(s - s_bad, n) == p)
print("Lenstra variant:", gcd(pow(s_bad, e, n) - m, n) == p)
def sign_crt_checked(m, fault=False):
    s = sign_crt(m, fault)
    if pow(s, e, n) != m: raise ValueError("fault detected")
    return s
```

1. Run the script and confirm both recovery methods print `True`.
2. Call `sign_crt_checked(m, fault=True)` and confirm the fault is detected.
3. Discuss: what if the attacker faults the verification comparison itself? (Chapter 5 addresses this.)

## Common pitfalls

- **Releasing unverified CRT signatures.** One fault breaks the key.
- **Assuming randomised padding is enough everywhere.** It blocks the simple Bellcore form, not all fault attacks.
- **Balancing for side channels without thinking about safe errors.** Dummy operations leak under faults.
- **Trusting hardware crypto engines blindly.** Fixed-function AES and RSA blocks can be faulted too.
- **Skipping point validation in ECC.** Invalid-curve faults exploit it.

## Exercises

1. Extend the worked example: fault `sp` instead of `sq` and confirm the attack recovers `q`.
2. Implement AES-128 in Python (or use a readable reference), inject a random byte fault in round 9, and verify the four-byte difference pattern in the ciphertext.
3. Write a toy ECDSA signer with deterministic nonces, simulate a fault in one signature, and explain how the two outputs could leak the key.

## Further reading

- Aumasson, *Serious Cryptography* (No Starch, 2017), "The Bellcore Attack on RSA-CRT"
- Boneh, DeMillo and Lipton, "On the Importance of Checking Cryptographic Protocols for Faults", EUROCRYPT 1997
- Schmidt and Herbst, "A Practical Fault Attack on Square and Multiply", FDTC 2008
- Piret and Quisquater, "A Differential Fault Attack Technique against SPN Structures, with Application to the AES and KHAZAD", CHES 2003
- Biehl, Meyer and Müller, "Differential Fault Attacks on Elliptic Curve Cryptosystems", CRYPTO 2000
- Anderson, *Security Engineering*, 2nd edition (2008)
