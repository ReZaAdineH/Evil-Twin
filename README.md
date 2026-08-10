# Evil-Twin
Someone else owns a domain that looks like yours ?
# Evil Twin

Type a domain. See every registered lookalike — typos, unicode homoglyphs, TLD
swaps, hyphen tricks — which ones resolve, and which ones have MX records and
are therefore configured to send email as your brand.

dnstwist for people who don't have a terminal.

---

## Deploy by drag and drop — no terminal

`evil-twin-upload.zip` in this folder is ready to upload as-is.

1. Cloudflare dashboard → **Workers & Pages** → **Create application** →
   **Pages** → **Upload assets**
2. Name the project, drag in `evil-twin-upload.zip`, deploy

That's the whole app live on `<project>.pages.dev`. Scanning works
immediately — DNS runs in the visitor's browser, so nothing else is needed.

The zip contains a `_worker.js`, which Cloudflare Pages recognises as
[advanced mode](https://developers.cloudflare.com/pages/functions/advanced-mode/)
and runs for every request. That's the same file as `src/worker.js`; it needs
no build step because it has no imports.

**To turn on the Share button**, add storage after that first deploy — still
no terminal:

1. **Storage & Databases** → **KV** → **Create namespace**, call it
   `evil-twin`
2. Back in your Pages project → **Settings** → **Bindings** → **Add** →
   **KV namespace**
3. Variable name `EVILTWIN_KV`, select the namespace you just made
4. Redeploy (Deployments → ⋯ → Retry deployment) — bindings only take effect
   on a new deployment

Until you do this, Share shows "Sharing is not configured on this deployment"
and everything else works fine. Screenshots work either way.

Rebuild the zip after any edit with `npm run build`.

---

## Deploy from the CLI (about 3 minutes)

Gives you `wrangler dev` for local testing, secrets, and versioned rollbacks.

```bash
npm install
npx wrangler login

# create the KV namespace that backs shared result links
npx wrangler kv namespace create EVILTWIN_KV
# paste the printed id into wrangler.toml -> [[kv_namespaces]] id = "..."

npm run deploy
```

You'll get a `https://evil-twin.<your-subdomain>.workers.dev` URL. Add a custom
domain in the Cloudflare dashboard under Workers & Pages → your worker →
Settings → Domains & Routes.

Local development:

```bash
npm run dev     # http://localhost:8787
npm test        # offline test suite, no network needed
```

### Optional

```bash
npx wrangler secret put URLSCAN_API_KEY
```

Raises the rate limit on the screenshot proxy. Screenshots work without it —
urlscan.io's search API is public — you'll just hit their anonymous limits
sooner.

---

## How it works

```
public/engine.js    permutation generation + RFC 3492 punycode encoder
public/scanner.js   DNS-over-HTTPS resolution, risk classification, scoring
public/app.js       UI: live-streaming results, filters, CSV, share
public/index.html   the app shell and styles
public/report.css   styles for server-rendered share pages
src/worker.js       share links, /r/:id pages, urlscan proxy, asset serving
build.mjs           packages dist/ + the drag-and-drop zip (no dependencies)
test/               offline tests with a stubbed DNS zone
```

The same `src/worker.js` runs unmodified on both Workers and Pages, because
both give it an `env.ASSETS` binding and both use module-worker syntax. The
only difference is the filename Pages expects.

**The scan runs in the visitor's browser.** DNS lookups go straight from the
browser to Cloudflare's and Google's DNS-over-HTTPS endpoints, round-robined,
with a 24-way concurrency pool. This means:

- the Worker does no DNS work, so a scan costs you nothing and can't be
  rate-limited into the ground by one popular link
- you never see what domain a visitor scanned unless they press Share
- a typical scan is ~1,200 candidates and finishes in 10–30 seconds

The Worker only handles three routes: `POST /api/share`, `GET /r/:id`, and
`GET /api/screenshot`. Everything else is a static asset.

### Registration detection

A DNS `A` query returning `NXDOMAIN` means nobody owns the name. `NOERROR`
means the name exists in DNS. Only for those do we follow up with `MX` and
`NS` — so a scan averages a little over one lookup per candidate rather than
three. A `NOERROR` with no A, MX or NS records at all is an empty non-terminal
and is *not* counted as a hit; crying wolf is the fastest way to lose a user's
trust.

### Risk levels

| Level | Meaning |
|---|---|
| **Critical** | Has MX records. Somebody deliberately configured this domain to send and receive mail. This is the invoice-fraud setup, and it works whether or not there's a website. |
| **High** | Registered and serving a live website. |
| **Medium** | Registered but not in use. Parked, or held in reserve. |
| **Low** | Resolves to your own IPs or shares your mail servers — almost certainly a defensive registration you already own. |
| **Available** | Not registered. |

The **deception score** is separate from risk: it estimates how likely a human
is to be fooled by the string itself. Unicode homoglyphs score highest because
`xn--` domains render identically to the real thing in most interfaces. Bit
flips score lowest — they matter to hardware, not to eyes.

### Permutation types

`homoglyph` · `typo` (omission, repetition, transposition, keyboard
replacement, keyboard insertion, character addition) · `tld-swap` across ~65
TLDs · `brand-word` (secure-, -login, invoice-, etc.) · `hyphenation` ·
`vowel-swap` · `subdomain` · `bitsquatting`

Keyboard-adjacency fuzzers cover QWERTY, QWERTZ and AZERTY, so European typo
patterns show up too. Bitsquatting is off by default — it's noisy and rarely
what a marketing director came for.

The punycode encoder is verified byte-for-byte against Node's IDNA
implementation in the test suite, so every `xn--` domain shown is exactly what
a registrar would produce.

---

## Honest limitations

- **DNS is not the registry.** A domain can be registered and have no DNS
  records at all; we'd report it as available. RDAP/WHOIS lookups would close
  this gap but can't be done from a browser and are heavily rate-limited.
- **A few TLDs use wildcard DNS**, which makes every name under them look
  registered. Rare since ICANN's 2003 Site Finder fallout, but not extinct.
- **Registered ≠ malicious.** Resellers, fans, parking companies and your own
  marketing team register lookalikes. The UI says this; keep it saying it.
- **No screenshot means no screenshot.** We read urlscan.io's public archive
  rather than submitting new scans, so obscure domains often have nothing.

---

## Roadmap

- RDAP registration dates and registrar names via a Worker-side proxy with
  caching (turns "registered" into "registered 6 days ago", which is a much
  louder signal)
- Scheduled re-scans with email alerts on new registrations — this is the part
  people would pay for
- Certificate Transparency search, which catches lookalikes that have a TLS
  cert but no A record yet
- Desktop builds

## Credit

Permutation logic owes a large debt to
[dnstwist](https://github.com/elceef/dnstwist) by Marcin Ulikowski. This is a
from-scratch JavaScript implementation of the same ideas, aimed at a different
user.
