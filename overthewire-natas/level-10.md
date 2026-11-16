# Natas Level 10: Bypassing a Command Injection Blocklist Filter

**Platform:** OverTheWire Natas  
**Vulnerability class:** OS Command Injection (OWASP A03, Injection) — incomplete blocklist filtering  
**Goal:** Get the server to execute an arbitrary command despite a character filter, to read the next level's password file.  

## Recon

This level is the same dictionary search feature as level 9, but viewing
the source showed one addition:

```php
if(preg_match('/[;|&]/', $key)) {
    print "Input contains an illegal character!";
} else {
    passthru("grep -i $key dictionary.txt");
}
```

A blocklist filter now rejects any input containing `;`, `|`, or `&`,
the exact character I used to solve level 9. Submitting the same
semicolon payload here returned "Input contains an illegal character!"
instead of running the command, so I needed a different way to separate
two shell commands that wasn't on this specific list.

## Exploitation

A newline character ends one shell command and starts another, the same
way `;` does, and it isn't included in the filter's pattern. A plain
single-line text input won't let you type a literal newline (pressing
Enter submits the form instead), so I sent the request with curl,
using bash's `$'...'` syntax to produce a real newline byte, which curl
then URL-encodes before sending:

```
curl -u natas10:<PASSWORD_FROM_NATAS9> \
  --data-urlencode $'needle=\ncat /etc/natas_webpass/natas11 #' \
  -G "http://natas10.natas.labs.overthewire.org/"
```

This built the full command as two lines:
```
grep -i 
cat /etc/natas_webpass/natas11 #dictionary.txt
```
The newline split it into two independent commands. The trailing `#`
comments out the rest of that second line (the ` dictionary.txt` that
the PHP script always appends after `$key`), so it's ignored instead of
causing an argument error. `cat` ran on its own and its output was
returned directly in the response.

I first tried command substitution (`` `cat ...` `` or `$(cat ...)`),
which also isn't on the blocklist. It technically passed the filter but
produced no visible output: substitution inserts its result as an
**argument to `grep`**, not as a standalone command, so the password
ended up being used as `grep`'s search term, which isn't found in the
dictionary and prints nothing. Only an actual command separator (`;` in
level 9, a newline here) runs the injected command as its own statement
with output sent directly back.

## Why it worked

The filter blocks three specific characters, but bash has other ways to
separate commands that the developer didn't account for. A blocklist
only stops what it explicitly lists; it does nothing against any
equivalent technique outside that list. The underlying flaw, user input
reaching a shell command unsanitized, was never actually fixed between
level 9 and level 10, only one way of exploiting it was blocked.

## Detection (SOC view)

In the access log this would show up as a request with a URL-encoded
newline in the parameter:

```
GET /?needle=%0acat+/etc/natas_webpass/natas11+%23
```

Things I'd alert on:
- A parameter value containing a raw or URL-encoded newline (`%0a`),
  backtick, or `$(` sequence, not just `;`, `|`, `&`
- A request to an endpoint that previously returned a filter-rejection
  message (like "Input contains an illegal character!"), followed
  shortly after by a similar request using different syntax that
  succeeded; that pattern on its own suggests an attacker iterating
  around a filter
- Requests where the parameter value references known sensitive file
  paths

## Fix

- Replace the blocklist with an allowlist: only permit a known-safe set
  of characters (e.g. letters and digits for a search term), rejecting
  everything else by default rather than trying to enumerate every
  dangerous character.
- Better still, don't shell out at all. Use a language-level search
  function instead of invoking `grep` as an external process.
- If shelling out is unavoidable, pass arguments through an API that
  takes them as a list/array rather than a concatenated string, so the
  shell never re-parses user input as syntax regardless of which
  characters it contains.

## Takeaway

A blocklist is a weak defense against command injection because it
treats the symptom (specific characters) rather than the cause (user
input reaching a shell at all). This level is a direct illustration:
the same underlying bug as level 9 survived a patch that only addressed
the exact exploit already demonstrated, not the class of vulnerability.

## What I got wrong at first

I assumed any technique that bypassed the character filter would also
produce visible output, since that's what happened with `;` on level 9.
Trying command substitution first, which did bypass the filter but
returned nothing, taught me that bypassing a filter and achieving the
intended result (standalone command execution with output returned) are
two different things that need to be verified separately.
