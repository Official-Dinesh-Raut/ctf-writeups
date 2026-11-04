# Natas Level 8: Reversible encoding used as a secret

**Plarform:** OverTheWire Natas  
**Vulnerability class:** Insecure cryptographic storage/ "security through obscurity" -relying on encoding rather than encryption.  
**Goal:** Recover the plaintext secret from a hardcoded value in the source, then submit it to get the password for natas9.

## Recon

The page showed a form asking for a secret, with a link to view the
source code. The PHP defined a hardcoded hex string and a function,
`encodeSecret()`, that ran an input through three chained operations:

```php
function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}
```

In order, this does:
1. `base64_encode($secret)` — base64-encode the input
2. `strrev(...)` — reverse the resulting string
3. `bin2hex(...)` — convert that reversed string to hex

On form submission, the script runs my input through `encodeSecret()`
and compares the result to the hardcoded hex string. If they match, it
reveals the password.

Since all three operations are plain encodings, not encryption, each
one has an exact inverse and none of them use a secret key. That meant
I didn't need to guess the input: I could start from the hardcoded hex
value and undo each step, in reverse order, to recover the original
secret directly.

## Exploitation

I reversed the transformation by undoing the three steps in the
opposite order from how they were applied:

1. **Hex-decode** the hardcoded string (undoes `bin2hex`)
2. **Reverse** the result (undoes `strrev` — reversing twice restores
   the original order)
3. **Base64-decode** that (undoes `base64_encode`)

I did this in CyberChef with a recipe of "From Hex" → "Reverse" →
"From Base64", pasting the hardcoded value as input. The output was a
short, readable string, the original secret.

I submitted that value in the form's `secret` field, and the page
printed the password for natas9.

## Why it worked

The developer used encoding functions (base64, string reversal, hex
conversion) as if they provided secrecy. All three are fully reversible
with no key involved, so anyone who can read the source code (which the
level explicitly provides) can run the same transformation backward and
recover the original value with certainty, not by guessing.

This is a common real-world mistake: encoding is not encryption. Base64
in particular is often mistaken for a protective measure when it's
really just a way to represent binary data as text.

## Detection (SOC view)

This vulnerability lives in the source code and the comparison logic,
not in network traffic, so there isn't a clean log signature for the
exploitation itself: a correct submission looks like any other POST
request with a `secret` field.

What this really calls for is detection at a different stage:
- **Source code review / static analysis** flagging reversible encoding
  functions (`base64_encode`, `strrev`, `bin2hex`, etc.) used anywhere
  near words like "secret", "password", or "token" in the codebase
- **Secret scanning tools** (like gitleaks or truffleHog) extended to
  flag not just plaintext secrets but hardcoded encoded blobs compared
  against user input, since that pattern indicates "security through
  obscurity"
- If logging is available at the application layer, repeated failed
  submissions to the same secret-checking endpoint from one client
  could indicate brute-forcing, though in this case the attack doesn't
  require brute force at all

## Fix

- Never use encoding (base64, hex, reversal) as a substitute for real
  cryptographic protection. If a secret must be verified, store a
  salted hash of it and compare hashes, not the reversible encoding of
  the plaintext.
- If the actual secret value is needed later (not just verified), use
  proper encryption with a securely stored key, not a hardcoded
  reversible transformation.
- Don't expose application source code to end users in production. This
  level deliberately shows it to teach the lesson, but in a real app,
  revealing the comparison logic hands an attacker everything they need.

## Takeaway

Encoding and encryption are not the same thing. Encoding is reversible
by anyone who knows the steps and needs no secret key, so it provides
no real confidentiality. If I can read the code that encodes a secret,
I can run it backward and recover the secret exactly.

## What I got wrong at first

I initially thought about guessing a plaintext value and running it
forward through `encodeSecret()` to check for a match, which would have
meant blind trial and error. Realizing each function was individually
reversible meant I could instead start from the known output and work
backward deterministically, with no guessing needed.
