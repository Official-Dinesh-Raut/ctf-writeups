# Natas Level 2: Exposed directory listing

**Platform:** OverTheWire Natas  
**Vulnerability class:** Directory listing enabled / information disclosure  
**Goal:** Find the password for the next level on a page that claims to have nothing on it.

## Recon

The page said there was nothing on it. The rendered page was blank, so I
looked at the source (Ctrl+U) and found an `<img>` tag pointing to a file
inside a `files/` directory. That told me the site had a directory I
couldn't see from the page itself.

## Exploitation

I requested the parent directory directly instead of just the image:

```
curl -u natas2:<PASSWORD_FROM_LEVEL_1> http://natas2.natas.labs.overthewire.org/files/
```

The server returned an index of every file in that directory. Besides the
image, there was a text file with user credentials. I requested that file
the same way and found the entry for the next level.

## Why it worked

Directory listing was enabled, so the server showed the contents of a
folder when no index page existed. Sensitive data was also stored inside a
web-accessible path. The page never linked to the text file, but being
unlinked is not the same as being protected. Anyone who guesses or finds
the folder can browse it.

## Detection (SOC view)

In the access log, this would show up as a short sequence of GET requests:

- `GET /files/pixel.png` (normal page load)
- `GET /files/` (a directory request, not a file)
- `GET /files/users.txt` (a file the page never references)

The signal is a client requesting a directory path and then a file that no
page links to. One occurrence could be a curious user. Many requests to
different directory paths from one IP in a short time suggests scanning or
enumeration, and I would alert on that pattern. Repeated 403 and 404
responses in the same window would strengthen it.

## Fix

- Disable directory indexing (`Options -Indexes` in Apache, or
  `autoindex off` in nginx).
- Never store credentials or other sensitive data in a web-accessible
  folder.
- Keep sensitive files outside the web root and protect them with
  proper authentication.

## Takeaway

When the source references a file, check its parent directory. Hidden
does not mean protected.

## What I got wrong at first

I only tried to open the image and didn't think to request the folder until I read the path carefully.
