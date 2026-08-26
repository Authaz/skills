# Keep the session alive — refresh tokens

For when Authaz is already wired up and users are being bounced to the login screen while they work. If Authaz isn't set up yet, follow the matching `setup-*.md` recipe first.

## What the SDK already does for you

`createAuthazHandler` mounts `POST /api/auth/refresh` for free. It is not something you build:

| Step | Owner |
|---|---|
| Storing the refresh token at callback time | Handler — `authaz_refresh_token` cookie, HttpOnly, 30 days |
| Exchanging it (`grant_type=refresh_token`) | Handler — `client.auth.refreshTokens()` |
| **Rotating** it and rewriting both cookies | Handler |
| Clearing both cookies when the refresh token is spent | Handler |
| **Deciding when to call it** | **You** |

That last row is the whole recipe. Nothing in the SDK calls `/api/auth/refresh` on its own except `<AuthazProvider autoRefresh>` on the React client — so a server-rendered app refreshes exactly never until you wire it.

## Step 1 — Confirm the symptom is expiry, not something else

The tell is a session that dies on a clock, not on an action:

```
authaz_access_token   maxAge = expiresIn   (typically 3600s)
authaz_refresh_token  maxAge = 30 days
```

Because the access cookie's `maxAge` *is* the token's lifetime, an expired access token is not a stale cookie the server can inspect — **the browser has already dropped it**. So every server-side guard (`requireUser`, `createAuthMiddleware`, `withAuth`, `requireAuth`) sees "no cookie" and reads it as "signed out", while a perfectly valid refresh token sits in the next cookie over.

Symptoms that match:

- Users are redirected to Universal Login roughly once an hour, mid-task.
- A dashboard tab left open returns `401` on its next `fetch`.
- It only happens on the *server* side of a Next.js app; a React SPA with `autoRefresh` is fine.

Symptoms that do **not** match — go to `troubleshoot-oauth.md` instead: `invalid_grant` right after login (PKCE/redirect-uri), signed out immediately on every request (cookie `secure`/`sameSite` vs. http), signed out only on one browser.

## Step 2 — Refresh on the server, in middleware

Middleware/proxy is the only place in a server-rendered request that can both read the incoming cookies and *write* new ones. A Server Component cannot set a cookie, so it can never repair its own session.

```ts
// proxy.ts (Next.js 16) — middleware.ts on 15 and earlier, same body
import { NextResponse, type NextRequest } from "next/server";

const ACCESS = "authaz_access_token";   // COOKIE_NAMES.ACCESS_TOKEN
const REFRESH = "authaz_refresh_token"; // COOKIE_NAMES.REFRESH_TOKEN

export async function proxy(request: NextRequest) {
  const stale = !request.cookies.has(ACCESS) && request.cookies.has(REFRESH);
  // Never on /api/auth — that is the refresh endpoint itself, plus the login and callback flow.
  if (!stale || request.nextUrl.pathname.startsWith("/api/auth")) return NextResponse.next();

  const refreshed = await fetch(new URL("/api/auth/refresh", request.nextUrl.origin), {
    method: "POST",
    headers: { cookie: request.headers.get("cookie") ?? "" },
    cache: "no-store",
  });
  if (!refreshed.ok) return NextResponse.next(); // refresh token spent — fall through to login

  const setCookies = refreshed.headers.getSetCookie();
  // On the REQUEST, so the page rendering *now* sees the new session…
  for (const header of setCookies) {
    const [name, ...rest] = header.split(";")[0].split("=");
    request.cookies.set(name.trim(), rest.join("=").trim());
  }
  // …and on the RESPONSE, so the browser keeps it.
  const response = NextResponse.next({ request: { headers: request.headers } });
  for (const header of setCookies) response.headers.append("set-cookie", header);
  return response;
}

export const config = { matcher: ["/api/:path*", "/dashboard/:path*"] };
```

Call the handler over HTTP rather than `client.auth.refreshTokens()` directly: the handler owns rotation, cookie lifetimes and the clear-both-cookies failure path, and it reads `next/headers`, which middleware does not have.

Both writes are load-bearing. Setting only the response cookie leaves the current render signed out, so the user still sees one spurious login redirect per expiry — the bug you were fixing, just less often.

## Step 3 — Refresh on the client, once at a time

Middleware covers navigation. It does not cover a tab that sits still and then fires a `fetch`. Retry once behind a refresh:

```ts
let inFlight: Promise<boolean> | null = null;

function refresh() {
  inFlight ??= fetch("/api/auth/refresh", { method: "POST", cache: "no-store" })
    .then((r) => r.ok)
    .catch(() => false)
    .finally(() => { inFlight = null; });
  return inFlight;
}

export async function fetchWithSession(input: RequestInfo, init?: RequestInit) {
  const response = await fetch(input, init);
  if (response.status !== 401) return response;
  if (!await refresh()) return response;
  return fetch(input, init);
}
```

**The shared promise is mandatory, not an optimisation.** Authaz rotates the refresh token on every exchange and treats a second use of the old one as a stolen token — `invalid_grant`, *refresh-token reuse detected*, and the whole token family is invalidated. A page that fires three requests at once and refreshes three times in parallel signs the user out for real. One refresh in flight; everyone else waits on it.

React SPA? You get this for free — `<AuthazProvider basePath="/api/auth" autoRefresh={true}>` refreshes ahead of expiry. Don't hand-roll on top of it.

## Step 4 — Verify

1. Sign in. In devtools, delete **only** the `authaz_access_token` cookie (leave `authaz_refresh_token`).
2. Navigate to a protected page. Expected: the page renders, and the response carries two fresh `Set-Cookie` headers. Failure: a redirect to Universal Login.
3. Delete `authaz_access_token` again and click something that fires a `fetch`. Expected: one `POST /api/auth/refresh` in the network tab, then the original request replayed and succeeding.
4. Delete **both** cookies. Expected: redirect to login — this path must still work.
5. Fire several requests at once with the access token deleted. Expected: exactly **one** `POST /api/auth/refresh`.

## Tuning the lifetimes

Lifetimes are application config, not code. Via `authaz apply` (see the `authaz-cli` skill) or the Dashboard:

```yaml
spec:
  authentication:
    settings:
      accessTokenLifetime: 3600      # seconds
      refreshTokenLifetime: 2592000  # seconds — 30 days
```

Shortening `accessTokenLifetime` is a real security lever (a leaked token expires sooner) and costs only more refreshes once the wiring above exists. Lengthening it to "fix" the redirects is the wrong lever — it makes the window worse and does not remove the cliff.

## Anti-patterns

- **Don't call `/api/auth/refresh` on a timer.** Refresh on demand — a token that expires with the tab closed does not need renewing, and every rotation is a chance to lose the race.
- **Don't refresh without a de-dupe.** Rotation plus reuse detection turns concurrent refreshes into a forced logout.
- **Don't refresh inside a Server Component.** It cannot set cookies, so the new token is discarded and you refresh again on the very next request.
- **Don't reimplement the exchange.** Call the mounted handler; `auth.refreshTokens()` directly means owning rotation and cookie policy yourself.
- **Don't read the token's `exp` to pre-empt expiry on the server.** The cookie is already gone by then; presence *is* the signal.
- **Don't retry more than once.** A second 401 after a successful refresh is an authorization problem, not an expiry one — surface it.

## References

- Handler source: `authaz-sdk-js/packages/next/src/index.tsx` — `handleRefresh`
- `troubleshoot-oauth.md` — `invalid_grant`, reuse detection, clock skew
- `protect-route.md` — where the guards this recipe repairs actually live
- `endpoints.md` — the full route table the handler mounts
