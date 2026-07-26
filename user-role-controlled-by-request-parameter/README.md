# User Role Controlled by Request Parameter — Broken Access Control

**Platform:** PortSwigger Web Security Academy

**Category:** Access Control (OWASP Top 10 2025 — A01: Broken Access Control)

**Difficulty:** Apprentice


## Objective
The application has an admin panel at `/admin` that identifies administrators using a forgeable cookie. The goal is to access the admin panel and use it to delete the user `carlos`.

## Methodology
1. Attempted to access `/admin` directly while logged in as a low-privileged user (`wiener`) and received: *"Admin interface only available if logged in as an administrator."*
2. Inspected cookies via Burp Suite and the browser's DevTools (Application → Cookies) and found a client-side cookie named `Admin`, set to `false`.

![Admin cookie visible in browser DevTools, set to false](screenshots/01-admin-cookie-devtools.png)

3. Modified the cookie value from `Admin=false` to `Admin=true` in an intercepted request and forwarded it — this granted access to the admin panel and its user management functionality.

## Exploit
Gaining access to the panel was not sufficient on its own: the delete action sends its own separate request, which still carried the original `Admin=false` cookie value, causing the first delete attempt to fail silently.

![Failed delete request still carrying Admin=false](screenshots/02-failed-delete-request.png)

Each outgoing request needed the cookie value corrected individually before being forwarded, until the delete request for `carlos` was sent with `Admin=true`.

*Note: a more efficient approach — instead of patching each intercepted request in Burp — would have been to edit the `Admin` cookie's value directly in the browser's DevTools (Application → Cookies) from `false` to `true`. The browser then persists that value and automatically resends `Admin=true` on every subsequent request, removing the need for repeated interception.*

## Proof of Concept
After sending the delete request with the corrected cookie value, the application confirmed the user `carlos` was deleted and the lab was marked as solved.

![Lab solved after deleting the user carlos](screenshots/03-solved.png)

## Root Cause
The application determined administrator privileges based on a client-side cookie value that was never validated against any server-side session or user record. Since the cookie was fully attacker-controlled and unsigned, any user could grant themselves administrative access simply by changing its value — the server trusted client input for a security-critical authorization decision.

## Remediation
- Never determine privilege level from a value the client can directly set or modify (cookie, hidden form field, request parameter).
- Store the user's role server-side, tied to their authenticated session, and re-check it on every privileged request.
- If a role-related value must be sent to the client at all, it should be cryptographically signed and validated server-side so tampering is detectable — though the safer default is to keep authorization decisions entirely server-side.
