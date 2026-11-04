# Natas Level 7: Path Traversal / Local File Inclusion (LFI)

**Platform:** OverTheWire Natas  
**Vulnerability class:** Path Traversal / Local File Inclusion (LFI) — information disclosure (OWASP A01, Broken Access Control)  
**Goal:** Find the password for natas8, hidden at a known file path on the server.

## Recon

The page had two links, Home and About, both pointing to `index.php` with a `page` parameter in the URL, something like `index.php?page=home`. That told me the same script was loading different content depending on the value of that parameter.

The page source also contained a comment stating that the password for the webuser `natas8` was stored at `/etc/natas_webpass/natas8`. Natas stores each level's password in a file like this on the server, outside the web root, so normally it isn't reachable through a URL. The question was whether the `page` parameter could be pointed at that file instead of one of the site's own pages.

## Exploitation

Since `page` controlled which file got loaded, I replaced its value with the absolute path from the hint:

```
http://natas7.natas.labs.overthewire.org/index.php?page=/etc/natas_webpass/natas8
```

This worked immediately, with no `../` needed. The page displayed the contents of that file in place of the normal Home or About content.

## Why it worked

The `page` parameter was used directly to decide which file the server should read and display, with no check on what value I could supply. The script trusted the parameter instead of restricting it to a small, known set of pages. Because an absolute path worked without any `../`, the include wasn't even limited to a subdirectory relative to the site: it would load any file the web server process had permission to read, anywhere on the filesystem.

## Detection (SOC view)

In the access log this would show up as a request like:

```
GET /index.php?page=/etc/natas_webpass/natas8
```

Things I'd alert on:
- A `page` (or similarly named) parameter whose value is an absolute path (starting with `/`) or contains `../`
- Requests where the parameter value matches known sensitive file locations (`/etc/passwd`, `/etc/shadow`, application config paths, credential files)
- A client requesting the normal pages (`index.php?page=home`, `index.php?page=about`) and then an unusual path in the same parameter shortly after, which suggests probing rather than normal browsing
- Unusually large or unexpected response sizes for `index.php`, since a system file is a different size than the site's own pages

## Fix

- Never build a file path directly from user input.
- Use an allowlist: map a short set of valid page names (`home`, `about`) to fixed filenames in code, and reject anything not on the list.
- If a path must be built dynamically, resolve it to an absolute path and confirm it still starts with the intended base directory before reading it.
- Run the web server process with the minimum file permissions needed, so even a successful traversal can't reach unrelated system files.
- Store secrets outside any path the application can be tricked into reading, and restrict file permissions on them.

## Takeaway

Any parameter that looks like it names a file or a page is worth testing for path traversal. The PortSwigger lab I did first used the same idea on a `filename` parameter for images; this level showed the identical flaw used to reach a real secret instead of a test file, and with and even weaker implementation since no traversal sequence was needed at all.

## What I got wrong at first

I expected to need `../` sequences to climb out of the site's directory, like in the PortSwigger lab, but the absolute path worked on its own. That told me this include had no restriction at all, not even a same-directory one.


