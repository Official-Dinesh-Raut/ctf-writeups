# Natas Level 6:Secret stored in an exposed include file

**Platform:** OverTheWire Natas  
**Vulnerability:** Information disclosure (sensetive file in the web root, exposed source code) 
**Goal:** Find the secret the form asks for, then use it to get the next level's password.  

## Recon

The page showed a form asking for a secret, plus a link to view the server-side source code. Reading the PHP showed that the script loads another file with `include`, then compares my input against a variable defined in that file. If they match, it prints the next password.

So the secret was not in the page itself. It was in a seperate file that the code pulled in. That raised one question: can I request that file directly?

## Exploitation

The include file lived in a folder under the web root, so I requested its path myself:

```
curl -u natas6:<PASSWORD_FROM_lEVEL_5> http://natas6.natas.labs.overthewire.org/<include_path>
```

The file's extension was not one the server executes as PHP, so the server returned its contents as plain text. That showed the secret assigned to a variable. I typed that value into the form on the main page and submitted it, and the page printed the credentials for the next level.

## Why it worked

The developer moved the secret into a seperate file, which hides it from the main page but not from the web. The file sat in a web-accessible directory, and its extension meant the server send the raw contents instead of running the code. The source code view also told me exactly where to look. This is the same idea as level 2 and 3; a file nobody links to is not a file nobody can reach.

## Detection (SOC view)

In the access log, the pattern would be:

- `GET /` (normal page load)
- `GET /<include-path>` ( a direct request for an include file that no page links to)
- `POST /` (the form submission with the secret)

I would alert on direct requests to include, config, and backup file extensions ((`.inc`, `.bak`, `.conf`, `.old`) from any client, and on requests to paths that are referenced only in server-side code, since a normal visitor never needs to request them. A request for such a file followed shortly by a successful form submission suggests the client read the file first.

## Fix

- Keep include and config files outside the web root so no URL maps to them.
- Use the `.php` extensions the server executes them instead of displaying them, or deny direct access to those paths in the server config.
- Never store secrets in code or in files under the web root. Use environment variables or a secrets manager.
- Don't expose the application's source code to users.

## Takeway

Hiding a secret in a different file is not protection if the server will still hand that file out. Check every file for script references and ask whether it can be requested directly.

## What I got wrong at first

I expected a SQL injection because of the form, but the levell was really about reading the source and following the include.)

