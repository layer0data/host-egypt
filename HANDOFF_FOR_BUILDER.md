# Message for the builder session: add the HOST Egypt BETA page

Copy everything below this line and paste it into the builder session.

---

Hi. Please add a **separate BETA page** to the repository `layer0data/host-egypt`. It's the same app with security fixes, so we can test it **without touching the live app**.

**Rules**
- Do **NOT** change `HOST_Egypt.html`, `host-egypt-settings.js` or `host-egypt-sw.js`. They are the live app.
- Only **two new files** get added: `HOST_Egypt_beta.html` and `host-egypt-beta-settings.js`.

**What the beta is (v4.7)**
- A copy of `HOST_Egypt.html` that loads `host-egypt-beta-settings.js` (which sets `window.HOST_EGYPT_BETA = true` and uses the same Google Sheet address as the live app).
- It keeps its own storage on the phone (prefix `hostegy-beta-`) and shows an orange "BETA" badge.
- It registers the same `host-egypt-sw.js` (unchanged). That service worker ignores the beta files, so the live app's offline copy is not affected.

**Security fixes in the beta**
- Escaping also covers `'`.
- A new `okUrl()`/`safeUrl()` blocks `javascript:` and other unsafe links in every `href`/`src`.
- Date and label helpers escape their fallback text.
- Every `data-…="${x.id}"` attribute is escaped.
- Public-form and family fields are escaped.
- The media mime type and base64 body are checked before use.

**Bug fixes in the beta**
- **Outbox:** if the server refuses a batch, each record is sent again on its own, so only the bad record is dropped. After a refusal the screen reloads.
- **Minute refresh:** it no longer overwrites a save that was still on its way. The duplicate refresh timer was removed.
- **Double taps:** a form is sent once per tap. Buttons that create records (register a lead, recurring expense, uniform sale, fees, waive, bulk issue, open session, approve profile, placement) ignore a second tap while saving.
- **Family notes:** newest first, dated by `at` (the sort was inside a comment).
- **Money:**
  - "paid until" for a family counts only when every child has paid.
  - A month with any part still due is not paid.
  - A partial payment asks whether to record it as fully paid or keep the rest as still due.
  - An amount of 0 is kept as 0.
- **Arabic digits** (٠-٩, ۰-۹) in number, phone and PIN fields and at login are read as 0-9.
- **Consent:** the emergency and rules boxes are no longer ticked by default.
- **Parent survey:** the "only if you have time" questions are really optional now.
- **Child login:** a child can tick their own home fitness.
- **Time zone:** the academy's own zone (`settings.tz`, Cairo by default) is used for today, quiet hours and family timestamps. Timestamps are never in the future.
- **Messages:**
  - PIN templates are blocked in group or compose sends.
  - "sent ✓" resets for each new compose.
  - "Prepare" opens the item that was tapped.
  - The weekly cap shows what is actually enforced (0 = no limit).
- **Coach Today:** the "your rating is waiting" line counts only the coach's own ratings.
- **Reports:** a child without development consent is flagged and never bulk-issued.
- **Offline:** opening a child's file no longer pops an error box.
- **Robustness:** unknown values (note kind, level, media status, milestone, skill band, day, missing `kid_logins`) no longer blank whole screens.

**How it was tested** (headless Chromium, on v4.6, commit `68bed69`)
- All owner (16), coach (7) and parent (4) screens render with no errors.
- An XSS value from the public form ran a script on the live page, but not on the beta.
- Unit checks passed: outbox split, payments, digits, time zone, notes order, double submit, PIN block, consent flag.

**Steps**

1. Go to the repository folder and get the latest `main`:
   ```bash
   git fetch origin main && git checkout main && git pull origin main
   ```
   (If you work on your own branch, create it from the latest `main`.)

2. Create a file named `build_beta.py` in the repository folder with **exactly** this content:

```python
#!/usr/bin/env python3
"""HOST Egypt: build the BETA page (v4.7: security fixes + bug fixes) from the live page.

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

# ---------- v4.7 bug fixes · saving and sync ----------
# outbox: a refused batch is retried one record at a time, so only the record the server refuses is dropped (not the whole batch)
rep("const groups = {}; mine.forEach(e => { const g = e.item ? 'i|' + e.a + '|' + e.sid : e.rec ? 'r|' + e.kind : 'b|' + e.key; (groups[g] = groups[g] || []).push(e) });",
    "const groups = {}; mine.forEach(e => { const g = e.solo ? 's|' + e.key : e.item ? 'i|' + e.a + '|' + e.sid : e.rec ? 'r|' + e.kind : 'b|' + e.key; (groups[g] = groups[g] || []).push(e) });   // v4.7: e.solo = sent on its own after its batch was refused")
rep("""        OB.list = OB.list.filter(x => !sent.some(([k, st]) => x.key === k && x.stamp === st));   // the server refused it for a reason: show the reason, do not retry forever
        refused++; alertBox(L('Not saved: ', 'لم يُحفظ: ') + err.message); continue;""",
"""        if (g.length > 1) { g.forEach(x => { x.solo = true }); obSave(); failed = true; continue }   // v4.7: one bad record must not drop the others: each is sent again on its own
        OB.list = OB.list.filter(x => !sent.some(([k, st]) => x.key === k && x.stamp === st));   // the server refused it for a reason: show the reason, do not retry forever
        refused++; alertBox(L('Not saved: ', 'لم يُحفظ: ') + err.message); setTimeout(() => { if (S.me && !OB.busy) reload() }, 1500); continue;   // v4.7: the screen goes back to what the academy really has""")

# the minute refresh never replaces data with a copy fetched while a save was on its way
rep("function busy(n) { S.saving += n;", "function busy(n) { if (n > 0) S.wgen = (S.wgen || 0) + 1; S.saving += n; if (S.saving <= 0) LOCKS_ = {};")
rep("""  const g = OB.gen; try { const j = await api('load'); if (g !== OB.gen || OB.busy || obMine().length) return; applyLoad(j); S.loadedAt = Date.now(); rerender() } catch (e) { }
}""", """  if (S.saving > 0) return;
  const g = OB.gen, wg = S.wgen || 0; try { const j = await api('load'); if (g !== OB.gen || wg !== (S.wgen || 0) || S.saving > 0 || OB.busy || obMine().length) return; applyLoad(j); S.loadedAt = Date.now(); rerender() } catch (e) { }   // v4.7: dropped when a save happened meanwhile
}""")
rep("""setInterval(async () => {
  if (!S.me || !S.t || document.visibilityState !== 'visible' || obMine().length || OB.busy) return;
  const m = document.getElementById('modal'); if (m && !m.hidden) return;
  const ae = document.activeElement; if (ae && /INPUT|TEXTAREA|SELECT/.test(ae.tagName)) return;
  if (S.view === 'evaluate' || S.sub === 'eval' || S.view === 'settings' || S.kid || location.hash === '#register') return;
  const g = OB.gen; try { const j = await api('load'); if (g !== OB.gen || OB.busy || obMine().length) return; applyLoad(j); rerender() } catch (e) { }
}, 60000);""", """setInterval(quietReload, 60000);   // v4.7: one refresh routine (same checks as above)""")

# double taps: a form is sent once; a button that creates something is ignored while its first tap is still saving
rep("""  const f = ev.target; ev.preventDefault(); const o = formObj(f); const id = f.dataset.id;
  try {
    if (f.id === 'loginForm') return doLogin(ev);""", """  const f = ev.target; ev.preventDefault();
  if (f.__busy) return; f.__busy = true; const sbs = [...f.querySelectorAll('button[type="submit"],button:not([type])')]; sbs.forEach(b => { b.disabled = true });   // v4.7: one send per tap
  const o = formObj(f); const id = f.dataset.id;
  try {
    if (f.id === 'loginForm') return await doLogin(ev);""")
rep("""  } catch (e) { /* the reason was already shown */ }
});""", """  } catch (e) { /* the reason was already shown */ }
  finally { f.__busy = false; sbs.forEach(b => { b.disabled = false }) }
});""")
rep("const isAr = () => S.lang === 'ar';", """const isAr = () => S.lang === 'ar';
/* v4.7: buttons that create a record (a player, a fee, an expense, a sale…) ignore a second tap while the first one is still saving */
let LOCKS_ = {};
const LOCK_ACTS = ['lead-register', 'spend-recur', 'uni-sell', 'fees-make', 'pay-waive', 'rep-issue-clean', 'open-session', 'profile-approve', 'placement-move'];
document.addEventListener('click', ev => {
  const el = ev.target.closest && ev.target.closest('[data-act]'); if (!el || LOCK_ACTS.indexOf(el.dataset.act) < 0) return;
  const k = [el.dataset.act, el.dataset.id || el.dataset.pid || el.dataset.sid || '', el.dataset.date || '', el.dataset.start || '', el.dataset.v || ''].join('|');
  if (LOCKS_[k] && (S.saving > 0 || Date.now() - LOCKS_[k] < 1500)) { ev.stopImmediatePropagation(); ev.preventDefault(); return }
  LOCKS_[k] = Date.now();
}, true);""")

# ---------- v4.7 bug fixes · features ----------
# family notes: newest first (the sort was inside a comment), dated by when they were written
rep("const famNotes = pid => (D().notes || []).filter(n => n.player_id === pid && ['month_plan', 'elite_session', 'kid_msg', 'placement', 'term_confirm'].indexOf(n.type) < 0)   // v4.2: each of these has its own card.sort((a, b) => String(b.date).localeCompare(String(a.date)));",
    "const famNotes = pid => (D().notes || []).filter(n => n.player_id === pid && ['month_plan', 'elite_session', 'kid_msg', 'placement', 'term_confirm'].indexOf(n.type) < 0).sort((a, b) => String(b.at || b.date || '').localeCompare(String(a.at || a.date || '')));   // v4.2: each of the excluded types has its own card; v4.7: newest first")
rep("fmtDS(n.date)", "fmtDS(n.date || String(n.at || '').slice(0, 10))", 5)

# money: a family is "paid until" a month only when every child is; a partial payment keeps the rest as still due
rep("return kids.length && ms.length === kids.length ? ms[0] : (ms[0] || '') }", "return kids.length && ms.length === kids.length ? ms[0] : '' }   // v4.7: a child with no paid month means the family is not paid up")
rep("const by = {}; rows.forEach(x => by[x.month] = x.status);", "const by = {}; rows.forEach(x => { const cur = by[x.month]; if (!cur || cur === 'paid' || cur === 'waived') by[x.month] = x.status });   /* v4.7: a month with any part still due is not paid */")
rep("if (f.id === 'payForm') { const x = (D().payments || []).find(r => r.id === id); await save('payments', Object.assign({}, x, { amount: +o.amount || x.amount, paid_at: o.paid_at, method: o.method, ref: o.ref, status: 'paid' })); modal(''); rerender(); return }",
    """if (f.id === 'payForm') {
      const x = (D().payments || []).find(r => r.id === id); const due = +x.amount || 0; const got = String(o.amount == null ? '' : o.amount).trim() === '' ? due : +o.amount;   // v4.7: an amount of 0 is kept as 0
      if (!(got >= 0)) { toast('⚠ ' + L('Please type the amount received.', 'برجاء كتابة المبلغ المستلم.')); return }
      if (got > 0 && got < due && !confirm(L(`${num(got)} is less than the ${num(due)} due. OK = record it as fully paid. Cancel = keep the remaining ${num(due - got)} as still due.`, `${num(got)} أقل من المستحق ${num(due)}. موافق = تسجيله مدفوعًا بالكامل. إلغاء = يبقى الباقي ${num(due - got)} مستحقًا.`))) {
        await save('payments', Object.assign({}, x, { amount: got, paid_at: o.paid_at, method: o.method, ref: o.ref, status: 'paid' }));
        await save('payments', Object.assign({}, x, { id: newId('pa_'), amount: due - got, paid_at: '', method: '', ref: '', status: 'due', note: L('remaining part', 'الجزء المتبقي') }));
      } else await save('payments', Object.assign({}, x, { amount: got, paid_at: o.paid_at, method: o.method, ref: o.ref, status: 'paid' }));
      modal(''); rerender(); return
    }""")

# Arabic digits (٠-٩ and ۰-۹) typed in number, phone and PIN fields are read as 0-9
rep("function formObj(f) { const o = {}; new FormData(f).forEach((v, k) => {",
    "const latinDigits = v => String(v == null ? '' : v).replace(/[\\u0660-\\u0669]/g, c => String(c.charCodeAt(0) - 0x0660)).replace(/[\\u06F0-\\u06F9]/g, c => String(c.charCodeAt(0) - 0x06F0));   // v4.7\nfunction formObj(f) { const o = {}; const numF = {}; [...(f.elements || [])].forEach(el => { if (el.name && (/^(number|tel|password)$/.test(el.type) || /numeric|decimal|tel/.test(el.inputMode || el.getAttribute('inputmode') || ''))) numF[el.name] = 1 }); new FormData(f).forEach((v, k) => { if (numF[k] && typeof v === 'string') v = latinDigits(v);")
s = s.replace("pin: $('#lgPin').value.trim(), phone: ($('#lgPhone') || {}).value || ''", "pin: latinDigits($('#lgPin').value.trim()), phone: latinDigits(($('#lgPhone') || {}).value || '')")
rep(r'pattern="\\d{4}"', 'pattern="[0-9\\u0660-\\u0669\\u06F0-\\u06F9]{4}"')

# consent is never pre-ticked
rep("name=\"consent_emergency\" ${p.consent_emergency !== 'no' ? 'checked' : ''}", "name=\"consent_emergency\" ${p.consent_emergency === 'yes' ? 'checked' : ''}")
rep("name=\"consent_rules\" ${p.consent_rules !== 'no' ? 'checked' : ''}", "name=\"consent_rules\" ${p.consent_rules === 'yes' ? 'checked' : ''}")

# the parent survey: the extra questions really are optional
rep("${b.map((q, i) => row(q, a.length + i)).join('')}</details>", "${b.map((q, i) => row(q, a.length + i)).join('').replace(/ required(?=[ >])/g, '')}</details>")

# a child on their own login can tick their home fitness
rep("regAction('km-tick', (d, el) => { if (!isParent() && !isOwnerish()) return;", "regAction('km-tick', (d, el) => { if (!isParent() && !isOwnerish() && !(isPlayer() && P(d.pid))) return;")

# the academy's own time zone (Settings tz; Cairo when not set) for today, quiet hours and family timestamps
rep("const cairoToday = () => { try { return new Intl.DateTimeFormat('en-CA', { timeZone: 'Africa/Cairo',", "const cairoToday = () => { try { return new Intl.DateTimeFormat('en-CA', { timeZone: (typeof SET === 'function' && SET().tz) || 'Africa/Cairo',")
rep("hm = new Intl.DateTimeFormat('en-GB', { timeZone: 'Africa/Cairo', hour: '2-digit', minute: '2-digit', hour12: false }).format(new Date())", "hm = new Intl.DateTimeFormat('en-GB', { timeZone: SET().tz || 'Africa/Cairo', hour: '2-digit', minute: '2-digit', hour12: false }).format(new Date())")
rep("const famNow = () => todayStr() + new Date().toISOString().slice(10);",
    "const famNow = () => { try { const now = new Date(); const pt = {}; new Intl.DateTimeFormat('en-GB', { timeZone: SET().tz || 'Africa/Cairo', year: 'numeric', month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', second: '2-digit', hour12: false }).formatToParts(now).forEach(x => { pt[x.type] = x.value }); const h = pt.hour === '24' ? '00' : pt.hour; const off = Math.round((Date.UTC(+pt.year, +pt.month - 1, +pt.day, +h, +pt.minute, +pt.second) - now.getTime()) / 60000); const sg = off < 0 ? '-' : '+', a = Math.abs(off); return `${pt.year}-${pt.month}-${pt.day}T${h}:${pt.minute}:${pt.second}${sg}${pad(Math.floor(a / 60))}:${pad(a % 60)}` } catch (e) { return new Date().toISOString() } };   // v4.7: the academy's own date and clock, with its offset (never a time in the future)")

# messages: PIN texts never go out with an empty PIN; ticks belong to one compose; Prepare opens the item that was tapped; the cap shows what is enforced
rep("function tplMissing(id) {", "const tplPinMissing = id => /\\{pin\\}/.test(String(tplText(id, 'en') || '') + String(tplText(id, 'ar') || '')) ? L('A family PIN is stored safely and cannot be put in a group message. Open the child, Family tab, set a new PIN and send it from there.', 'الرقم السري للعائلة محفوظ بأمان ولا يمكن وضعه في رسالة جماعية. افتح ملف الطفل، تبويب العائلة، اضبط رقمًا سريًا جديدًا وأرسله من هناك.') : '';   // v4.7\nfunction tplMissing(id) {\n  if (tplPinMissing(id)) return tplPinMissing(id);")
rep("const missing = c.pids || c.lids ? '' : tplMissing(c.tpl);", "const missing = c.pids || c.lids ? tplPinMissing(c.tpl) : tplMissing(c.tpl);")
rep("function composeModal(c) {", "function composeModal(c) {\n  if (c && c.tpl) S.sentNow = {};   // v4.7: a new compose starts with no ✓")
rep("data-act=\"mc-open\" data-i=\"${i}\"", "data-act=\"mc-open\" data-i=\"${i}\" data-k=\"${esc(mcKey(it))}\"")
rep("regAction('mc-open', d => { const it = msgCentreItems()[+d.i]; if (!it) return;", "const mcKey = it => [it.tpl, it.aud || '', (it.pids || []).join(','), (it.lids || []).join(',')].join('|');   // v4.7\nregAction('mc-open', d => { const all = msgCentreItems(); const it = d.k ? all.find(x => mcKey(x) === d.k) : all[+d.i]; if (!it) { toast(L('This item changed. Please look again.', 'تغيّر هذا البند. برجاء المراجعة مرة أخرى.')); return }")
rep("value=\"${esc(s.weekly_msg_cap == null ? 3 : s.weekly_msg_cap)}\"", "value=\"${esc(s.weekly_msg_cap == null ? 0 : s.weekly_msg_cap)}\"")

# coaches: "your rating is waiting" counts only your own ratings
rep("const myTaps = sid => (D().obs || []).some(o => o.session_id === sid && o.source === 'coach');", "const myTaps = sid => (D().obs || []).some(o => o.session_id === sid && o.source === 'coach' && (!o.rater || o.rater === S.me.id));")

# reports: a child without development consent is never issued in bulk
rep("  if (!r.snapshot) return [L('not prepared yet', 'لم يُجهَّز بعد')];", "  if ((P(r.player_id) || {}).consent_dev !== 'yes') f.push(L('no family consent', 'بلا موافقة العائلة'));   // v4.7\n  if (!r.snapshot) return f.concat([L('not prepared yet', 'لم يُجهَّز بعد')]);")

# opening a child's file offline no longer pops up an error (audit calls are quiet)
rep("act('view', { player_id: p.id }, true).catch(() => { })", "api('view', { player_id: p.id }).catch(() => { })")
rep("act('kid_seen', { player_id: p.id }, true).then(", "api('kid_seen', { player_id: p.id }).then(")

# a value the page does not know never blanks a whole screen
rep("L(...NOTEK[n.kind])", "L(...(NOTEK[n.kind] || [esc(n.kind), esc(n.kind)]))")
rep("esc(L(IX_LVL[l][0], IX_LVL[l][1]))", "esc(L(...((IX_LVL[l] || [l, l]).slice(0, 2))))")
rep("L(MD_ST[m.status][0], MD_ST[m.status][1])", "L(...((MD_ST[m.status] || [esc(m.status), esc(m.status)]).slice(0, 2)))")
rep("L(...MILESTONE[m.id])", "L(...(MILESTONE[m.id] || [esc(m.id), esc(m.id)]))")
rep("const x = b.can[bBand(p)][lv - 1]; return L(x[0], x[1]) }", "const x = ((b.can || {})[bBand(p)] || [])[lv - 1]; return x ? L(x[0], x[1]) : '' }")
rep("const dayName = d => L(DAYS[d][0], DAYS[d][1]);", "const dayName = d => { const x = DAYS[d]; return x ? L(x[0], x[1]) : '' };")
rep("D().kid_logins[f.dataset.pid] = Object.assign(", "(D().kid_logins = D().kid_logins || {})[f.dataset.pid] = Object.assign(")
rep("D().kid_logins[d.pid] = Object.assign({}, D().kid_logins[d.pid], j.rec)", "(D().kid_logins = D().kid_logins || {})[d.pid] = Object.assign({}, D().kid_logins[d.pid], j.rec)")
rep("D().kid_logins[d.pid] = Object.assign({}, k, j.rec)", "(D().kid_logins = D().kid_logins || {})[d.pid] = Object.assign({}, k, j.rec)")

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
   - If it prints `STOPPED: ...`, then `HOST_Egypt.html` changed since v4.6 and nothing was written. In that case, apply the same fixes by hand to a copy named `HOST_Egypt_beta.html`. The comments in the script list every change. Then go to step 4.

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
   git add HOST_Egypt_beta.html host-egypt-beta-settings.js && git commit -m "Add BETA page v4.7: security and bug fixes (live page untouched)"
   ```

6. Push, and get it onto `main`. If you're on a branch, open a pull request and merge it once I approve. GitHub Pages only publishes `main`. The beta will then open at:
   `https://layer0data.github.io/host-egypt/HOST_Egypt_beta.html`

7. Tell me the link and confirm that the live page `HOST_Egypt.html` didn't change.

**Later, when the beta has been stable for 1–2 weeks**
- Copy the beta over the live page and change its `<script src="host-egypt-beta-settings.js">` back to `host-egypt-settings.js`.
- Delete the two beta files.
- Do this only when I ask.
