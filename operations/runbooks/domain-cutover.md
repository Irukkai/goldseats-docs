# Runbook — Domain Cutover

**Last drilled:** not applicable. This is a one-time procedure, executed as a
[M9](../../product/milestones/M9.md) deliverable. The DNS-emergency section at the bottom is the
part worth rehearsing.

---

## The situation

DNS for `goldseats.app` currently contains:

```
CNAME www.goldseats.app -> stellar-hotteok-63503a.netlify.app
```

**The Netlify account that owns `stellar-hotteok-63503a` has been lost.** We cannot log into it,
cannot redeploy it, cannot change its settings, and cannot delete the site. The landing page
currently served at `www.goldseats.app` is frozen and unmanageable.

**Why this is survivable: we control the registrar.** DNS authority is what actually matters. We
can point `goldseats.app` anywhere we like regardless of who owns the Netlify site behind that
CNAME. The orphaned deploy simply stops receiving traffic and becomes irrelevant.

What we lose: nothing of value. `index.html` is in the repo, it's being migrated into
`goldseats-web` in [M3](../../product/milestones/M3.md) anyway, and there is no data or
configuration in that Netlify account we need.

What we must be careful about: **not breaking `www.goldseats.app` during the transition**, since
it's the live marketing page and the only public presence GoldSeats currently has.

---

## When to use this

- Executing the [M9](../../product/milestones/M9.md) cutover to a new, owned host
- `goldseats.app` or `www.goldseats.app` isn't resolving
- TLS certificate errors on either
- `www` and apex behaving inconsistently
- You need to move hosts for any reason

---

## Prerequisites

**Verify all of these before starting.** Discovering missing registrar access mid-cutover is the
bad outcome.

- [ ] **DNS registrar login for `goldseats.app`**, MFA working, tested
- [ ] A hosting account **we own**, with `goldseats-web` deployed and verified on its default
      hostname
- [ ] The API deployed and reachable
- [ ] `dig` available locally
- [ ] Nobody mid-release — freeze deploys for the cutover window
- [ ] An hour of uninterrupted time

Confirm registrar access first, explicitly:

```bash
dig +short NS goldseats.app
dig +short CNAME www.goldseats.app     # expect stellar-hotteok-63503a.netlify.app
dig +short A goldseats.app
```

Then actually log into the registrar and confirm you can edit a record. Reading DNS proves
nothing about your ability to change it.

---

## Part 1 — Before the cutover

### Step 1 — Record the current state

```bash
{
  echo "=== $(date -u) ==="
  dig +noall +answer goldseats.app      A
  dig +noall +answer goldseats.app      AAAA
  dig +noall +answer www.goldseats.app  CNAME
  dig +noall +answer goldseats.app      MX
  dig +noall +answer goldseats.app      TXT
  dig +noall +answer goldseats.app      NS
  dig +noall +answer goldseats.app      CAA
} | tee dns-before-cutover.txt
```

Commit that file to `goldseats-docs` or store it with the incident. **Check for MX and TXT
records** — if email for the domain works today, breaking SPF or DKIM during a web cutover is an
easy and painful mistake.

### Step 2 — Confirm the old account is unrecoverable

Attempt recovery properly once, and record the attempt:

- [ ] Password reset on every plausible email address
- [ ] Check for a linked GitHub or Google identity that might still authenticate
- [ ] Netlify support, quoting the site name `stellar-hotteok-63503a` and proving domain control

**We don't need recovery to proceed.** This step exists so the post-cutover note can say we
tried, and so we're not leaving an account with our domain configured in it if it's reclaimable.

Record the outcome in this runbook when it's known.

### Step 3 — Choose and verify the new host

An open founder decision — see
[`../environments.md`](../environments.md#hosting--an-open-decision). Netlify under a new owned
account, or Vercel.

Whichever: deploy `goldseats-web` and verify it fully on the host's default hostname **before
touching DNS**.

```bash
export NEWHOST="goldseats-web-xyz.vercel.app"   # or the Netlify equivalent

curl -fsS "https://$NEWHOST/" | grep -q 'GoldSeats'
curl -fsS "https://$NEWHOST/films" > /dev/null
curl -sI "https://$NEWHOST/" | head -20
```

Then in a browser: load it, check the console, check a film page, check it on a phone. A cutover
to a broken deploy is worse than no cutover.

### Step 4 — Add the custom domain on the new host

Do this **before** changing DNS. Most hosts will accept the domain, show it as pending
verification, and pre-provision what they can.

1. Add both `goldseats.app` and `www.goldseats.app` in the host's domain settings
2. Note the exact target values it gives you — an A record for apex, a CNAME for `www`, or an
   ALIAS/ANAME
3. Note the TLS provisioning method — most do ACME automatically once DNS resolves

### Step 5 — Lower the TTL

**At least 48 hours before the cutover.** This is the step that determines whether a mistake is
recoverable in five minutes or in a day.

At the registrar, set TTL to `300` on:

- `goldseats.app` A / ALIAS
- `www.goldseats.app` CNAME

```bash
dig goldseats.app     | grep -A1 'ANSWER SECTION'
dig www.goldseats.app | grep -A1 'ANSWER SECTION'
```

Wait for the **old** TTL to expire before assuming the new one is in effect everywhere. If the
old TTL was 86400, that's a full day.

---

## Part 2 — The cutover

Target window: 15 minutes of work, then up to an hour of propagation.

### Step 6 — Change the records

At the registrar:

| Record | From | To |
| --- | --- | --- |
| `goldseats.app` (apex) | current A | new host's A / ALIAS |
| `www.goldseats.app` | `stellar-hotteok-63503a.netlify.app` | new host's target |

**Leave MX, TXT, SPF, DKIM, DMARC, and CAA untouched.** You are changing web hosting, not email.

If the new host needs a CAA record for certificate issuance, add it — don't replace an existing
one without checking what depends on it.

### Step 7 — Watch propagation

```bash
watch -n 10 'dig +short www.goldseats.app; echo ---; dig +short goldseats.app'
```

Check from outside your own resolver too:

```bash
dig +short @1.1.1.1 www.goldseats.app
dig +short @8.8.8.8 www.goldseats.app
dig +short @9.9.9.9 goldseats.app
```

With TTL at 300, expect most resolvers within 5–10 minutes and stragglers within an hour.

### Step 8 — Wait for TLS

The host provisions a certificate once DNS resolves to it. Usually automatic within minutes.

```bash
echo | openssl s_client -servername goldseats.app -connect goldseats.app:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

Confirm: the subject covers both `goldseats.app` and `www.goldseats.app`, and the expiry is
roughly 90 days out for an ACME certificate.

**If TLS doesn't provision within 30 minutes:** check for a CAA record blocking the issuer, and
check the host's dashboard for a verification error.

### Step 9 — Verify

```bash
# Both hostnames serve the app
curl -fsS https://goldseats.app/     | grep -q 'GoldSeats'
curl -fsS https://www.goldseats.app/ | grep -q 'GoldSeats'

# HTTP redirects to HTTPS
curl -sI http://goldseats.app/ | grep -i '^location'

# www and apex agree on a canonical form
curl -sI https://www.goldseats.app/ | grep -iE '^(HTTP|location)'

# Security headers present
curl -sI https://goldseats.app/ | grep -iE 'strict-transport|content-security|x-content-type'

# API reachable
curl -fsS https://api.goldseats.app/health/ready | jq '{status, version}'

# The old target is no longer in the path
dig +short www.goldseats.app | grep -qv 'netlify' && echo "off the orphaned site"
```

Then in a browser, on a real phone and a desktop:

- Home page, a film page, a seat map
- No mixed-content or console errors
- Favicon and Open Graph preview correct
- Lighthouse still above the budgets in
  [`../../engineering/performance-budgets.md`](../../engineering/performance-budgets.md)

### Step 10 — Restore the TTL

**After 48 hours of stability**, raise TTL back to `3600` for the apex and `www`. Leaving it at
300 forever means more DNS queries and no benefit once you're settled.

### Step 11 — Close out

- [ ] Update [`../environments.md`](../environments.md) with the chosen host, removing the "open
      decision" note
- [ ] Commit `dns-after-cutover.txt` alongside the before state
- [ ] Set up certificate expiry monitoring, alerting at 14 days
- [ ] Confirm the orphaned Netlify site receives no traffic
- [ ] Note in the [M9](../../product/milestones/M9.md) milestone that the cutover is complete
- [ ] Update this runbook with anything that was wrong in it

---

## Rolling back the cutover

The TTL is 300, so reverting is fast.

1. At the registrar, restore the records from `dns-before-cutover.txt`
2. Wait 5–10 minutes
3. Verify:
   ```bash
   dig +short www.goldseats.app   # back to stellar-hotteok-63503a.netlify.app
   curl -fsS https://www.goldseats.app/ | grep -q 'GoldSeats'
   ```

**Important caveat:** rolling back to the orphaned Netlify site works only while that site still
exists. We don't control it, so we cannot guarantee it stays up. **Treat rollback as available
but not dependable**, and prefer fixing forward on the new host.

---

## DNS emergency — not a planned cutover

If `goldseats.app` is down and you're not mid-cutover:

### 1. Is it DNS?

```bash
dig +short goldseats.app
dig +short @1.1.1.1 goldseats.app
dig +trace goldseats.app | tail -20
```

- **No answer at all** → the DNS record is missing, or the nameservers changed. Check the
  registrar immediately, including whether the domain is still registered and paid.
- **Answer, but wrong target** → someone changed a record. Check the registrar audit log.
- **Correct answer, site still down** → not DNS. → [`api-rollback.md`](api-rollback.md) or the
  host's status page.

### 2. Is it TLS?

```bash
echo | openssl s_client -servername goldseats.app -connect goldseats.app:443 2>/dev/null \
  | openssl x509 -noout -dates
```

Expired certificate → trigger renewal in the host dashboard. This should be automatic; if it
isn't, that's a finding for the review.

### 3. Is it the registration?

```bash
whois goldseats.app | grep -iE 'expiry|expiration|status'
```

**An expired domain registration is a Sev-1 with a very simple fix and a very bad tail.** Renew
immediately and enable auto-renew. Record who holds the payment method in
[`../on-call.md`](../on-call.md)'s access checklist.

---

## Related

- [`../environments.md`](../environments.md) — DNS table and the hosting decision
- [`../../product/milestones/M9.md`](../../product/milestones/M9.md) — cutover as a launch gate
- [`../incident-response.md`](../incident-response.md) — severities and comms
- [`api-rollback.md`](api-rollback.md) — when the app, not DNS, is broken
