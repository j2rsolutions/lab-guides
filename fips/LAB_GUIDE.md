# The FIPS Lab — Lab Guide

*Self-paced version of the labs. Everything runs on your own lab server — nothing on your laptop.*

> [!NOTE]
> This lab is about **provable** cryptography, so its "numbers" are states, not counts: is the
> validated module *active*, is a non-approved algorithm *refused*, does approved crypto *still work*.
> The whole point is the difference between "crypto that happens to work" and "crypto you can put in
> an audit package." Output shown below is illustrative — **your** seat's output is the evidence.

**Table of contents**

- [Lab 0 — Get your seat](#lab-0--get-your-seat)
- [Lab 1 — Why FIPS at all (~10 min)](#lab-1--why-fips-at-all-10-min)
- [Lab 2 — Prove the stock image is *not* validated (~10 min)](#lab-2--prove-the-stock-image-is-not-validated-10-min)
- [Lab 3 — Swap the image, inherit FIPS mode (~10 min)](#lab-3--swap-the-image-inherit-fips-mode-10-min)
- [Lab 4 — Watch a non-approved algorithm get refused (~10 min)](#lab-4--watch-a-non-approved-algorithm-get-refused-10-min)
- [Lab 5 — Your code on the FIPS image (~15 min)](#lab-5--your-code-on-the-fips-image-15-min)
- [Lab 6 — The compliance evidence package (~10 min)](#lab-6--the-compliance-evidence-package-10-min)

## The situation

Your product just entered a regulated deal — federal, or a customer who inherits federal
requirements — and the security questionnaire has a line item that stops the sale cold:

> "All cryptographic modules must be **FIPS 140-3 validated**. Provide evidence."

Your service uses TLS. It hashes passwords. It works. But "uses crypto" and "uses a validated
cryptographic module *in FIPS mode*" are different claims, and only the second one closes the deal. The
difference isn't the code you wrote — it's the crypto library underneath and *how it's configured*.
This lab makes that difference visible, provable, and — with a Chainguard FIPS image — a base-image
swap rather than a research project.

---

## Lab 0 — Get your seat

1. Open the portal — **the URL is in your welcome email** (and on your table card at in-person events).
2. Log in with the username and password on your table card (`lab01`…`labNN`).
3. Click **Your Lab Server (labNN)**. A terminal opens in the browser tab. (If you also see a
   **desktop** tile, that's a browser for later — the terminal is where the lab happens.)
4. Clipboard: press **Ctrl+Alt+Shift** to open the side menu, paste into its clipboard box, then paste normally.

Two images are pre-cached on your seat and two tiny scripts drive the whole lab:

```bash
cd ~/labs/fips && ls
docker images | grep -E 'fips-demo|python'
```

| Image | What it is |
|---|---|
| `fips-demo:stock` | the public `cgr.dev/chainguard/python` — the image your service uses today |
| `fips-demo:fips` | Chainguard `python-fips` — same Python, FIPS-validated OpenSSL module, **FIPS mode on** |

| Script | What it proves |
|---|---|
| `./posture.sh <image>` | which OpenSSL is behind Python, and whether it is in FIPS mode |
| `./refuse-weak.sh <image>` | the money shot: does the image *refuse* a non-approved algorithm? |

The demo is Python-native (`hashlib`), so what you see is exactly what your application code would see.

---

## Lab 1 — Why FIPS at all (~10 min)

FIPS 140-3 is the US standard for validated cryptographic modules. "Validated" means an accredited lab
tested the *module* and NIST issued a certificate — it is **not** the same as "uses strong algorithms."
A build can use AES-256 and SHA-256 everywhere and still fail the requirement, because the *module*
isn't validated or isn't running in its approved (FIPS) mode.

Keep the two claims side by side for the rest of the lab:

| Claim | What it means | Who cares |
|---|---|---|
| **approved algorithm** | AES, SHA-2, RSA, ECDSA, ML-KEM… are on NIST's list | developers |
| **validated module in FIPS mode** | the specific library build was tested, and it's *running in the mode that was tested* — which also means it **refuses** non-approved algorithms | the auditor |

The auditor asks for the second one. Everything below is about making that state visible.

---

## Lab 2 — Prove the stock image is *not* validated (~10 min)

Look at what's providing crypto in the stock image, and whether it will compute a digest that FIPS
mode must refuse (MD5):

```bash
./posture.sh fips-demo:stock
```

```
OpenSSL: OpenSSL 3.x.x ...
MD5:    computed  -> NOT in FIPS mode
SHA-256: 2d711642b726b044 ... (approved, always works)
```

The library works. It computes strong hashes. It *also* happily computes MD5 — which is the tell:
a validated module in FIPS mode would have refused it. This image is running the **default** OpenSSL
provider, not the validated module in its approved mode. That is exactly the gap the questionnaire
is probing.

Write down: **stock = not FIPS.** The app still serves — "works" was never the question.

---

## Lab 3 — Swap the image, inherit FIPS mode (~10 min)

Same check against the Chainguard FIPS image:

```bash
./posture.sh fips-demo:fips
```

```
OpenSSL: OpenSSL 3.x.x ...
MD5:    REFUSED   -> FIPS mode active (ValueError)
SHA-256: 2d711642b726b044 ... (approved, always works)
```

Two things changed and one didn't:

- **MD5 is refused.** Only a module running in FIPS mode does that. This is the state the auditor
  wants to see.
- **The OpenSSL behind Python is Chainguard's FIPS-validated build**, with the FIPS provider active
  at startup — no environment variables, no config you have to remember to set.
- **SHA-256 still works.** Approved crypto is unaffected. Your application keeps running.

Note what you did to get here: **nothing.** No code change, no `openssl.cnf`, no runtime flag. The FIPS
image ships with the validated module loaded and FIPS mode *on by default*. In your own service the
change is one line: the `FROM`.

**Log it: FIPS provider active on `fips-demo:fips`.**

---

## Lab 4 — Watch a non-approved algorithm get refused (~10 min)

FIPS mode isn't just "strong crypto available" — it's "non-approved crypto *refused*." That refusal
is the teaching moment, so run it on its own, on both images, and read the difference:

```bash
./refuse-weak.sh fips-demo:stock
./refuse-weak.sh fips-demo:fips
```

```
  MD5 computed — NOT FIPS mode
  MD5 REFUSED — FIPS mode active (as it should be)
```

See the refusal for yourself, from Python, the way your code would:

```bash
docker run --rm --entrypoint python fips-demo:fips -c 'import hashlib; hashlib.md5(b"x")'
```

```
Traceback (most recent call last):
  ...
ValueError: [digital envelope routines] unsupported
```

That's the difference between *hoping* nobody used MD5 and *proving* nobody **could**. A validated
module in FIPS mode actively closes the weak path; the only path left open is the approved one.

> **Facilitator beat — "but my app uses MD5 for checksums, not security."** Python has
> `hashlib.md5(data, usedforsecurity=False)` for non-cryptographic uses (cache keys, ETags). Try it on
> the FIPS image:
> `docker run --rm --entrypoint python fips-demo:fips -c 'import hashlib; print(hashlib.md5(b"x", usedforsecurity=False).hexdigest())'`
> If it computes, the image lets non-security uses through while refusing MD5 *as security* — exactly
> the distinction an auditor draws. If it is refused too, the image runs the FIPS provider *only*, the
> strictest posture, and even checksums must move to SHA-2. Either answer is a finding worth knowing
> before the audit, and the refusal is how you find every call site that hasn't moved.

---

## Lab 5 — Your code on the FIPS image (~15 min)

The point of Lab 3 was "it's a `FROM` line." Prove it with real code. Write a small service that does
the crypto your product actually does — hash a token, verify an HMAC — and run it on both images:

```bash
cat > ~/labs/fips/svc.py <<'EOF'
import hashlib, hmac, ssl
print("OpenSSL:", ssl.OPENSSL_VERSION)
secret = b"demo-secret"
token = hashlib.sha256(b"user:42").hexdigest()
tag = hmac.new(secret, token.encode(), hashlib.sha256).hexdigest()
print("token  :", token[:16], "...   (SHA-256)")
print("hmac   :", tag[:16], "...   (HMAC-SHA-256)")
print("verify :", hmac.compare_digest(tag, hmac.new(secret, token.encode(), hashlib.sha256).hexdigest()))
try:
    hashlib.md5(b"legacy-checksum")
    print("md5    : computed  -> this image is NOT in FIPS mode")
except ValueError:
    print("md5    : REFUSED   -> FIPS mode active")
EOF

docker run --rm -v ~/labs/fips/svc.py:/svc.py --entrypoint python fips-demo:stock /svc.py
echo "----"
docker run --rm -v ~/labs/fips/svc.py:/svc.py --entrypoint python fips-demo:fips  /svc.py
```

Same code, same Python, same output for everything approved — and one line that differs: the FIPS
image refuses the legacy call. **Your application didn't change; its cryptographic posture did.**

Now ship it the way you'd ship for real — a Dockerfile whose base is the FIPS image:

```bash
cat > ~/labs/fips/Dockerfile <<'EOF'
FROM fips-demo:fips
COPY svc.py /app/svc.py
ENTRYPOINT ["python", "/app/svc.py"]
EOF
docker build -q -t svc:fips ~/labs/fips && docker run --rm svc:fips
```

In your own repo, that `FROM` becomes `cgr.dev/<your-org>/python-fips:latest` (or the `-fips` variant
of whatever runtime you ship: `nginx-fips`, `node-fips`, `jdk-fips`, …). Same pattern, same result.

> **[v1.1] Post-quantum.** Regulators are already naming ML-KEM for "harvest now, decrypt later"
> mandates, and the honest question is how PQC interacts with FIPS validation (PQC algorithms are
> only now entering CMVP). This lab leaves that as a facilitator slide rather than a hands-on leg —
> there is no PQC handshake staged on the seat in v1.

---

## Lab 6 — The compliance evidence package (~10 min)

Turn the states you proved into artifacts the security team can file:

```bash
mkdir -p ~/fips-evidence && cd ~/fips-evidence
~/labs/fips/posture.sh     fips-demo:fips > posture.txt      # module + FIPS mode active
~/labs/fips/refuse-weak.sh fips-demo:fips > refusal.txt      # non-approved algorithm refused
docker run --rm svc:fips                  > svc-on-fips.txt  # YOUR code, on the FIPS image

# provenance: the FIPS story and the supply-chain story are one story
# (org-entitled images are signed by Chainguard's build infrastructure — the
# certificate issuer is issuer.enforce.dev, not GitHub Actions)
FIPS_IMG=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep python-fips | head -1)
cosign verify --certificate-oidc-issuer=https://issuer.enforce.dev \
  --certificate-identity-regexp='^https://issuer\.enforce\.dev/' "$FIPS_IMG" > signature.txt 2>&1 && echo "signature verified"
syft "$FIPS_IMG" -o spdx-json > sbom.spdx.json
ls -l
```

You hand the reviewer: the active-module proof, the refusal proof, your own service's output on the
validated image, the image signature, and an SBOM — plus a pointer to Chainguard's FIPS 140-3
validation certificate for the module itself (Chainguard publishes the CMVP certificate number for its
FIPS images; cite it in the package).

---

## Your evidence log

| Check | `fips-demo:stock` | `fips-demo:fips` |
|---|---|---|
| FIPS-validated module, FIPS mode active? | | |
| MD5 (non-approved) refused? | | |
| SHA-256 / HMAC-SHA-256 (approved) still work? | | |
| your `svc.py` runs unchanged? | | |
| signature verified + SBOM captured? | n/a | |

## Next steps

1. **Run Lab 2 against one of your real service images** — does it compute MD5? Then it isn't in FIPS mode.
2. **Swap the base to the mapped `-fips` image** and re-run Labs 3–5 — module active, weak crypto
   refused, your code unchanged. Every refusal you hit is a real finding you'd otherwise have met at audit.
3. **Bring your questionnaire to a working session** — J2R maps your stack to the validated modules and
   helps assemble the evidence package.

Jonathan Spigler · J2R Solutions · jonathan@j2rsolutions.io
Phil Brooks · Partner Solutions Engineer, Chainguard
