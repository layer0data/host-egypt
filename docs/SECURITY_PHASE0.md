# HOST Egypt · Security Phase 0 (free tools only)

Everything below costs **nothing**. Items marked ✅ are already done in this branch; the rest need your Google / GitHub account, so only you can do them.

## The live app is not touched: a separate BETA page

- **Live app (unchanged):** `HOST_Egypt.html` + `host-egypt-settings.js`. Everyone keeps using it as today.
- **New BETA:** `HOST_Egypt_beta.html` + `host-egypt-beta-settings.js`, opened at `…/HOST_Egypt_beta.html`.
  - An orange "BETA" badge sits at the top, and the tab title starts with "BETA".
  - It keeps its **own** login, outbox and offline copy on the phone, so it never mixes with the live app.
  - It talks to the **same Google Sheet**. The beta only changes how the page *shows* data, not what it saves. To test on a copy of the sheet instead, change the address in `host-egypt-beta-settings.js`.
- **When the beta is stable** (a week or two with the owner, a coach and a parent using it), replace the content of `HOST_Egypt.html` with the beta, keep `<script src="host-egypt-settings.js">`, and delete the two beta files.

## ✅ 1. Page hardening (done, in `HOST_Egypt_beta.html`)

- `esc()` now also escapes `'`.
- New `safeUrl()` / `okUrl()`: every link and picture address (maps, videos, photos, sources, logos, downloads) only allows `https:`, `http:`, `tel:`, `mailto:`, `blob:` and in-app `data:image/…` / `data:video/…`. A `javascript:` link becomes `#`.
- Date and label helpers (`fmtD`, `fmtDS`, `fmtDL`, `fmtMonth`, `lab`) no longer hand back raw text when a value is not a real date / known label.
- All 275 `data-…="${x.id}"` style attributes are escaped (some ids are chosen on the phone, e.g. by a family).
- Text coming from the public registration form, families and imports is escaped where it was not (referrer, experience years, family feedback scores, old-record values, observer names, reason/status fallbacks, coach name on incident cards).
- Media from the server: the type must be `image/*` or `video/*`; the file body is cleaned to base64 characters.

Tested in a headless browser: owner (16 screens), coach (7) and parent (4) all render with no errors; with a malicious "Recommended by" value the old page ran the injected script, the new page does not.

## 2. Put the Apps Script backend under version control (free)

The server is the real security boundary and today it is not saved anywhere but Google. Use **clasp** (Google's free command-line tool) and a **private** GitHub repository (private repos are free):

1. On GitHub: **New repository** → name `host-egypt-backend` → **Private** → Create.
2. In script.google.com open the project → **Project Settings** → copy the **Script ID**.
3. Turn on the Apps Script API once: <https://script.google.com/home/usersettings>.
4. On any computer with Node.js (free):

```bash
npm install -g @google/clasp
```

```bash
clasp login
```

```bash
clasp clone YOUR_SCRIPT_ID --rootDir host-egypt-backend
```

```bash
cd host-egypt-backend && git init && git add . && git commit -m "Apps Script backend" && git branch -M main && git remote add origin https://github.com/layer0data/host-egypt-backend.git && git push -u origin main
```

No computer? Copy each `.gs` file from the Apps Script editor into the new repository with GitHub's **Add file → Create new file** in the browser.

Then add that repository to a Claude session so the server-side items in section 4 can be fixed.

## 3. Should this repository be private?

Be aware: a web page is always readable by anyone who opens it, so a private repository does **not** hide the app code or the Apps Script address from someone who visits the site. It only hides the history and the files that are not published.

- **GitHub Pages from a private repository needs a paid plan.** Do not make this repository private while it is served by GitHub Pages, or the site goes offline.
- Free option if you want it private anyway: **Cloudflare Pages** (free, works with private repositories, gives a free `your-name.pages.dev` address):
  1. <https://dash.cloudflare.com> → sign up free → **Workers & Pages → Create → Pages → Connect to Git** → pick this repository.
  2. Build command: *(empty)*. Output directory: `/`. Deploy.
  3. Open `https://<name>.pages.dev/HOST_Egypt.html`, check login works, share the new link with staff and families.
  4. Only then make this GitHub repository private and switch off GitHub Pages.
- Never put passwords, PIN lists, API keys or the backend code into this public repository.

Recommendation: keep this page repository as it is for now, put the backend in the **private** repository (section 2), and spend the effort on section 4. That is where the real protection is.

## 4. Server-side fixes (need the backend code; all free)

| # | Fix | Why |
|---|-----|-----|
| 1 | Staff log in with **phone (or username) + PIN**, and PINs of **6 digits** | Today a 4-digit PIN alone can match *any* staff member |
| 2 | **Rate-limit on the server** per account *and* across all accounts (e.g. 5 wrong tries → 15 min lock; 50 wrong tries per hour in total → lock logins for the academy and alert the owner) | The device id the phone sends is made up by the phone and cannot be trusted |
| 3 | **Tokens expire** (e.g. staff 12 h of inactivity, parents 30 days) and are checked on every call | A lost phone stays logged in forever today |
| 4 | `load` sends each role **only** what it may see (no fees/phones/medical to coaches & evaluators; no medical to marketing; no private notes to families) | The page hides these, but the data still reaches the phone |
| 5 | Every write checks the **role** and **validates** fields (dates are `YYYY-MM-DD`, ids are server-made, links start with `https://`, numbers are numbers) | Stops bad data at the source |
| 6 | `bulk` / `evrecord` answer **per record** (saved / refused) | Today one refused record deletes the whole batch on the phone |
| 7 | Generate PINs with `Utilities.getUuid()` / secure random, store only hashes with a salt | Predictable PINs are guessable |

Free alternatives to paid SMS/WhatsApp codes: longer PINs plus the server rate limits above give most of the protection at zero cost.
