# Natas level 4: Trusting referer header

**Platform:** OverTheWire Natas  
**Vulnerability class:** Broken access control(trusting a client-controlled header)  
**Goal:** Find the password for the next level on the page that disallowed access based on the referer header.

## Recon
The page said access was disallowed because I was visiting from a different address than the one it expected. That suggested the server was checking where my request claimed to come from. Browsers send this in the `Referer` header, which names the page you followed a link from.

I confirmed what my client sends by adding `-v` to curl and reading the
request headers.

## Exploitation

To alter the headers we need a proxy(Burpsuite is very suitable for this). Since I didn't want the hassle to setup Burpsuite right now I used curl to change the header using the command:

```
curl -u natas4:<PASSWORD_FROM_LEVEL_2>  http://natas4.natas.labs.overthewire.org -H "Referer: http:/natas5.natas.labs.overthewire.org"
```

This pulled all the elements of the correct page where the flag was hidden.

# Why it worked

The server made an access decision based on a header that the client can fully control. Anyone can set the `Referer` to any value, so the check proves nothing about where the request really came from.

# Detection (SOC view)

- A Referer pointing to a page that does not link to this resource, such as a different level's hostname
- A request with no earlier page load in the same session, meaning the client never actually visited tha page they calimed to have come from
- A User-Agent like `curl/x.x` paired with a browser-style Referer
- The same client sending the same URL with and without a Referer, or with changing Referer values,,which suggests probing

## Fix

- Never use `Referer`, `User-Agent`, or other request headers to decide
  who is allowed in. They can all be forged.
- Enforce access control on the server with real authentication and
  session checks.
- If the Referer is used at all (for example to reduce cross-site request
  forgery), treat it as a weak extra signal, never as the only check.

## Takeaway

Anything the client sends, including headers, cookies and form fields, can be edited. A control built on client-supplied data is not a control.

## What I got wrong at first

I tried clicking the refresh page link in the browser and it kept failing, because the browser was sending the real Referer.

