# ARROW — halaman pemeriksaan (review page)

Static review page for the ARROW approval flow. One file: `index.html`.

**This is not `arrow-pwa`.** That project is the iPhone data-entry app, with
reduced capabilities and a different backend. This folder is only the
Maker–Checker–Approver review page for the Android app's backend, and the two
must not be merged or cross-imported.

## Why this is hosted here and not in Apps Script

The page used to be served by Apps Script itself (`backend/Approval.html`, via
`?page=review`). On a phone that is signed into a Google account, opening an
Apps Script web app resolves the deployment against that account and Drive
answers with *"Maaf, saat ini tidak dapat membuka file"* — even though the
deployment is public. Incognito worked, which is the clue: the account context
is the problem, not the link.

Served as a static file, the page is just HTML. Its `fetch()` calls send no
credentials, so every request to the backend is anonymous and that account
context never exists.

Two things make the cross-origin call work, and both are load-bearing:

- the Apps Script response carries `Access-Control-Allow-Origin: *`
- the request uses `Content-Type: text/plain;charset=utf-8`, which keeps it a
  "simple" request so the browser never sends a CORS preflight — Apps Script
  cannot answer one

Do not "fix" the content type to `application/json`. That triggers a preflight
and the page stops working.

## Publishing to GitHub Pages

```bash
git init && git add -A && git commit -m "ARROW review page"
```

Create a repo named `arrow-review` on GitHub, then:

```bash
git remote add origin https://github.com/YOUR-USERNAME/arrow-review.git
git branch -M main && git push -u origin main
```

In the repo: **Settings → Pages → Source: Deploy from a branch → `main` / `/ (root)`**.

The site appears at `https://YOUR-USERNAME.github.io/arrow-review/` after a
minute or two.

## After it is live

In `backend/Approvals.gs`, set:

```javascript
var REVIEW_PAGE_URL = 'https://YOUR-USERNAME.github.io/arrow-review/';
```

Then paste `Approvals.gs` and `Code.gs` into the Apps Script editor, save, and
**deploy a new version** — a deployment is a frozen snapshot, so saving alone
changes nothing. Then run `setUpApprovals()` and `checkApprovalSetup()`.

## The link format

`approvalLinks()` builds links like:

```
https://YOUR-USERNAME.github.io/arrow-review/#m=2026-07&t=<token>
```

The token is in the **fragment**, after `#`. Browsers never send the fragment to
a server, so the token stays out of GitHub's access logs and out of `Referer`
headers. It follows that the whole link must be copied — a link truncated at the
`#` will show *"Tautan tidak lengkap"*.

The token is an HMAC of the approver's email plus an expiry, signed with
`TOKEN_SECRET`. A valid signature alone is not enough: `findApprover()` also
requires the email to still be listed on the `Approvers` tab, so deleting a row
revokes access immediately.

## Publishing this page does not expose the data

The repo is public, but it holds no data and no secret — only the page and the
`/exec` endpoint URL, which is already inside the distributed APK. Every request
is authenticated by the token in the fragment, which lives only in the links
sent to the eight named approvers.
