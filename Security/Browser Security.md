#security #browser #cors #csrf #xss #cookie
# CORS, CSRF, XSS and SameSite

## Core distinctions

| Concept      | Meaning                                                                                                                    | Security question                                             |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **CORS**     | Cross-Origin Resource Sharing: a browser-enforced policy configured through server response headers.                       | May JavaScript on this origin read another origin's response? |
| **CSRF**     | Cross-Site Request Forgery: an attacker induces an unwanted request using the victim's automatically attached credentials. | Is this authenticated state-changing request legitimate?      |
| **XSS**      | Cross-Site Scripting: attacker-controlled content executes as code inside the application.                                 | Can untrusted data become executable content?                 |
| **SameSite** | A cookie attribute restricting when cookies accompany cross-site requests.                                                 | Should this request carry this cookie?                        |

Authentication identifies the user; authorization checks permission. Neither CORS nor a CSRF token replaces those checks.

## CORS does not generally prevent CSRF

The **same-origin policy** normally prevents JavaScript from reading responses from another origin. CORS lets the destination server permit that access, for example:

```http
Access-Control-Allow-Origin: https://app.example.com
```

Some cross-origin requests require an `OPTIONS` **preflight** before the browser sends the actual request. Others, including ordinary HTML form submissions, can reach the server without preflight or permission to read the response.

A CSRF attacker does not need to read the response: causing the action is enough. Therefore, a CORS error does not mean the server received no request or performed no action. CORS also does not restrict clients such as `curl`.

## CSRF: abusing automatically attached credentials

A typical attack involves a user logged into a bank visiting an attacker-controlled page. That page submits a form to the bank's transfer endpoint. If cookie policy permits the session cookie to accompany the request, and the server checks only authentication, the transfer may succeed.

The browser attaches eligible cookies automatically; it does not know whether the user intended the action. **Changing an endpoint from `GET` to `POST` alone is not CSRF protection.** Ordinary forms can submit `POST` requests too.

## CSRF tokens: the additional secret

A CSRF token is an unpredictable value that the legitimate application must **explicitly submit** with a protected request. The login cookie alone is insufficient.

### Synchronizer-token pattern

1. The server generates a cryptographically secure random token and associates it with the user's session.
2. The application receives it in HTML or a response payload, such as JSON.
3. The application submits it in a form field or request header.
4. The server verifies it against the current session **before any side effect**, rejecting missing or invalid tokens.

A form can include:

```html
<input type="hidden" name="csrf_token" value="SERVER_GENERATED_TOKEN">
```

A JavaScript request can instead include:

```http
X-CSRF-Token: SERVER_GENERATED_TOKEN
```

These are alternative transport examples; the server generates and validates the actual value.

**Why it works:** another origin normally cannot read the application's token. Knowing the field name is not enough, and a token from the attacker's own session must not validate against the victim's session.

A hidden input is not inherently secret: scripts on the page can read it. Protection comes from unpredictability, session binding, and cross-origin read restrictions—not from hiding the field visually. A valid token is not proof of a human click or permission to perform the action.

### Lifetime and handling

- A **session-scoped token** is common; it need not change after every request. Generate a new token for a new session, especially at login.
- Per-request tokens can reduce replay opportunities but complicate concurrent requests, multiple tabs, and back-button navigation.
- Keep tokens out of URLs and logs: URLs can leak through history, logs, and referrers.
- For the synchronizer pattern, do not rely on a token supplied only as a cookie: cookies are attached automatically.
- Prefer the framework's built-in protection over a custom implementation.

### Double-submit cookies

This alternative stores a CSRF token in a cookie and requires the application to copy it into a separate request field or header. A naive implementation only checks that the two values match.

**Naive matching is vulnerable if an attacker can plant or overwrite a cookie**, for example through a compromised sibling subdomain. Prefer a framework's **signed, session-bound double-submit implementation**. Signing without session binding is not sufficient.

If JavaScript must read the CSRF-token cookie, it cannot be `HttpOnly`; the separate **session cookie should still be `HttpOnly`**.

## XSS: why CSRF tokens can be defeated

XSS occurs when untrusted data is inserted into a page in a way that lets the browser execute attacker-controlled code. For an ordinary comment-display element:

```js
// Unsafe for untrusted comments: interprets the value as HTML.
commentElement.innerHTML = userComment;

// Safe for plain-text comments: creates text, not HTML markup.
commentElement.textContent = userComment;
```

Malicious HTML can execute code through constructs such as JavaScript event handlers. `textContent` here means an ordinary text-display element, not a `script` or `style` element.

Common forms of XSS:

- **Stored:** malicious content is saved, such as in a comment, then rendered to visitors.
- **Reflected:** request input, such as a search query, is inserted unsafely into the response.
- **DOM-based:** client-side code passes attacker-controlled data to an unsafe browser API.

These categories can overlap.

### The broken CSRF assumption

A separate attacker origin normally cannot read the token. **An XSS script executes under the application's own origin**, so it can often read the token from the page or fetch it through the same APIs the application uses. It can then submit a valid-token request, with the browser attaching the victim's session cookie.

The attacker need not steal the token or cookie and send it elsewhere; they can act directly from the compromised page.

| Defense | Why it does not reliably stop XSS-driven actions |
|---|---|
| **CSRF token** | Injected code can often obtain and submit the valid token. |
| **`HttpOnly` session cookie** | Blocks JavaScript from reading the cookie, but the browser still attaches it to eligible JavaScript-initiated requests. |
| **`SameSite`** | Requests made inside the compromised application are ordinarily same-site. |
| **CORS / origin checks** | Injected code acts under the trusted application's origin. |

### Preventing XSS

Use framework auto-escaping and safe text APIs. Apply context-appropriate encoding where necessary: HTML text, attributes, URLs, and JavaScript have different rules. Avoid raw-HTML escape hatches and code-evaluating APIs such as `eval`. If users must author rich HTML, use a maintained sanitizer such as DOMPurify. A strict **Content Security Policy (CSP)** is an additional layer, not a substitute for safe rendering.

## Does SameSite eliminate the need for CSRF tokens?

**Not universally. For cookie-authenticated applications, explicit SameSite settings plus framework CSRF protection are the safer default.**

The table describes SameSite eligibility; cookie scope, fetch credentials settings, and other browser policies still apply.

| Setting | Cookie eligibility | Main limitation |
|---|---|---|
| `Strict` | Same-site requests only. | Untrusted same-site origins remain a risk; external navigation may initially omit the login cookie. |
| Explicit `Lax` | Same-site requests, plus cross-site top-level navigations using safe methods such as `GET`. | State-changing `GET` endpoints are exposed; untrusted same-site origins remain a risk. |
| `None` | Same-site and cross-site requests; requires `Secure`. | Provides no SameSite-based CSRF protection. |

Explicit `Lax` blocks a major attack path: an ordinary cross-site form `POST` carrying the session cookie. But there are gaps.

### Same-site is broader than same-origin

An **origin** is scheme + host + port. For ordinary domain URLs, a modern schemeful **site** is scheme + registrable domain; subdomains and ports do not distinguish sites.

```text
https://app.example.com
https://untrusted.example.com
```

These are different origins but the same site. An attacker controlling the second host can initiate requests to the first that qualify as same-site, even with `SameSite=Strict`.

This does **not** require the attacker to read the application's session cookie or the cookie to be shared across subdomains: the browser can attach a host-only cookie when sending the request to its matching host. A session-bound CSRF token still helps because the attacker origin normally cannot read it.

### Lax navigation and browser defaults

- `Lax` allows cross-site top-level navigation using safe methods. **Do not perform state changes through `GET`**, such as `GET /delete-account`; use a protected state-changing request instead.
- Set `SameSite` explicitly. Some browsers' default-Lax behavior permits top-level cross-site `POST` requests for cookies created within the previous two minutes. That exception is not the normal behavior of explicit `SameSite=Lax`.

### When tokens can be omitted

Tokens are not the only CSRF defense. Depending on the application, alternatives include:

- Server-side validation of exact allowed request origins, with an explicit policy for missing or `null` origins and any `Referer` fallback.
- APIs that require a custom request header on every state-changing request, reject requests without it, and restrict CORS to trusted origins. The header forces cross-origin preflight; simply configuring CORS is not enough.
- Authentication using credentials that application code explicitly adds, such as an `Authorization` bearer token, **provided no automatically attached authentication fallback is accepted**. Putting a JWT in a cookie does not remove CSRF risk.

Omitting tokens requires another defense with understood assumptions—not merely the presence of a SameSite attribute.

## References

- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [MDN: Set-Cookie, including SameSite and HttpOnly](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
