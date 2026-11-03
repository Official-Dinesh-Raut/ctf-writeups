# Natas Level 1: Blocked right-click  

**Platform:** OverTheWire Natas  
**Vulnerability class:** Client-side restriction bypass / information disclosure  
**Goal:** Find the password for the next level, even though right-clicking is blocked.  

## Recon  

The page said that the password on the page but right-click was probibited. This suggested that the restriction was implemented in the browser (JS intercepting the context menu), not on the server. The question was rather the server was still sending the content.  

## Exploitation  

I bypassed the block by viewing the source code without using the right-click menu:

- `Ctrl+U` to open view-source, or `F12` for dev tools.
- Or request the page directly with curl:

        curl -u natas1:<PASSWORD_FROM_LEVEL_0> http://natas1.natas.labs.overthewire.org  

## Why it worked  

The right-click block only changes the browser's menu behaviour. The server still sends the full HTML to my machine, and anything delivered to the client can be read by them using Keyboard shortcuts, dev tools, and tui tools like curl which can skip the right-click handler entirely.  

## Detection (SOC view)  

There is nothing to detect in the logs. It is one ordinary authenticated GET request, the same as any other legitimate visitor would send. This shows the limit of log-based detection: a client-side "protection" produces no extra traffic when it is bypassed.  

Weak signals at most:  
- A `curl/x.x` User-Agent insted of a browser one
- A request with no follow-up requests for the page's CSS, JS, or images, because curl doesn't fetch them like.  

## Fix  

- Don't put secrets in HTML, comments, or JavaScript, since the client receives all of it.
- Don't rely on blocking right-click, text selection, or dev tools. THey are cosmetic and can't be enforced.
- Enforce access control on the server.  

## Takeaway  

Client-side restrictions are not security controls. If the browser receives it, the user can see it.  

## What I got wrong at first  

I checked the comments same as level 1 and got the password and I tried to copy the password but it didn't let me, the whole comment line element was being copied, so I switched to view-source.  
