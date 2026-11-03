# Natas Level 3: Hidden directory in robots.txt

**Platform:** OverTheWire Natas  
**Vulnerability class:** Information disclosure via robots.txt  
**Goal:** Find the password for the next level on a page that claims no information is leaked.  

## Recon

The page said there was nothing on it, and the source contained a comment
saying not even Google would find it this time. That hinted that the
developer was hiding something from search engines, which made me think of
the file that tells crawlers what to skip: `/robots.txt`.

## Exploitation

I requested the file from the root of the site:

```
curl -u natas3:<PASSWORD_FROM_LEVEL_2> http://natas3.natas.labs.overthewire.org/robots.txt
```

It contained a `Disallow:` line naming a directory. I browsed to that
directory directly and found a directory listing with a text file holding
credentials, the same kind of file as in level 2.

## Why it worked

`robots.txt` is a polite request to well-behaved crawlers, not an access
control. It is also public, so listing a path in it advertises that the
path exists. The directory itself had no authentication, so anyone who
read the file could walk straight in. Directory listing was also enabled,
as in level 2.

## Detection (SOC view)

In the access log, the pattern would be:

- `GET /robots.txt`
- `GET /<disallowed-directory>/`
- `GET /<disallowed-directory>/users.txt`

Real search engine crawlers obey `robots.txt`, so a client that reads it
and then visits the disallowed paths is not behaving like a crawler. If
the User-Agent is a browser or `curl/x.x`, that is worth a closer look.
I would alert on requests to paths listed as `Disallow` in `robots.txt`
from clients that are not verified crawlers. Many such requests from one
IP suggests deliberate enumeration.

## Fix

- Protect sensitive directories with real authentication.
- Don't list secret paths in `robots.txt`. Use `noindex` headers or
  authentication to keep content out of search results instead.
- Disable directory listing (`Options -Indexes` in Apache).
- Keep sensitive files outside the web root.

## Takeaway

`robots.txt` is a signpost for attackers as much as for crawlers. Hiding
something is not the same as protecting it.

## What I got wrong at first

I looked for hidden files in
the page source first before realizing the hint was about search engines.
