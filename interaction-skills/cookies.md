# Cookies

Dump, inspect, and restore a Chrome session's cookie jar via CDP. For **cookie-auth sites** (X/Twitter, Instagram, Facebook, most SaaS) cookies ARE the full session — dump, stash the blob, restore later, you're still logged in.

**Cookies alone are not always enough.** Many SPAs (LinkedIn, Notion, Slack, most internal tools) stash part of their auth state in `localStorage` too. Cookie-only restore gets you the session cookie but the app still forces re-auth because its client-side state is missing. For those, skip to [Also save localStorage](#also-save-localstorage) below and capture both in one blob.

Neither variant covers: IndexedDB, service-worker caches, or Chrome user-data (passwords, extensions, autofill). See [What this still doesn't cover](#what-this-still-doesnt-cover).

## Fast start — save (cookies only)

```bash
browser-harness <<'PY'
import json, gzip, pathlib
out = pathlib.Path("~/.browser-harness/profiles/work.json.gz").expanduser()
out.parent.mkdir(parents=True, exist_ok=True)
jar = cdp("Network.getAllCookies")["cookies"]
out.write_bytes(gzip.compress(json.dumps({"cookies": jar}).encode()))
print(f"saved {len(jar)} cookies → {out}")
PY
```

## Fast start — restore (cookies only)

```bash
browser-harness <<'PY'
import json, gzip, pathlib
blob = json.loads(gzip.decompress(
    pathlib.Path("~/.browser-harness/profiles/work.json.gz").expanduser().read_bytes()))
cdp("Network.setCookies", cookies=blob["cookies"])
print(f"restored {len(blob['cookies'])} cookies")
PY
```

Then `new_tab("https://x.com")` — you should land on the logged-in feed, not the login wall.

`Network.setCookies` (plural) accepts the exact shape `getAllCookies` returns — `httpOnly`, `secure`, `sameSite`, `expires`, `partitionKey` all round-trip without translation. Prefer the plural form over `Network.setCookie` (singular) when replaying a captured jar; the singular form has stricter validation and will reject fields the plural accepts.

## Filter by domain

`getAllCookies` returns everything — often thousands of cookies, most of it ad trackers you don't need. Narrow to the sites you actually care about:

```bash
browser-harness <<'PY'
import json, gzip, pathlib
KEEP = ["linkedin.com", "x.com", "twitter.com", "instagram.com", "facebook.com"]

def matches(c):
    d = c["domain"].lstrip(".")
    return any(d == s or d.endswith("." + s) for s in KEEP)

jar = [c for c in cdp("Network.getAllCookies")["cookies"] if matches(c)]
out = pathlib.Path("~/.browser-harness/profiles/socials.json.gz").expanduser()
out.parent.mkdir(parents=True, exist_ok=True)
out.write_bytes(gzip.compress(json.dumps({"cookies": jar}).encode()))
print(f"saved {len(jar)} cookies from {len({c['domain'] for c in jar})} domains")
PY
```

Cookie `domain` values are either bare hostnames (host-only cookies: `www.linkedin.com`) or leading-dot forms (`.linkedin.com` = host + all subdomains). Stripping the leading dot and matching both `d == "linkedin.com"` and `d.endswith(".linkedin.com")` catches both.

## Inspect a saved blob before restoring

Always look before you load. The agent should report what it's about to pour into the browser, not do it silently.

```bash
browser-harness <<'PY'
import json, gzip, pathlib, collections
blob = json.loads(gzip.decompress(
    pathlib.Path("~/.browser-harness/profiles/socials.json.gz").expanduser().read_bytes()))
by_domain = collections.Counter(c["domain"] for c in blob["cookies"])
for dom, n in sorted(by_domain.items(), key=lambda kv: -kv[1])[:20]:
    print(f"{n:4}  {dom}")
for origin, data in blob.get("origins", {}).items():
    print(f"localStorage  {origin}  {len(data.get('localStorage', {}))} items")
PY
```

Never dump raw cookie values into chat. They're bearer credentials — the user shouldn't see them and most are opaque tokens anyway. A per-domain count is all you need to summarize *"you have logins for linkedin, x.com, facebook — restore?"*

## Chat-driven flow

Cookies are real auth. Handle them with the same care as passwords.

- **Before saving**: tell the user which domains (and origins, if capturing localStorage) you're about to capture and where the blob will land. Let them narrow the list.
- **Before restoring**: show the per-domain count + per-origin localStorage counts from the inspect snippet. Let them confirm.
- **After restoring**: navigate to one or two key origins and check `page_info()` — does the title show the logged-in UI, or the login wall? Report which sites actually picked up the session.

## Also save localStorage

Some sites won't trust a cookie-only restore. LinkedIn is the canonical example: its `li_at` cookie gets you partway in, but a bundle of `voyager-web:*` entries in `localStorage` plus a device-trust token carry the rest of the auth state. Without them, LinkedIn shows the login wall even though the cookie is valid.

Save cookies + localStorage for an explicit set of origins:

```bash
browser-harness <<'PY'
import json, gzip, pathlib

# Ask the user which origins matter. Don't guess — localStorage is
# origin-scoped, you have to visit each one, and each visit is a real
# page load the user is paying latency for.
ORIGINS = [
    "https://www.linkedin.com",
    "https://x.com",
    "https://www.instagram.com",
]

blob = {"cookies": cdp("Network.getAllCookies")["cookies"], "origins": {}}

for origin in ORIGINS:
    new_tab(origin)
    wait_for_load()
    raw = js("JSON.stringify(Object.fromEntries(Object.entries(localStorage)))")
    blob["origins"][origin] = {"localStorage": json.loads(raw) if raw else {}}

out = pathlib.Path("~/.browser-harness/profiles/full.json.gz").expanduser()
out.parent.mkdir(parents=True, exist_ok=True)
out.write_bytes(gzip.compress(json.dumps(blob).encode()))
total_ls = sum(len(v["localStorage"]) for v in blob["origins"].values())
print(f"saved {len(blob['cookies'])} cookies + {total_ls} localStorage items across {len(blob['origins'])} origins")
PY
```

Restore cookies first (everything at once), then walk each origin and replay its localStorage, then reload so the app re-initializes with both:

```bash
browser-harness <<'PY'
import json, gzip, pathlib
blob = json.loads(gzip.decompress(
    pathlib.Path("~/.browser-harness/profiles/full.json.gz").expanduser().read_bytes()))

# 1. Install the full cookie jar BEFORE any origin is loaded.
cdp("Network.setCookies", cookies=blob["cookies"])
print(f"restored {len(blob['cookies'])} cookies")

# 2. For each origin: navigate (cookies attach automatically), set its
#    localStorage, then reload so the app's init code runs with BOTH
#    cookies and localStorage available. The reload is load-bearing —
#    without it, the first render decided "not logged in" before
#    localStorage existed, and a lot of SPAs cache that conclusion.
for origin, data in blob.get("origins", {}).items():
    new_tab(origin)
    wait_for_load()
    for k, v in data.get("localStorage", {}).items():
        js(f"localStorage.setItem({json.dumps(k)}, {json.dumps(v)})")
    goto(origin)
    wait_for_load()
    print(f"  {origin}: {page_info().get('title', '?')}")
PY
```

Order matters:

1. **Cookies before anything else.** `setCookies` with no pages loaded installs every entry without filtering.
2. **Navigate to each origin.** localStorage is origin-scoped — same-origin policy means you can only `setItem` on the origin's own document.
3. **Set localStorage items on that origin's document.**
4. **Reload (`goto(origin)`) so the site's init JS runs again**, now with both cookies and localStorage present.

## Traps

- **Restore cookies first, navigate second.** Once a page is loaded, `setCookies` filters entries whose `domain` doesn't match the current origin's allowable-set. Install into a blank browser or `about:blank` tab, then navigate — every cookie lands.
- **localStorage needs the origin's document context.** You can't `setItem` for `linkedin.com` from an `about:blank` tab — the origin has to be the current tab's origin. That's why the snippet above navigates before setting.
- **Session cookies vanish when Chrome exits.** Entries with `session: true` (no `expires`) die on browser shutdown. Restoring them into a fresh browser works, but save-then-restore across a hard Chrome restart will miss them. `SameSite=Strict` + session auth (some banking / admin consoles) is the worst case; expect a re-login.
- **Partitioned cookies (CHIPS, `partitionKey` set)** are scoped to the top-frame origin. Keep the `partitionKey` field on restore — `setCookies` preserves it if you pass the cookie through unchanged.
- **Auth state is plaintext on disk.** Whatever path you pick, lock it down: `chmod 700 ~/.browser-harness/profiles` + `chmod 600` on the blobs. Don't put it on shared / synced / cloud-backed storage. Anyone who reads the file owns the account.
- **Never commit blobs to git.** Add `*.json.gz` under your profile dir to `.gitignore`. Even for demos — the whole point is that it's a bearer credential.
- **Domain mismatch is silent.** If `setCookies` drops cookies (wrong domain, malformed), it doesn't raise; it just returns fewer-than-expected successes. After restore, navigate to a canary site and verify you're logged in — don't trust return codes.
- **Origins are strict.** `https://www.linkedin.com` and `https://linkedin.com` are different localStorage scopes. If `new_tab("https://linkedin.com")` redirects to `www.linkedin.com`, you've just written localStorage to the wrong bucket. Match what the user's browser actually uses.
- **Protocol is part of the origin.** HTTP and HTTPS don't share localStorage.
- **Quota.** localStorage is ~5-10MB per origin; if a site has stuffed video thumbnails or analytics blobs in there, replay can fail at `setItem` with `QuotaExceededError`. Wrap the inner loop in `try/except` if you've hit it.
- **Device fingerprinting is orthogonal to this doc.** Some sites (LinkedIn, Google, banks) fingerprint User-Agent, IP, timezone, canvas rendering. Even a perfect cookie + localStorage restore on a different machine/IP can trigger *"We don't recognize this device — please log in."* No storage-layer fix; you need to match fingerprints or accept the re-auth.

## What this still doesn't cover

- **IndexedDB.** Some SPAs cache auth tokens or session state there. CDP exposes `IndexedDB.*` methods but round-tripping a full DB (with schemas, versions, object stores) is its own doc.
- **`sessionStorage`.** Dies with the tab by design. You *can* capture with `Object.entries(sessionStorage)` and replay with `sessionStorage.setItem`, but you're forging a fresh session with synthetic state — most sites tolerate it, some don't. Not covered above; follow the localStorage pattern if you need it, and swap `localStorage` for `sessionStorage` throughout.
- **Service worker registrations and cache storage.** Usually re-register on page load, so restore "just works" once the user is back online. If a site relies on SW-cached auth tokens, cookies + localStorage won't help.
- **Full Chrome profile (passwords, history, extensions, download prefs, autofill).** That's a filesystem copy of the entire user-data dir, not a CDP operation. For the Browser Use cloud (remote browsers), see `profile-sync.md` — it handles the SaaS variant of "ship my local Chrome profile to a remote browser."
- **Cross-engine (Chrome → Firefox).** Cookie jars and localStorage formats differ across browser engines. Save and restore within the same Chrome / Chromium family.
