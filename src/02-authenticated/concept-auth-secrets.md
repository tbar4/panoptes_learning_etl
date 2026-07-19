# Concept: Session Auth & Secrets from the Environment

> **Kind:** Concept. **Seam introduced this arc:** an authenticated source — a login handshake that yields a session cookie, plus the discipline of keeping secrets out of the URL and off disk.

## The idea

CelesTrak was an *open* GET: no key, no login, anyone can pull the TLE catalog. Space-Track is not. Before it will answer a single query it wants you to **log in** — POST your username and password to a login endpoint, and in return it hands you a **session cookie**. Every subsequent request must carry that cookie, or the server treats you as a stranger and refuses.

That is the shape of a huge fraction of real-world APIs, and it forces two questions this chapter answers:

1. **How does the login handshake work as code?** You POST credentials once, pull the cookie off the response, and forward it by hand on the query that follows. Two requests, one session.
2. **Where do the credentials come from — and where must they *never* go?** They come from the **environment**. They must never be hard-coded in the source, never written to disk, and — the one people get wrong — never placed in a **URL**.

The cookie mechanic is mechanical; the secrets discipline is the part that separates a toy from something you would run against a real account. We take both in turn, on a toy you cannot mistake for the answer key, then map it onto the shipped `SpaceTrackSource`.

## A runnable toy: the members-only clubhouse

Here is the whole session dance on a domain that is obviously not orbital data — a members-only clubhouse. The "server" knows the correct password and, on a good login, mints a session cookie. The lounge only serves callers who present that cookie. Everything is synchronous `std`, so it runs on the playground:

```rust
/// A members-only clubhouse. The server knows the correct password and, on a
/// successful login, hands back a short-lived session token — a cookie.
struct Clubhouse {
    password: String,
    valid_cookie: String,
}

impl Clubhouse {
    /// POST /login — check the password, mint a session cookie.
    fn login(&self, member: &str, pass: &str) -> Result<String, &'static str> {
        if pass == self.password {
            Ok(format!("session={}; member={member}", self.valid_cookie))
        } else {
            Err("401 bad credentials")
        }
    }

    /// GET /lounge — only serves callers who present the session cookie.
    fn lounge(&self, cookie: &str) -> Result<String, &'static str> {
        // The server reads the cookie back off the request, exactly as it
        // issued it — no re-sending of the password.
        if cookie.contains(&format!("session={}", self.valid_cookie)) {
            Ok("today's special: orbital-mechanics trivia night".into())
        } else {
            Err("403 no valid session")
        }
    }
}

fn main() {
    let server = Clubhouse {
        password: "hunter2".into(),
        valid_cookie: "abc123".into(),
    };

    // The secret comes from the environment, with a fallback so this toy runs
    // anywhere. In production there is no fallback — a missing var is an error.
    let secret = std::env::var("CLUB_PASSWORD").unwrap_or_else(|_| "hunter2".into());

    // 1. Log in once. The password crosses the wire in the POST body, never
    //    in the URL.
    let cookie = server.login("ada", &secret).expect("login should succeed");
    println!("got cookie:  {cookie}");

    // 2. Every later request forwards the cookie by hand — not the password.
    match server.lounge(&cookie) {
        Ok(data) => println!("lounge says: {data}"),
        Err(e) => println!("denied:      {e}"),
    }

    // 3. A request with no session is refused.
    match server.lounge("session=stolen-guess") {
        Ok(data) => println!("lounge says: {data}"),
        Err(e) => println!("no cookie:   {e}"),
    }

    // The anti-pattern, shown but not used: the secret in the URL query string.
    // This string would land in access logs, proxy caches, and browser history.
    let leaky_url = format!("https://club.example/lounge?password={secret}");
    let safe_url = "https://club.example/lounge".to_string();
    println!("leaky url:   {leaky_url}");
    println!("safe url:    {safe_url}  (+ Cookie header)");
}
```

```text
got cookie:  session=abc123; member=ada
lounge says: today's special: orbital-mechanics trivia night
no cookie:   403 no valid session
leaky url:   https://club.example/lounge?password=hunter2
safe url:    https://club.example/lounge  (+ Cookie header)
```

Read the session lifecycle off those three calls:

- **The password crosses the wire exactly once**, in the `login` call. After that, only the *cookie* travels. If the cookie leaks, an attacker gets a bounded session; the password — the reusable, un-rotatable secret — stayed put.
- **The cookie is forwarded by hand.** The client holds the string returned from `login` and passes it into `lounge`. Nothing is automatic: no cookie jar, no framework magic. This is deliberately explicit because the real source does exactly this, one line at a time.
- **No cookie means no service.** The third call presents a bogus session and gets `403`. The server does not fall back to re-authenticating; a missing or wrong cookie is simply a refusal.

## Secrets come from the environment — and never touch the URL

Look again at where `secret` came from: `std::env::var("CLUB_PASSWORD")`. That is the whole rule for credentials. The environment is a per-process channel that is *not* committed to git, *not* baked into the binary, and *not* written to any file the program controls. A deployment sets the variable; the code reads it; nobody's password ends up in a commit.

Reading an environment variable is safe and ordinary. Note, though, that in Rust 2024 *writing* one (`std::env::set_var`) is an `unsafe` operation — it can race other threads reading the environment — a small, pointed reminder that the environment is process-global state you read from, not a scratchpad you scribble into. Your source only ever reads.

Now the part people get wrong. The two printed URLs are identical *except* that the leaky one carries the password as a query parameter. Both would "work." Only one is safe:

<div class="callout trap">
<span class="callout-label">Never put a secret in the URL</span>
A URL is the single least-private part of an HTTP request. Query strings are written verbatim into web-server access logs, cached by intermediary proxies, retained in browser and CLI history, and echoed back in <code>Referer</code> headers. A password or API key in the query string is a password in a dozen log files you do not control and cannot scrub. Secrets ride in the request <em>body</em> (the login POST) or in a <em>header</em> (the cookie, an <code>x-api-key</code>) — places that are not routinely logged. The rule is absolute: if it is secret, it is never in the URL.
</div>

## The mapping onto the shipped `SpaceTrackSource`

The clubhouse is `SpaceTrackSource` with the domain swapped and the I/O made real. Against the shipped `crates/etl-sources/src/spacetrack.rs`:

- **`Clubhouse::password` + the env read** ↔ **`Credentials`**, a two-field struct (`user`, `pass`) built either by `Credentials::new("u", "p")` or by `Credentials::from_env()`, which reads `SPACETRACK_USER` and `SPACETRACK_PASS` and returns `EtlError::Auth("… not set")` when either is missing. Missing credentials are an auth error, classified *permanent* — no amount of retrying conjures a password.
- **`login`** ↔ a real `reqwest` POST to `{base}/ajaxauth/login`, sending the credentials as **form fields** (`identity`, `password`) in the request body — not the URL.
- **pulling the cookie off the response** ↔ reading the `SET_COOKIE` response header, keeping only the first `;`-separated segment (the `name=value` pair, dropping `Path`, `HttpOnly`, and friends).
- **forwarding the cookie to `lounge`** ↔ attaching that string as the `COOKIE` request header on the follow-up GET to the CDM query endpoint.

Here is the shipped handshake, trimmed to the two requests and the cookie in between. It uses `reqwest`, `async-trait`, and `chrono`, so it is `rust,ignore` — build and test it in the workspace with `cargo test -p etl-sources`, never on the playground:

```rust,ignore
async fn extract(&self) -> Result<Vec<RawRecord>, EtlError> {
    let base = self.base_url.trim_end_matches('/');
    let client = reqwest::Client::new();

    // 1. Log in. Credentials ride in the form body, never the URL.
    self.limiter.acquire().await; // (rate limiting — next chapter)
    let login = client
        .post(format!("{base}/ajaxauth/login"))
        .form(&[
            ("identity", self.creds.user.as_str()),
            ("password", self.creds.pass.as_str()),
        ])
        .send()
        .await
        .map_err(|e| EtlError::Network(e.to_string()))?;
    if !login.status().is_success() {
        // Non-2xx at login funnels through the SAME classifier as any request:
        // 401/403 -> Auth (permanent); 5xx/429 -> transient, still retryable.
        return Err(crate::http::classify_status(login.status(), login.headers()));
    }

    // 2. Pull the session cookie off the response, keep only "name=value".
    let cookie = login
        .headers()
        .get(reqwest::header::SET_COOKIE)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.split(';').next())
        .ok_or_else(|| EtlError::Auth("login returned no session cookie".into()))?
        .to_string();

    // 3. Query, carrying the cookie by hand in the COOKIE header.
    self.limiter.acquire().await;
    let resp = client
        .get(format!(
            "{base}/basicspacedata/query/class/cdm_public/orderby/TCA/format/json"
        ))
        .header(reqwest::header::COOKIE, cookie)
        .send()
        .await
        .map_err(|e| EtlError::Network(e.to_string()))?;
    if !resp.status().is_success() {
        return Err(crate::http::classify_status(resp.status(), resp.headers()));
    }
    let payload = resp.text().await.map_err(|e| EtlError::Network(e.to_string()))?;
    Ok(vec![RawRecord { source: self.name().to_string(), payload, fetched_at: Utc::now() }])
}
```

The shape is the toy's, line for line: log in → keep the cookie → forward it → read the body. Only the transport and the domain changed. The `self.limiter.acquire().await` calls belong to the next chapter — for now, read them as "wait my turn before each request."

## Transient vs permanent — *at login*, not just at query

Here is the detail that pays off two arcs later, and it is worth pausing on. Both the login request and the query request funnel their non-`2xx` responses through the *same* `classify_status` from Part II. That is not just code reuse — it is the point.

A login can fail two very different ways:

- **`401` / `403`** — the credentials are wrong. This is `EtlError::Auth`, and it is **permanent**. Retrying sends the identical wrong password and gets the identical rejection; the retry loop must give up immediately and tell you to fix your credentials.
- **`503` / `429`** — Space-Track's login endpoint is briefly overloaded or throttling you. This is `EtlError::Network` or `EtlError::RateLimited`, and it is **transient**. The credentials are fine; the server was momentarily unavailable. The retry loop *should* wait and try the whole handshake again.

If login errors were all bucketed as "auth failed," a transient `503` at the login step would abort a run that a single retry would have completed. Because login goes through `classify_status`, the transient/permanent decision you built once in Part II is made correctly here too — a failed *login* is classified by the same rules as a failed *query*.

<div class="callout">
<span class="callout-label">The payoff</span>
One classifier, used at every HTTP boundary, means the retry loop in Part IV never has to special-case "was this a login or a query?" It asks the error <code>is_transient()</code> and the answer is already right, whether the failure happened during the handshake or the data fetch.
</div>

## Questions to lock

1. Walk the session handshake in three steps: what is sent in the login request, what comes back, and what the client attaches to the follow-up request. Which secret crosses the wire, and how many times?
2. Where do credentials come from in the shipped source, and what are the three places a secret must never go?
3. Why is a secret in a URL query string worse than the same secret in a request header or body? Name two places the query-string version leaks to.
4. `Credentials::from_env` returns `EtlError::Auth` when a variable is missing. Is that transient or permanent, and why is that the right call?
5. A login attempt comes back `503`. Should the run give up or retry, and which function makes that decision correctly for *both* the login and the query steps?

The graded quiz for this arc is at the end of the part, on the [Concept-Check: Authentication](./quiz.md) page.
