# Natas Level 5: Trusting a login flag in a cookie

**Platform:** OverTheWire Natas  
**Vulnerability class:** Broken authentication / insecure client-side state (OWASP A01, A07)  
**Goal:** Get the server to treat me as logged in, even though I never authenticated.

# Recon

The page said I was not logged in. The first request's response headers showed the server setting a cookie:

```
set-Cookie: loggedin=0
```

At first the browser's Storage tab showd no cookie, so I confirmed with `curl -v` that the server really was sending it. The name `loggedin` and the value `0` suggested a simple yes/no flag the server reads on each request.

## Exploitation

Cookies are stored by the client and send back at each request, so I can change the value before it is send. I changed the flag to the opposite value and reloaded the page.

With curl, the same thing looks like this:

```
curl -u natas5:<PASSWORD_FROM_LEVEL-4> -b "loggedin=<NEW_VALUE>" http://natas5.natas.labs.overthewire.org/
```

The page switched from "not logged in" to showing the credentials for the next level.

## Why it worked

The server stored the login decision in a cookie that the user controls, and then trusted it on every request. It never checked that a real authentication had happened. Anyone who knows or guessed the cookie's name and meaning can grant themselves access.  
## Detection (SOC view)

The cookie travels in the `Cookie` request header , which standard access logs usually do not record , so this may need extra logging to see. If cookies are logged (or if application logs record session state), I would look for:

- Request carrying a priviledged flag (`loggedin=1`) with no earlier login request from the same client
- The same client sending the same URL with different cookie values in a short time, which suggests tampering or probing
- A `curl/x.x` User-Agent on an authenticated page
- Authenticated access with no matching authentication event in the appplication's own logs

The general rule is to alert on authenticate dactivity that has no authentication event before it

## Fix

- Keep login state on the server. Give the client only a random, unguessable session ID and look up the session server-side.
- If data must live in the cookie, sign it ( for example with an HMAC) so tampering is detected.
- Set cookie flags: `HttpOnly`, `Secure` and `SameSite`.
- Never use a simple readable flag as proof of identity.

## Takeaway

A cookie is user-controlled input like any other. Storing "am I logged in" on the client lets the user answer the question for themselves.

## What I got wrong at first

The browser's Storage tab showed no cookie, so I checked the response headers instead and found the `Set-Cookie` header. Reading the headers directly was more reliable than trusting the browser's storage view.
