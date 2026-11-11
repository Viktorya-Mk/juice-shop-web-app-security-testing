# Web Application Security Testing: OWASP Juice Shop

## Summary

I assessed OWASP Juice Shop v20.0.0, a deliberately vulnerable e-commerce application, against four attack paths: DOM-based XSS, IDOR, JWT "none" algorithm bypass, and privilege escalation at registration. Three succeeded and one failed, and all four are documented below.

| # | Test | Result | Impact |
|---|------|--------|--------|
| 1 | DOM-based XSS (search bar) | Exploited | Arbitrary JavaScript runs in the victim's browser |
| 2 | IDOR (basket API) | Exploited | Any logged-in user can read other users' baskets |
| 3 | JWT "none" algorithm bypass | Exploited | Forged admin token accepted without a signature |
| 4 | Privilege escalation via registration | Not exploitable | Server ignored the injected role |

## Lab environment

- **Target:** OWASP Juice Shop v20.0.0, run in Docker on a Kali Linux VM and reached over a host-only network
- **Attacker machine:** Kali Linux in VMware, isolated from any other network
- **Tools:** Burp Suite Community (proxy, Repeater), Firefox with Developer Tools, jwt.io, curl
- **Approach:** Gray-box (the application and its architecture were known in advance)
- **Scope and limits:** Manual testing of the four paths only. No automated scanning and no denial-of-service testing. Everything ran in an isolated lab; never try this on systems you don't own or have permission to test.

*The Juice Shop container running on port 3000.*\
<img height="90" alt="image" src="https://github.com/user-attachments/assets/edac6092-eaff-49b7-abf0-c84f2d598bdf" />


*OWASP juice shop running.*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/81465e6e-ad5b-454e-91ca-a99b7dc776df" />

## Findings

### 1. DOM-based XSS in the search bar

**What I did:** Entered an iframe payload into the search bar: `<iframe src="javascript:alert('XSS')"></iframe>`.

**Result:** The alert fired. In Burp, the search request returned a `304 Not Modified`, because the payload sits in the URL hash fragment (`#/search?q=...`), which the browser handles client-side and never sends to the server. This shows the flaw is in how the Angular front end renders input, not in a server-side reflection.

*The payload in the URL and the alert popup it triggers.*\
*1*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/3621a103-17fa-417e-92d6-d8d7dd1cf000" />
*2*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/2b622f73-0cd3-4b57-9db9-8c1f0bdcdbdd" />

*Burp history: the search request is a cached 304, confirming execution happens client-side.*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/66755df1-a670-4551-a69f-91ac0bd432fd" />



**Fix:** Never render user input as trusted HTML (avoid bypassing Angular's built-in sanitization), and add a Content Security Policy that blocks inline script and `javascript:` URIs.

### 2. IDOR on the basket API

**What I did:** Created two normal accounts (attacker and victim), added items to the victim's basket, then logged in as the attacker and intercepted `GET /rest/basket/7` (the attacker's own basket). In Burp Repeater I changed the ID to `6`.

**Result:** The server returned the victim's basket. It trusted the ID in the URL and never checked that the authenticated user owned that basket. The "Your Basket" restriction existed only in the front-end interface, so it gave no real protection.

*Repeater: the request modified from basket 7 to basket 6.*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/da18da30-7619-41b0-9ee4-0b951756b343" />

*The response returns basket 6 with its products, while logged in as the attacker.*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/ca941f9f-3bd1-4912-b583-e770ff333c5b" />


*Juice Shop confirms the "view another user's shopping basket" challenge as solved.*\
<img height="100" alt="image" src="https://github.com/user-attachments/assets/999f8a1b-b904-4c45-8856-8e932eabba67" />


**Fix:** Enforce object-level authorization on the server for every request (compare the resource owner with the authenticated user). Unpredictable IDs help as defense in depth but don't replace the check.

### 3. JWT "none" algorithm bypass

**What I did:** Logged in with curl to capture a valid JWT and decoded it. I then built a forged token with the header `{"alg":"none","typ":"JWT"}`, changed the role claim from `customer` to `admin`, and left the signature empty. I tested it against a protected endpoint.

**Result:** The server accepted the unsigned token. This is a critical authentication flaw: anyone can forge an identity without knowing any secret.

*Step 1: a valid token captured from the login response (lab test account).*\
<img height="100" alt="image" src="https://github.com/user-attachments/assets/42629828-22d6-4ae3-9083-453701ef359c" />
*Decoded token*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/37184f7f-a6e9-4a63-bac8-71ebe159b4e5" />

*Step 2: the forged token. The header uses `alg: none`, the role is `admin`, and the signature is empty.*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/3dcfbfb2-3c73-4fd2-b125-2423c5123c18" />

*Step 3: replacing token in browser.*\
<img height="167" alt="image" src="https://github.com/user-attachments/assets/efe77c34-b8af-49a6-97ae-f0101c084582" />

**Fix:** Pin the accepted signing algorithms when verifying tokens and reject `none`, keep the JWT library up to date, and don't trust a role claim without checking it against server-side data.

### 4. Privilege escalation at registration (not exploitable)

**What I did:** Intercepted the `POST /api/Users` registration request in Burp and added privileged fields to the JSON body, including `"role":"admin"` and variations such as `"role":"administrator"`, `"type":"admin"` and `"isAdmin":true`.

**Result:** The server ignored the injected fields and assigned the default `customer` role, so this endpoint correctly protects against mass assignment. I'm including it because documenting a failed test is part of an honest assessment.

*The intercepted registration request with `"role":"admin"` added to the body.*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/9b72e056-eaa1-4a21-b640-fd941a46836c" />



*The token issued for the new account, decoded: the role is still `customer`.*\
<img height="250" alt="image" src="https://github.com/user-attachments/assets/15b63e63-4864-4f58-a5ca-f16d38553ee3" />


## Takeaways

- **Front-end restrictions are not security controls.** The IDOR and the JWT issue both come from trusting data the client controls. Authorization has to be enforced on the server, on every request.
- **Small authorization mistakes carry large risk.** Two of the three successful attacks needed nothing beyond a proxy and a text editor, and one of them (the forged token) gave a path to impersonating any user.
- **Authentication tokens need strict verification.** Accepting `alg: none` shows why the signing algorithm must be pinned on the server and the role claim never trusted on its own.
- **Negative results are worth reporting.** The registration endpoint correctly rejected the injected role. Documenting what is secure as well as what is broken makes an assessment more credible and shows where a development team is already doing things right.
