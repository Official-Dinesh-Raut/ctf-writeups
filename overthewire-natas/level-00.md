# Natas Level 0: Password in the HTML source

**Platform:** OverTheWire Natas  
**Vulnerability class:** Information disclosure (sensitive data in client-side code)  
**Goal:** Find the password for the next level on the page.

## Recon

The page said the password was on the page, but nothing was visible in the
browser. Since browsers only render part of what the server sends, I
requested the page with curl to see the raw response:

    curl -v -u natas0:natas0 http://natas0.natas.labs.overthewire.org/

The verbose output showed:
- The request used HTTP Basic authentication (an `Authorization: Basic ...` header)
- The server replied `200 OK` and identified itself as Apache on Ubuntu
- The response body was a small HTML page

## Exploitation

The HTML body contained a comment (`<!-- ... -->`) holding the next
level's credentials. A single authenticated GET request was enough.
No tools beyond curl were needed. The browser's view-source (Ctrl+U)
would show the same thing.

## Why it worked

Comments are part of the response body. They are hidden from the rendered
page, but "not displayed" does not mean "not sent". Anyone who requests the
page receives the secret.

## Detection (SOC view)

This is one normal GET request with valid credentials, so it looks
legitimate in the access log. That is the lesson: nothing flags the
leak, so prevention has to happen before deployment.

Things I would still note:
- Basic auth only base64-encodes the username and password, it does not
  encrypt them. Over plain HTTP, anyone on the network path can read them.
- A User-Agent like `curl/8.x` on a page meant for browsers can be a weak
  signal of scripted access.

## Fix

- Never put secrets or credentials in HTML, comments, or any client-delivered code.
- Add code review and automated secret scanning (e.g. gitleaks) to the
  deployment process.
- Serve authenticated pages over HTTPS.

## Takeaway

Always check what the server sends, not just what the browser shows.
