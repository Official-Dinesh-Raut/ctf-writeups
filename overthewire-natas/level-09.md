# Natas Level 9: OS Command Injection via Unsanitized `grep` Call

**Platform:** OverTheWire Natas  
**Vulnerability class:** OS Command Injection (OWASP A03, Injection)  
**Goal:** Get the server to execute an arbitrary command, to read the next level's password file, by injecting into a dictionary search feature.  

## Recon

The page is a "search a dictionary" feature: a text box where you type a
word, and the page returns matching lines from a `dictionary.txt` file.
Viewing the source showed the backend logic:

```php
if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    passthru("grep -i $key dictionary.txt");
}
```

My search input (`needle`) is inserted directly into a shell command
string with no quoting or escaping, then run with `passthru()`, which
sends the raw command output straight back in the response. There's no
filtering of any kind on this input.

## Exploitation

Since my input reaches a shell with no restriction, I could inject a
semicolon to end the intended command and start a second one of my own:

```
needle = ;cat /etc/natas_webpass/natas10
```

This built the full command:
```
grep -i ;cat /etc/natas_webpass/natas10 dictionary.txt
```
Bash treats `;` as a command separator, so it ran `grep -i` (empty,
harmless, since no search term was given) and then
`cat /etc/natas_webpass/natas10` as a second, fully separate command. Its
output was sent back directly in the response.

## Why it worked

The script builds a shell command by directly concatenating unsanitized
user input into a string, then executes it with `passthru()`. Nothing
checks what the input contains, so any shell metacharacter in it is
interpreted by the shell exactly as if I'd typed it into a terminal
myself. A semicolon is one of the simplest ways to chain a second,
independent command onto the first.

## Detection (SOC view)

In the access log this would show up as a request like:

```
GET /?needle=;cat+/etc/natas_webpass/natas10
```

Things I'd alert on:
- A `needle` (or similarly named) parameter containing shell
  metacharacters (`;`, `|`, `&`, `` ` ``, `$(`) rather than a plain
  search word
- Requests where the parameter value references known sensitive file
  paths (`/etc/`, `/etc/passwd`, credential file locations)
- Any endpoint known or suspected to shell out to an external process
  (like a "search" feature backed by `grep`) receiving input with
  command syntax rather than plain text

## Fix

- Never build a shell command from unsanitized user input.
- Use a language-level search function (e.g. PHP's own string/array
  search functions, or reading the file and matching lines in code)
  instead of invoking an external `grep` process at all.
- If shelling out is unavoidable, use a parameterized/array argument
  API so the shell never re-parses user input as syntax, and apply an
  allowlist of permitted characters as a secondary layer.

## Takeaway

Passing raw user input into a shell command, with no filtering
whatsoever, gives an attacker full command execution, not just the
ability to manipulate the intended command's behavior. Any character
meaningful to the shell becomes a tool for the attacker.

## What I got wrong at first

I initially tried submitting
through the page's search box and wasn't sure whether the semicolon
would be encoded correctly, so I checked the resulting URL after
submission to confirm it matched what I expected.
