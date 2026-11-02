# What I need from you (step by step)

There are 4 parts:
- **Part A:** you on github.com, about 5 minutes. Needed now.
- **Part B:** a message for your builder session (the books). Needed now.
- **Part C:** installing the new backend on your TEST sheet. Later, only when I say it's ready.
- **Part D:** removing the leaked files. Later, when I say it's ready.

Nothing here touches the live app.

---

## Part A: on github.com (you, about 5 minutes)

A Claude session can't create repositories, so please do Part A yourself in a browser. You must be logged in as **layer0data**.

### A1. Create the private repository
1. Open **https://github.com/new**
2. **Owner:** `layer0data`
3. **Repository name:** `host0-source`
4. Choose **Private**. This matters, because the backend contains the HOST protocol.
5. Tick **Add a README file**.
6. Click **Create repository**.

### A2. Let Claude work in it (and in host0-intake for the clean-up)
1. Open **https://github.com/settings/installations**
2. Next to **Claude**, click **Configure**.
3. Under **Repository access**, keep **Only select repositories**.
4. Click **Select repositories** and add **host0-source** and **host0-intake**. Keep **host-egypt** in the list too.
5. Click **Save**. If GitHub asks you to accept new permissions, click **Accept**.

### A3. Refresh the link with Claude
1. Open **https://claude.ai/connect-github** and click connect or reconnect.

### A4. Tell me
Write **"created"** in this chat.

---

## Part B: copy and paste this message into your builder session

> Hi. Another Claude session is moving the HOST0 source into a private repository and needs the reference books. Please:
>
> 1. Export the **HOST0 Platform Book (v4.6)** and the **OLE Book** from Claude Docs as Markdown.
> 2. Save them in the **private** repository `layer0data/host0-source`, on branch `main`, in the folder `docs/books/`, named `HOST0_Platform_Book_v4.6.md` and `OLE_Book.md`. Commit message: "Add HOST0 and OLE books".
> 3. If you can't push to that repository, share both books instead (Claude Docs share link, view access) and give the two links to me, the user.
> 4. Tell me whether you have anything **newer** than the v4.8 build inside `host0-intake/HOST_Egypt_Practice.html` (29 Oct). If yes, say what it is and where it is.
> 5. Do **not** change the `main` branch or the live files of `host-egypt` or `host0-intake`, and never put PINs or passwords in any repository.
>
> Reply with "books shared" plus the links, or "books pushed".

When the builder answers, paste its answer into this chat.

---

## Part C: install the new backend on the TEST sheet (LATER, only when I say "ready for TEST")

This is your **TEST** Google Sheet, the one used by the TEST page. **Not** the live sheet.

1. Open the **TEST** Google Sheet, then go to **Extensions → Apps Script**.
2. In GitHub, open `host0-source` → `backend/Code.gs` → click **Raw** → select all (Ctrl+A or Cmd+A) → copy.
3. In Apps Script, click the file **Code.gs** → select all → delete → paste.
4. At the **top of the file**, set your **TEST** values in the lines I'll mark with `CHANGE_ME`, for example the TEST owner PIN. Never use the live owner PIN here.
5. Click **Save** (the disk icon).
6. Click **Deploy → Manage deployments**. Click the **pencil** on the existing deployment, set **Version: New version**, then click **Deploy**. This keeps the same web address, so the TEST page needs no change.
7. If Google asks for permission ("Authorize access"), choose your account → **Advanced** → **Go to … (unsafe)** → **Allow**. That warning is normal for your own scripts.
8. Write **"test deployed"** in this chat.

If anything goes wrong, use **Deploy → Manage deployments → pencil → Version** and pick the previous version. That puts the old code back in seconds.

---

## Part D: remove the leaked files (LATER, when I say v4.8 is safe in the private repo)

I'll do this part myself once you agree. I'll delete `OLE_Demo.html` and `HOST_Egypt_Practice.html` from `host0-intake` and also clean its **history**, because deleted files stay downloadable from a public repository's history. Your intake forms keep the same links.

I'll ask you before doing it.
