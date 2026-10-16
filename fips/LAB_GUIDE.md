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
3. **Your folder today: FIPS lab.** Sign in once. The portal shows one **folder per lab track**
   running today; your welcome email says which folder is which. For this guide open the folder
   named **FIPS lab**. On a single-lab day there are no folders — the same two tiles sit at the
   portal root as **Your Lab Server (labNN)** and **Your Lab Server desktop (labNN)**. Inside your
   folder you get **two tiles** — both are your own seat:
   - **FIPS lab (labNN)** — a **terminal** in the browser tab. The lab happens here.
   - **FIPS lab desktop (labNN)** — a **desktop** on the *same* server (Firefox, a File
     Manager and a Terminal launcher; it's already logged in, no password to type). Today it's
     for nothing you'll need — there is nothing to browse today. It's optional: in Lab 6 you
     can open your evidence folder in the File Manager.

   Other folders are **other servers**: files and images do not carry across. Everywhere below,
   **your desktop tile** means **FIPS lab desktop (labNN)** — or **Your Lab Server desktop (labNN)**
   on a single-lab day.

   **Keep both open.** A tile normally opens in the same browser tab — **Ctrl-click** (or
   middle-click) a tile to open it in a **new browser tab**, so the terminal and the desktop sit
   side by side. Firefox on the desktop reaches anything listening on your seat
   (`http://localhost:<port>`), and the File Manager reaches every file in your home folder.
   Every desktop step below names a launcher *and* the URL or path to type into Firefox — if the
   launcher isn't on your desktop, type it; same result.
4. Clipboard: press **Ctrl+Alt+Shift** to open the side menu, paste into its clipboard box, then paste normally.

Two images are pre-cached on your seat and two tiny scripts drive the whole lab:

```bash
cd ~/labs/fips && ls
docker images | grep -E 'fips-demo|python'
```

| Image | What it is |
|---|---|
| `fips-demo:stock` | the public `cgr.dev/chainguard/python` — the image your service uses today |
| `fips-demo:fips` | Chainguard `python-fips` — same Python, FIPS-validated OpenSSL module, **FIPS mode on**; also ships Chainguard's *legacy-allowed* provider (on by default) so MD5/SHA-1 still work for **non-security** uses |

| Script | What it proves |
|---|---|
| `./posture.sh <image>` | which OpenSSL is behind Python, whether the validated module is in FIPS mode, and what happens to MD5 — as a checksum and *as security* |
| `./refuse-weak.sh <image>` | the money shot: does the image *refuse* MD5 used as security — and MD5 at all, in strict mode? |

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

Look at what's providing crypto in the stock image: is a validated module running in FIPS mode, and
what does it do with a non-approved algorithm (MD5) — as a checksum, and *as security* (an HMAC tag)?

```bash
./posture.sh fips-demo:stock
```

```
OpenSSL:     OpenSSL 3.x.x ...
FIPS mode:   off       -> NOT a validated module in FIPS mode (get_fips_mode=0)
MD5 digest:  computed  -> nothing here to refuse it
HMAC-MD5:    computed  -> MD5 accepted AS SECURITY: not FIPS mode
SHA-256:     2d711642b726b044 ... (approved, always works)
```

The library works. It computes strong hashes. It *also* happily signs with MD5 — which is the tell: a
validated module in FIPS mode would have refused that. This image is running the **default** OpenSSL
provider, not the validated module in its approved mode (`get_fips_mode=0` says so outright). That is
exactly the gap the questionnaire is probing.

Write down: **stock = not FIPS.** The app still serves — "works" was never the question.

---

## Lab 3 — Swap the image, inherit FIPS mode (~10 min)

Same check against the Chainguard FIPS image:

```bash
./posture.sh fips-demo:fips
```

```
OpenSSL:     OpenSSL 3.x.x ...
FIPS mode:   ACTIVE    -> validated module in its approved mode (get_fips_mode=1)
MD5 digest:  computed  -> allowed for NON-security use only (checksums; FIPS 140-3 IG 2.4.A)
HMAC-MD5:    REFUSED   -> FIPS mode refuses MD5 as security
SHA-256:     2d711642b726b044 ... (approved, always works)
```

Three things changed and one didn't:

- **FIPS mode is active.** `get_fips_mode=1` — the OpenSSL behind Python is Chainguard's
  FIPS-validated build with the FIPS provider active at startup. This is the state the auditor wants
  to see.
- **MD5 *as security* is refused.** Only a validated module running in FIPS mode does that.
- **A plain MD5 *digest* still computes — on purpose.** FIPS 140-3 (IG 2.4.A) permits non-approved
  digests for non-security uses (checksums, ETags, cache keys), and Chainguard ships a *legacy-allowed*
  provider, on by default, so real apps don't fall over on a checksum. Lab 4 shows the switch that
  turns even that off.
- **SHA-256 still works.** Approved crypto is unaffected. Your application keeps running.

Note what you did to get here: **nothing.** No code change, no `openssl.cnf`, no runtime flag. The FIPS
image ships with the validated module loaded and FIPS mode *on by default*. In your own service the
change is one line: the `FROM`.

**Log it: FIPS provider active on `fips-demo:fips`.**

---

## Lab 4 — Watch a non-approved algorithm get refused (~10 min)

FIPS mode isn't just "strong crypto available" — it's "non-approved crypto *refused*." That refusal
is the teaching moment, so run it on its own, on both images, and read the difference. The script
refuses two ways: MD5 **as security** (an HMAC tag — refused as shipped), and MD5 **at all** with the
legacy-allowed provider switched off (`CHAINGUARD_LEGACY_ALLOWED=0`, the strict posture):

```bash
./refuse-weak.sh fips-demo:stock
./refuse-weak.sh fips-demo:fips
```

```
  HMAC-MD5 computed — MD5 accepted as security: NOT FIPS mode
  MD5 digest, strict mode: computed — nothing to refuse it: NOT FIPS mode
```
```
  HMAC-MD5 REFUSED — FIPS mode active (as it should be)
  MD5 digest, strict mode: REFUSED — only the validated module is loaded
```

See the refusal for yourself, from Python, the way your code would:

```bash
docker run --rm -e CHAINGUARD_LEGACY_ALLOWED=0 --entrypoint python fips-demo:fips -c 'import hashlib; hashlib.md5(b"x")'
```

```
Traceback (most recent call last):
  ...
_hashlib.UnsupportedDigestmodError: [digital envelope routines] unsupported
```

(Drop the `-e` and ask for an HMAC instead — `hmac.new(b"k", b"x", hashlib.md5)` — and you get the
same refusal as a `ValueError`, with no switch at all.)

That's the difference between *hoping* nobody used MD5 for security and *proving* nobody **could**. A
validated module in FIPS mode actively closes the weak path; the only path left open is the approved one.

> **Facilitator beat — "but my app uses MD5 for checksums, not security."** That is exactly the
> distinction the standard draws, and the image draws it for you: as shipped, `hashlib.md5(data)`
> computes (non-security use, permitted by FIPS 140-3 IG 2.4.A — Chainguard's *legacy-allowed*
> provider, on by default in FIPS images built from September 2026), while `hmac.new(key, data,
> hashlib.md5)` is refused. Python's own marker for the same idea is `hashlib.md5(data,
> usedforsecurity=False)`; use it, and code reviewers can see the intent. If your auditor wants **zero**
> MD5, ship strict — `ENV CHAINGUARD_LEGACY_ALLOWED=0` in the Dockerfile (Lab 5) — and every unmarked
> checksum call fails loudly; that refusal is how you find every call site that hasn't moved to SHA-2.

---

## Lab 5 — Your code on the FIPS image (~15 min)

The point of Lab 3 was "it's a `FROM` line." Prove it with real code. Write a small service that does
the crypto your product actually does — hash a token, verify an HMAC — and run it on both images:

```bash
cat > ~/labs/fips/svc.py <<'EOF'
import hashlib, hmac, ssl, _hashlib
print("OpenSSL:", ssl.OPENSSL_VERSION, "| FIPS mode:", "ACTIVE" if _hashlib.get_fips_mode() else "off")
secret = b"demo-secret"
token = hashlib.sha256(b"user:42").hexdigest()
tag = hmac.new(secret, token.encode(), hashlib.sha256).hexdigest()
print("token  :", token[:16], "...   (SHA-256)")
print("hmac   :", tag[:16], "...   (HMAC-SHA-256)")
print("verify :", hmac.compare_digest(tag, hmac.new(secret, token.encode(), hashlib.sha256).hexdigest()))
try:
    print("md5sum :", hashlib.md5(b"legacy-checksum").hexdigest()[:16], "...   (a checksum: non-security use)")
except ValueError:
    print("md5sum : REFUSED   -> strict mode: even checksums must move to SHA-2")
try:
    hmac.new(secret, token.encode(), hashlib.md5)
    print("md5 tag: computed  -> MD5 accepted as security: this image is NOT in FIPS mode")
except ValueError:
    print("md5 tag: REFUSED   -> FIPS mode active")
EOF

docker run --rm -v ~/labs/fips/svc.py:/svc.py --entrypoint python fips-demo:stock /svc.py
echo "----"
docker run --rm -v ~/labs/fips/svc.py:/svc.py --entrypoint python fips-demo:fips  /svc.py
```

```
OpenSSL: OpenSSL 3.x.x ... | FIPS mode: off
token  : ea3fd43be1e57d62 ...   (SHA-256)
hmac   : be68d9d39613d91c ...   (HMAC-SHA-256)
verify : True
md5sum : 861f9980b13ac8df ...   (a checksum: non-security use)
md5 tag: computed  -> MD5 accepted as security: this image is NOT in FIPS mode
----
OpenSSL: OpenSSL 3.x.x ... | FIPS mode: ACTIVE
token  : ea3fd43be1e57d62 ...   (SHA-256)
hmac   : be68d9d39613d91c ...   (HMAC-SHA-256)
verify : True
md5sum : 861f9980b13ac8df ...   (a checksum: non-security use)
md5 tag: REFUSED   -> FIPS mode active
```

Same code, same Python, same output for everything approved *and* for the checksum — and one line
that differs: the FIPS image refuses MD5 the moment it is used as security. **Your application didn't
change; its cryptographic posture did.**

Now ship it the way you'd ship for real — a Dockerfile whose base is the FIPS image:

```bash
cat > ~/labs/fips/Dockerfile <<'EOF'
FROM fips-demo:fips
COPY svc.py /app/svc.py
ENTRYPOINT ["python", "/app/svc.py"]
EOF
docker build -q -t svc:fips ~/labs/fips && docker run --rm svc:fips
```

> **Facilitator beat — shipping strict.** Add one line under `FROM` — `ENV CHAINGUARD_LEGACY_ALLOWED=0`
> — rebuild, and the `md5sum` line flips to `REFUSED -> strict mode`. That is the posture for an
> auditor who wants no MD5 anywhere: the image itself finds every checksum that still uses it.

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

> [!TIP]
> **On your desktop (optional)** (**FIPS lab desktop (labNN)** — or **Your Lab Server desktop
> (labNN)** on a single-lab day): **File Manager** → `fips-evidence` (the **FIPS lab** launcher
> opens `~/labs/fips`; `~/fips-evidence` is the folder you just created next to it). Double-click
> `posture.txt` and `refusal.txt` to read the two proofs; `sbom.spdx.json` opens in Firefox's JSON viewer
> (`file:///home/labuser/fips-evidence/sbom.spdx.json`).

You hand the reviewer: the active-module proof, the refusal proof, your own service's output on the
validated image, the image signature, and an SBOM — plus a pointer to Chainguard's FIPS 140-3
validation certificate for the module itself (Chainguard publishes the CMVP certificate number for its
FIPS images; cite it in the package).

---

## Your evidence log

| Check | `fips-demo:stock` | `fips-demo:fips` |
|---|---|---|
| FIPS-validated module, FIPS mode active (`get_fips_mode`)? | | |
| MD5 as security (HMAC-MD5) refused? | | |
| MD5 digest refused in strict mode (`CHAINGUARD_LEGACY_ALLOWED=0`)? | | |
| SHA-256 / HMAC-SHA-256 (approved) still work? | | |
| your `svc.py` runs unchanged? | | |
| signature verified + SBOM captured? | n/a | |

## Next steps

1. **Run Lab 2 against one of your real service images** — `FIPS mode: off`, HMAC-MD5 computed? Then it isn't in FIPS mode.
2. **Swap the base to the mapped `-fips` image** and re-run Labs 3–5 — module active, weak crypto
   refused, your code unchanged. Every refusal you hit is a real finding you'd otherwise have met at audit.
3. **Bring your questionnaire to a working session** — J2R maps your stack to the validated modules and
   helps assemble the evidence package.

Jonathan Spigler · J2R Solutions · jonathan@j2rsolutions.io
Phil Brooks · Partner Solutions Engineer, Chainguard
