# Message for the builder session: add the HOST Egypt BETA page

Copy everything below this line and paste it into the builder session.

---

Hi. Please add a **separate BETA page** to the repository `layer0data/host-egypt`. It's the same app with security fixes, so we can test it **without touching the live app**.

**Rules**
- Do **NOT** change `HOST_Egypt.html`, `host-egypt-settings.js` or `host-egypt-sw.js`. They are the live app.
- Only **two new files** get added: `HOST_Egypt_beta.html` and `host-egypt-beta-settings.js`.

**What the beta is**
- A copy of `HOST_Egypt.html` with these security fixes:
  - escaping that also covers `'`
  - a new `okUrl()`/`safeUrl()` that blocks `javascript:` and other unsafe links in every `href`/`src`
  - date and label helpers (`fmtD`, `fmtDS`, `fmtDL`, `fmtMonth`, `lab`) that escape their fallback text
  - every `data-…="${x.id}"` attribute escaped
  - the public-form and family fields (referrer, experience years, feedback scores, legacy values, observer names, reason/status fallbacks, the incident `ack_by` name) escaped
  - the media mime type and base64 body checked before use
- It loads `host-egypt-beta-settings.js`, which sets `window.HOST_EGYPT_BETA = true` and uses the same Google Sheet address as the live app.
- It keeps its own storage on the phone (prefix `hostegy-beta-`) and shows an orange "BETA" badge.
- It registers the same `host-egypt-sw.js` (unchanged). That service worker ignores the beta files, so the live app's offline copy is not affected.
- It was tested in headless Chromium on v4.6 (commit `68bed69`):
  - all owner (16), coach (7) and parent (4) screens render with no errors
  - a malicious "Recommended by" value from the public form ran a script on the live page's owner Growth screen, but not on the beta

**Steps**

1. Go to the repository folder and get the latest `main`:
   ```bash
   git fetch origin main && git checkout main && git pull origin main
   ```
   (If you work on your own branch, create it from the latest `main`.)

2. Create a file named `build_beta.py` in the repository folder with **exactly** this content:

```python
#!/usr/bin/env python3
"""HOST Egypt: build the BETA page (Security Phase 0) from the live page.

Run from the repository folder:   python3 build_beta.py
It READS HOST_Egypt.html and host-egypt-settings.js (never changes them) and WRITES two new files:
  HOST_Egypt_beta.html         the same app with the security fixes, its own storage on the phone, a BETA badge
  host-egypt-beta-settings.js  points the beta at the same Google Sheet as the live app
If HOST_Egypt.html has changed in a way the script does not expect, it stops with "STOPPED:" and writes nothing.
"""
import re, sys, os
SRC, OUT, SET_SRC, SET_OUT = 'HOST_Egypt.html', 'HOST_Egypt_beta.html', 'host-egypt-settings.js', 'host-egypt-beta-settings.js'
for f in (SRC, SET_SRC):
    if not os.path.exists(f): sys.exit(f'STOPPED: {f} not found. Run this inside the host-egypt repository folder.')
s = open(SRC, encoding='utf-8').read()
def rep(old, new, count=1):
    global s
    c = s.count(old)
    if c != count: sys.exit(f'STOPPED: expected {count} match(es), found {c}, for: {old[:90]!r}. Nothing was written.')
    s = s.replace(old, new)

# 1. esc also covers single quotes; add safeUrl
rep("""const esc = s => String(s == null ? '' : s).replace(/[&<>"]/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;' }[c]));""",
"""const esc = s => String(s == null ? '' : s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
/* v4.7 security: an address put in a link or a picture is used only when it is a web, phone or mail address, or an in-app picture / video;
   anything else (javascript:, data:text/html…) becomes '#'. The result is always escaped. */
const safeUrl = u => {
  const v = String(u == null ? '' : u).trim(); if (!v) return '';
  const m = /^([a-z][a-z0-9+.\\-]*):/i.exec(v.replace(/[\\u0000-\\u0020\\u007f]/g, ''));
  if (m) { const k = m[1].toLowerCase(); if (!(k === 'http' || k === 'https' || k === 'mailto' || k === 'tel' || k === 'blob' || (k === 'data' && /^data:(image|video)\\/[a-z0-9.+\\-]+;base64,/i.test(v)))) return '#'; }
  return esc(v);
};""")

# 2. formatters / labels never hand back raw text
for f in ['fmtD', 'fmtDS', 'fmtDL']:
    old = f"const {f} = s => {{ const d = parseD(s); if (!d) return s || '';"
    rep(old, f"const {f} = s => {{ const d = parseD(s); if (!d) return esc(s || '');")
rep("return d ? d.toLocaleDateString(LOC(), { month: 'long', year: 'numeric' }) : k };", "return d ? d.toLocaleDateString(LOC(), { month: 'long', year: 'numeric' }) : esc(k) };")
rep("return o ? L(o[1], o[2]) : (id || '') };", "return o ? L(o[1], o[2]) : esc(id || '') };")

# 3. links and media go through safeUrl
c1 = s.count('href="${esc('); s = s.replace('href="${esc(', 'href="${safeUrl(')
c2 = s.count('src="${esc('); s = s.replace('src="${esc(', 'src="${safeUrl(')
rep('href="${v45Esc(f.link)}"', 'href="${safeUrl(f.link)}"')
rep('src="${th}"', 'src="${safeUrl(th)}"', 2)
rep('src="${f.url}"', 'src="${safeUrl(f.url)}"', 2)
rep('href="${f.url}" download="${fname}"', 'href="${safeUrl(f.url)}" download="${esc(fname)}"')
rep("{ url: 'data:' + (j.mime || 'image/jpeg') + ';base64,' + j.data, mime: j.mime };",
    "{ url: 'data:' + (/^(image|video)\\/[a-z0-9.+\\-]+$/i.test(j.mime || '') ? j.mime : 'image/jpeg') + ';base64,' + String(j.data || '').replace(/[^A-Za-z0-9+\\/=]/g, ''), mime: j.mime };")
rep("img.src = j.thumbs[id];", "img.src = safeUrl(j.thumbs[id]).replace(/&amp;/g, '&');")

# 4. ids in attributes are always escaped (some ids are chosen on the phone, e.g. by a family)
s, c3 = re.subn(r'="\$\{([A-Za-z_]+\.(?:id|pid|sid|player_id|session_id|key))\}"', r'="${esc(\1)}"', s)

# 5. single spots fed by the public form or by families
rep('data-tpl="thanks_referral" data-pid="${k}"', 'data-tpl="thanks_referral" data-pid="${esc(k)}"')
rep("(x.experience_years ? ' (' + x.experience_years + L(' y', ' سنة') + ')' : '')", "(x.experience_years ? ' (' + esc(x.experience_years) + L(' y', ' سنة') + ')' : '')")
rep("'value'].map(k => f[k] || '–').join(' · ')", "'value'].map(k => esc(f[k] || '–')).join(' · ')")
rep("did.measured.observers.join(', ') || '–'", "esc(did.measured.observers.join(', ')) || '–'")
rep("did.measured.observers.join('، ') || '–'", "esc(did.measured.observers.join('، ')) || '–'")
rep("L(...(REASON[r] || [r, r]))", "L(...(REASON[r] || [esc(r), esc(r)]))", 2)
rep("L(...(JSTAT[j.status] || [j.status, j.status]))", "L(...(JSTAT[j.status] || [esc(j.status), esc(j.status)]))")
c4 = s.count('${a.ack_by}'); s = s.replace('${a.ack_by}', '${esc(a.ack_by)}')

# 6. one address checker without escaping (for DOM properties), the escaped version for HTML
rep("""const safeUrl = u => {
  const v = String(u == null ? '' : u).trim(); if (!v) return '';
  const m = /^([a-z][a-z0-9+.\\-]*):/i.exec(v.replace(/[\\u0000-\\u0020\\u007f]/g, ''));
  if (m) { const k = m[1].toLowerCase(); if (!(k === 'http' || k === 'https' || k === 'mailto' || k === 'tel' || k === 'blob' || (k === 'data' && /^data:(image|video)\\/[a-z0-9.+\\-]+;base64,/i.test(v)))) return '#'; }
  return esc(v);
};""", """const okUrl = u => {
  const v = String(u == null ? '' : u).trim(); if (!v) return '';
  const m = /^([a-z][a-z0-9+.\\-]*):/i.exec(v.replace(/[\\u0000-\\u0020\\u007f]/g, ''));
  if (m) { const k = m[1].toLowerCase(); if (!(k === 'http' || k === 'https' || k === 'mailto' || k === 'tel' || k === 'blob' || (k === 'data' && /^data:(image|video)\\/[a-z0-9.+\\-]+;base64,/i.test(v)))) return '#'; }
  return v;
};
const safeUrl = u => esc(okUrl(u));""")
rep("img.src = safeUrl(j.thumbs[id]).replace(/&amp;/g, '&');", "img.src = okUrl(j.thumbs[id]);")
rep("${x.l.entries.map(e => e.raw).join(' ')}", "${x.l.entries.map(e => esc(e.raw)).join(' ')}")

# 7. the BETA page: its own settings file, its own storage on the phone, a badge
rep('<script src="host-egypt-settings.js"></script>', '<script src="host-egypt-beta-settings.js"></script>')
rep("const APP_PFX = window.HOST_EGYPT_TEST ? 'hostegy-test-' : 'hostegy-';",
    "/* v4.7 BETA: the beta page (HOST_Egypt_beta.html) keeps its own login, outbox and copy on the phone too, so it never touches the live app's */\nconst APP_PFX = window.HOST_EGYPT_TEST ? 'hostegy-test-' : window.HOST_EGYPT_BETA ? 'hostegy-beta-' : 'hostegy-';")
rep("if (window.HOST_EGYPT_TEST) { try { document.title = 'TEST · ' + document.title; document.body.classList.add('is-test') } catch (e) { } }",
    "if (window.HOST_EGYPT_TEST) { try { document.title = 'TEST · ' + document.title; document.body.classList.add('is-test') } catch (e) { } }\nelse if (window.HOST_EGYPT_BETA) { try { document.title = 'BETA · ' + document.title; document.body.classList.add('is-beta') } catch (e) { } }")
rep('body.is-test::before{', 'body.is-beta::before{content:"BETA · v4.7 security";position:fixed;top:0;left:50%;transform:translateX(-50%);z-index:99;background:#B45309;color:#fff;font:700 11px/1 system-ui;padding:4px 10px;border-radius:0 0 8px 8px;pointer-events:none}\nbody.is-test::before{')

m = re.search(r"HOST_EGYPT_ENDPOINT\s*=\s*'([^']*)'", open(SET_SRC, encoding='utf-8').read())
if not m: sys.exit('STOPPED: could not read the Google Sheet address from host-egypt-settings.js. Nothing was written.')
open(OUT, 'w', encoding='utf-8').write(s)
open(SET_OUT, 'w', encoding='utf-8').write(
    "// HOST Egypt BETA (HOST_Egypt_beta.html): the new, more secure version, tested here before it replaces the live page.\n"
    "// It uses the SAME Google Sheet as the live app (the beta only changes how the page shows data, not what it saves).\n"
    "// To test on a separate copy of the sheet instead, paste that copy's Apps Script web app address between the quotes.\n"
    "window.HOST_EGYPT_BETA = true;\n"
    f"window.HOST_EGYPT_ENDPOINT = '{m.group(1)}';\n")
print(f'OK: wrote {OUT} ({len(s):,} bytes) and {SET_OUT}. {SRC} and {SET_SRC} were not changed.')
```

3. Run it:
   ```bash
   python3 build_beta.py
   ```
   - If it prints `OK: wrote HOST_Egypt_beta.html ...`, go to step 4.
   - If it prints `STOPPED: ...`, then `HOST_Egypt.html` changed since v4.6 and nothing was written. In that case, apply the same fixes by hand to a copy named `HOST_Egypt_beta.html`. The comments in the script list every change (sections 1–7). Then go to step 4.

4. Check it:
   ```bash
   node -e "const h=require('fs').readFileSync('HOST_Egypt_beta.html','utf8');new Function(h.split('<script>')[1].split('</script>')[0]);console.log('JS OK')"
   ```
   ```bash
   git status --short
   ```
   `git status` must list **only** `HOST_Egypt_beta.html`, `host-egypt-beta-settings.js` and `build_beta.py`, with **no** change to `HOST_Egypt.html`. Open `HOST_Egypt_beta.html` in a browser: it must show the login page with an orange "BETA" badge.

5. Delete the helper script so it isn't published, then commit only the two new files:
   ```bash
   rm build_beta.py
   ```
   ```bash
   git add HOST_Egypt_beta.html host-egypt-beta-settings.js && git commit -m "Add BETA page with Security Phase 0 fixes (live page untouched)"
   ```

6. Push, and get it onto `main`. If you're on a branch, open a pull request and merge it once I approve. GitHub Pages only publishes `main`. The beta will then open at:
   `https://layer0data.github.io/host-egypt/HOST_Egypt_beta.html`

7. Tell me the link and confirm that the live page `HOST_Egypt.html` didn't change.

**Later, when the beta has been stable for 1–2 weeks**
- Copy the beta over the live page and change its `<script src="host-egypt-beta-settings.js">` back to `host-egypt-settings.js`.
- Delete the two beta files.
- Do this only when I ask.
