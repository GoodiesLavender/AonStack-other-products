# Aon Technology Solutions — standalone "Technology Solutions" page

This is a **separate website** with its own Vercel project. It has no code ties to the main Aon website.

- **The page:** it uses the Aon Stack design, has three category sections (AI, SaaS, Telecom) and three inquiry forms.
- **The email function:** a Vercel Function emails each inquiry to **info@aontechnology.com**.
- **Suggested address:** `solutions.aontechnology.com`. Use any subdomain you like; the domain's DNS stays on Cloudflare.

When you're ready to connect it to the main site, add a link from the main site's navigation to this page's address. Nothing else is needed.

---

## What's in this folder

```
public/                        ← everything visitors can see (Vercel serves only this folder)
  index.html                   the page
  solutions-config.js          ← products, services, form fields, main-site + privacy links (edit this one)
  inquiry-validation.js        form rules, shared by the page and the server
  solutions-page.js            page behaviour (no secrets)
  favicon.svg
api/
  technology-inquiry.js        Vercel Function → POST /api/technology-inquiry (never public)
vercel.json                    tells Vercel: no build step, serve public/, security headers
package.json                   "type": "module" (required by Vercel for this code)
.env.example                   list of settings
tests/  scripts/               local testing only (excluded from deployment by .vercelignore)
```

**Links back to the main site.** Two links point at the main site: the logo and "Aon Stack Home" go to `https://aontechnology.com`, and "Privacy Policy" goes to `https://aontechnology.com/privacy-policy`. Change either at the top of `public/solutions-config.js`. Set `mainSiteUrl: ''` to keep every link on this page.

---

## Step 1: Set up Resend (sends the emails)

1. Create a free account at resend.com and go to **Domains → Add Domain** → `aontechnology.com`.
2. Resend shows 3–4 DNS records: a DKIM `TXT` at `resend._domainkey`, plus an `MX` and a `TXT` on a `send` subdomain.
   - In **Cloudflare → aontechnology.com → DNS → Records**, add each one exactly as shown.
   - These are separate from your existing email records, so **your current email is not affected**.
3. Click **Verify** in Resend. It usually takes a few minutes.
4. Go to **API Keys → Create API key** with sending access, and copy it. It starts with `re_`.

## Step 2: Deploy to Vercel

**Option A: GitHub (recommended, redeploys automatically whenever this repository changes)**

This code lives in **GoodiesLavender/AonStack-other-products**.

1. In Vercel, go to **Add New → Project** and **Import** `AonStack-other-products`.
   - If it isn't listed, click **Adjust GitHub App Permissions** and give Vercel access to it.
2. Create it as a **new project**. Don't connect this repository to your existing Aon website project, or it would replace your homepage.
3. Framework Preset: **Other**. Leave Build and Output settings empty, because `vercel.json` handles them.
4. Before clicking **Deploy**, open **Environment Variables** and add the settings from Step 3.

The repository contains no passwords or keys. Those live only in Vercel's Environment Variables, so it's safe for the repository to be public.

**Option B: Vercel CLI (no GitHub)**
In a terminal, inside this folder:

```bash
npx vercel            # log in, create the project (answer "Other" if asked)
npx vercel --prod     # deploy to production
```

Then add the settings from Step 3 in the dashboard and run `npx vercel --prod` again.

## Step 3: Add the settings in Vercel

**Where:** Vercel → project → **Settings → Environment Variables**. Add or change variables, then **redeploy**, because they only apply to new deployments.

| Variable | Environments | Value |
|---|---|---|
| `RESEND_API_KEY` | Production, Preview | the `re_…` key from Step 1 |
| `INQUIRY_FROM_EMAIL` | Production, Preview | `Aon Technology Solutions Website <website@aontechnology.com>` |
| `ALLOWED_ORIGINS` | **Production only** | `https://solutions.aontechnology.com`, or whatever address you choose in Step 4 |
| `INQUIRY_TO_EMAIL` | **Preview only** (optional) | Your own address, so tests on preview links don't go to info@. Production defaults to info@aontechnology.com. |
| `BEHIND_CLOUDFLARE_PROXY` | optional | Leave unset. Set to `true` **only** if you turn on Cloudflare's orange-cloud proxy (not recommended). |
| `INQUIRY_TIMEZONE`, `RATE_LIMIT_MAX`, `RATE_LIMIT_WINDOW_SECONDS` | optional | Defaults: `America/Chicago`, `6`, `600` |
| `UPSTASH_REDIS_REST_URL` / `_TOKEN` | optional | Makes the rate limit shared across every function instance. Upstash from the Vercel Marketplace works too; its `KV_REST_API_URL` / `KV_REST_API_TOKEN` names are accepted. |

Never add `EMAIL_DRY_RUN=true` in Vercel. It stops emails from being sent.

## Step 4: Point your Cloudflare domain at Vercel

1. In **Vercel → project → Settings → Domains**, add `solutions.aontechnology.com` (or your choice). Vercel shows the DNS record it needs, usually a `CNAME` to `cname.vercel-dns.com`.
2. In **Cloudflare → DNS → Records → Add record**:
   - Type **CNAME**
   - Name `solutions`
   - Target: the value Vercel showed
   - **Proxy status: DNS only (grey cloud)**
3. Wait for Vercel to show **Valid Configuration**. It issues the HTTPS certificate automatically.

**Why grey cloud:** Vercel doesn't recommend putting Cloudflare's proxy in front of it. The proxy causes certificate and "Invalid Configuration" problems and hides visitors' real IPs from the spam protection. Cloudflare still manages your DNS either way.

---

## How a submission works

1. The visitor clicks a card, for example **Devs.ai → Request Information**. The right form opens with it already ticked.
2. The browser checks the required fields, email, phone, ZIP and state.
3. The browser sends the answers to `/api/technology-inquiry` on this site.
4. The Vercel Function then:
   - accepts only JSON posts under 32 KB from this site's own address
   - rate-limits each visitor (6 per 10 minutes, hashed IP)
   - drops bots caught by a hidden honeypot field or by finishing in under 3 seconds
   - re-validates and cleans every answer
   - emails the organised inquiry through Resend. The key exists only in Vercel.
5. The email arrives at **info@aontechnology.com** with **Reply-To set to the customer**. Subjects:
   - `[Aon Website Lead - AI Solutions] Company Name - Customer Name`
   - `[Aon Website Lead - SaaS] Company Name - Customer Name`
   - `[Aon Website Lead - Telecom] Company Name - Customer Name`
6. The visitor sees *"Thank you. We've received your request."* with **Return to Technology Solutions**.

**Troubleshooting:** open Vercel → project → **Logs** and filter to `/api/technology-inquiry`.
- `[inquiry] sent type=…` means the email was sent.
- Any other message says what's missing, for example the API key or an unverified domain.

---

## Testing

**On your computer** (optional; needs Node 18+). Nothing is emailed:

```bash
npm test        # 22 server tests (validation, spam, rate limit, email format)
npm run local   # opens http://localhost:8788/ — inquiries are saved to .outbox/ instead of sent
```

**On a Vercel Preview link** (with `INQUIRY_TO_EMAIL` set to your address for Preview), and once more on the live address:

- **AI:**
  1. Click **Devs.ai → Request Information**. Devs.ai should be pre-ticked.
  2. Submit the form empty. You should see 5 required-field errors.
  3. Complete it and submit. The subject should be `[Aon Website Lead - AI Solutions] …`.
- **SaaS:**
  1. Click **Google Workspace → Request Information**. It should be pre-ticked.
  2. Submit. The subject should be `[Aon Website Lead - SaaS] …`.
- **Telecom:**
  1. Click **Fiber Broadband → Check Availability**. It should be pre-ticked.
  2. Tick **Other**. The "Tell us what you're looking for" box should appear.
  3. Choose **Custom** bandwidth. A description box should appear.
  4. Click **Add Another Location**.
  5. Submit without an address. You should see address errors.
  6. Complete it and submit. The subject should be `[Aon Website Lead - Telecom] …`.

**Security check:** this command should return `403`:

```bash
curl -i -X POST https://solutions.aontechnology.com/api/technology-inquiry \
  -H "Content-Type: application/json" -H "Origin: https://example.com" -d '{}'
```

**Direct links for ads and emails** open a form with an item pre-ticked, for example `https://solutions.aontechnology.com/?inquiry=telecom&item=fiber-broadband`.

---

## Adding products later

Open `public/solutions-config.js` and copy one entry in `CATALOG`. Commit it, or redeploy if you used the CLI.

```js
{ id: 'zoho-workplace', category: 'saas', name: 'Zoho Workplace', kind: 'product', inquiryType: 'saas',
  description: 'Short, neutral description — no pricing or performance claims.' },
```

The card, the form checkbox, click-to-preselect and the server validation all update together.

## Content rules built in

- No vendor logos, ratings, pricing, availability promises, partnership claims or testimonials.
- Descriptions are neutral and original.
- Your consent text sits under every submit button.
- There's no marketing opt-in.
- Each form says it's an information request, not a purchase or service order.
