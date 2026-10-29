# Setting up the admin

The admin at `/admin/` lets anyone with a sign-in edit every word, image
and video on the site from a browser. They don't need a GitHub account or git.
This guide covers the one-time setup and then the full edit, save and publish
flow you use to check that it works.

**How it fits together.** The site is a Cloudflare **Worker** with static
assets. It is *not* Cloudflare Pages. The Worker serves the built pages, and
for `/api/admin/*` only, it also runs the admin script in
`workers/admin/src/index.js`. That script holds a GitHub token and commits
your edits to the repository for you:

```
/admin  ──Save draft──▶  commit to the `draft` branch
        ──Publish─────▶  merge `draft` into `main`
                          └─▶ GitHub Actions sees the push, runs build.py, deploys
                               └─▶ public site updated (1–3 minutes)
```

So three things have to be true for a change to reach the public page:
the Worker must have its **secrets**, the repository must have the **deploy
workflow and its two Cloudflare secrets**, and the Worker name in
`wrangler.toml` must be the Worker you are looking at.

| | |
|---|---|
| Repository | `tonyd-official/Tony_Deng_Website` |
| Worker | `tony-deng-website` |
| Address | `tony-deng-website.jimmy98006.workers.dev` (a custom domain, tonydengmusic.com, is to follow) |
| Deploys | `.github/workflows/deploy.yml`, on every push to `main` |

---

## 1. Deploys (GitHub Actions)

Cloudflare does not build anything. Every push to `main`, including each
Publish from the admin, runs `.github/workflows/deploy.yml`, which runs
`python build.py --dist` and then `wrangler deploy`. Saves to `draft` do not
deploy. The workflow needs two repository secrets (GitHub → the repository →
**Settings** → **Secrets and variables** → **Actions**):

| Name | Value |
|---|---|
| `CLOUDFLARE_API_TOKEN` | a Cloudflare API token that can edit Workers on the account |
| `CLOUDFLARE_ACCOUNT_ID` | the Cloudflare account ID |

To deploy without a push: **Actions** → **Deploy to Cloudflare** → **Run
workflow**. Every run, and its log if it fails, is under **Actions**.

Check: open the workers.dev address. The site loads. Open `/admin/`. The
sign-in box appears.

> `wrangler deploy` deploys to the Worker named on the `name =` line of
> `wrangler.toml`. If that line names a different Worker, the deploy goes
> somewhere you did not expect.

> Branch protection: blocking force-pushes and deletion on `main` is fine.
> Do not protect `draft`, and do not require pull requests on `main`:
> Publish merges `draft` into `main` directly and then moves `draft` to the
> merge with a forced update, and either rule makes it fail.

## 2. Make a GitHub token

github.com → your picture → **Settings** → **Developer settings** →
**Personal access tokens** → **Fine-grained tokens** → **Generate new token**.

This is the token the **admin** commits with. It is a Worker secret (step 3),
not the `GITHUB_TOKEN` that GitHub Actions makes for itself.

- **Resource owner**: `tonyd-official`
- **Token name**: e.g. `tony-site-admin`
- **Expiration**: as long as you are comfortable with (up to a year). Put
  the date in your calendar. **When the token expires, saving stops
  working**, and the admin shows `GitHub 401` (see Troubleshooting).
- **Repository access**: *Only select repositories* →
  `tonyd-official/Tony_Deng_Website` only.
- **Permissions** → Repository permissions → **Contents: Read and write**.
  (Metadata: Read-only is added automatically.) Nothing else.

Generate it and copy it straight away, because GitHub shows it only once.

## 3. Add the two secrets to the Worker

The Worker must exist (deployed once) before the dashboard accepts secrets.

Cloudflare dashboard → **Workers & Pages** → the Worker → **Settings** →
**Variables and Secrets** → **Add**. Add each of these with type **Secret**
(not Text):

| Name | Value |
|---|---|
| `GITHUB_TOKEN` | the token from step 2 |
| `ADMIN_PASSWORD` | the owner password. Make it long. |

Click **Deploy** when it asks. Secrets survive later deploys, so this is
once only. Never write the password into a file. The repository is
public.

Changing `ADMIN_PASSWORD` later signs everyone out.

## 3b. Separate sign-ins

Each person has their own username and password, managed from **site admin**
in the top-left corner of the admin. They are kept in a Durable Object
(`AdminStore`) that the deploy creates from `wrangler.toml`, so there is
nothing to set up in the dashboard.

The first account is **TonyDeng**, and its password is `ADMIN_PASSWORD`.
TonyDeng can change it to something easy to remember from **site admin**,
and `ADMIN_PASSWORD` keeps working for TonyDeng as the way back in. Removing
TonyDeng asks for `ADMIN_PASSWORD`; removing anyone else asks for an
"are you sure?". New people start on `1234567890` and must choose their own
password the first time they sign in.

## 3c. Email (password links and "Message Tony")

1. Make a free account at **brevo.com**. Under **Senders, domains & dedicated
   IPs → Senders**, add and verify the address mail should come from.
2. **SMTP & API → API keys** → generate a key.
3. Add two more secrets to the Worker, as in step 3:

| Name | Value |
|---|---|
| `BREVO_API_KEY` | the key from Brevo |
| `MAIL_FROM` | the verified sender address |

Then in the admin set **Where messages go** (on the Contact & footer tab,
or on the site admin page). Every message is emailed there with the
sender as the reply address, so pressing Reply answers them directly. People
who leave a phone number are answered from the inbox on the site admin
page, which opens the replier's own phone to text them.

Without a domain of its own, the site sends from a free address, so the
first emails may land in spam: mark them "not spam" once.

## 4. Check the full flow

Do this once after setup, and again whenever something seems wrong.

1. Open `https://<worker address>/admin/` and sign in (TonyDeng and `ADMIN_PASSWORD` the first time).
2. The bar at the top should say **signed in · live site is up to date**.
   If it says *Not connected* or shows a red message, go to Troubleshooting.
3. Pick something easy to spot, e.g. **Tony D** tab → a line of hero text.
   Click into the box and change a word. The bar shows **unsaved changes**,
   and **Save draft first** appears beside Publish.
4. Optionally type a note in *What changed?*, then press **Save draft**
   (or Ctrl+S / ⌘S). A green "Saved to the draft branch" message appears,
   and the bar shows **1 change waiting to publish**.
5. Press **Publish** → OK. A green "Published" message appears.
6. GitHub → the repository → **Actions**. A new **Deploy to Cloudflare** run
   starts within a few seconds. Wait for the green ✓ (usually 1–3 minutes).
7. Open the public page in a new tab (or hard-refresh with Ctrl+F5) and see
   the change.

To undo the test, change the word back and repeat steps 4–5.

### Why Publish is sometimes grey

Publish sends what is **saved** on the draft branch. It does not send what is
in the form. So it waits until:

- there are no unsaved edits (the hint says **Save draft first**), and
- there is at least one saved change that is not live yet (the hint says
  **Nothing new to publish**).

## Troubleshooting

| What you see | Cause and fix |
|---|---|
| Clicking a text box does nothing / typing goes nowhere | Fixed in October 2026. Each rich text field was wrapped in a `<label>`, so a click moved focus to its **B** button. If you still see it, the Worker is serving an old build. Check **Actions**. |
| `Wrong username or password` | Just that, or `ADMIN_PASSWORD` is not set on *this* Worker. |
| `The ADMIN_PASSWORD secret is not set` (or GITHUB_TOKEN) | Step 3 was skipped, or done on a Worker other than `tony-deng-website`. |
| `GitHub 401 on …` | The token expired or was revoked. Make a new one (step 2) and replace the `GITHUB_TOKEN` secret (step 3). |
| `GitHub 403 on …` or `GitHub 404`-style "not found" | The token cannot see this repository, or lacks *Contents: Read and write*. Edit the token's repository access and permissions. |
| `Someone else saved while you were editing` | Someone else saved first (or another tab did). Press **Reload**, make your change again, and save. Nothing they saved is lost. |
| `Merge conflict between draft and live` | Someone changed `main` directly in a way that conflicts. This needs someone with git. |
| Published, but the public page has not changed | The deploy did not start or it failed. Open **Actions**: re-run a failed run, or **Run workflow** if none appeared. A red run whose log mentions authentication means the `CLOUDFLARE_*` repository secrets (step 1) are wrong. Then hard-refresh the page. |
| Publish fails with a GitHub error that mentions a protected branch, or says "Merge conflict" when nobody touched `main` | Branch protection on `draft`, or a pull-request rule on `main`, blocks the admin. See the note in step 1. |
| `Not signed in` after a while | The session lasts 7 days, or ends sooner if the password was changed. Sign in again. Unsaved edits stay on the page if you sign in from a **new tab**. |

## Trying it without deploying anything

```
python tools/devserver.py
```

This serves the site and a local stand-in for the admin API at
<http://localhost:5502/admin/>. It writes straight to `content/*.json` on disk
and rebuilds after each save. There is **no password**, so use it on
localhost only.

## What the admin cannot do

These are left out on purpose, because each needs someone with git:

- Changing the page structure, such as section order, new sections or layout
- Editing CSS or JavaScript
- Deleting images from the repository (uploading a replacement is enough;
  old files just sit there)
- Resolving a merge conflict between `draft` and `main`

## Later: a domain and Cloudflare Access

With separate sign-ins (step 3b) each person has their own password, and a
removed person is signed out at once. What is still missing: everyone who can
sign in can publish, and the sign-in page is open to anyone who finds
`/admin/` (eight wrong tries locks that username for 15 minutes). Once the
site has its own domain, Cloudflare Access can go in front of `/admin/` as
well. The earlier version of this file (in git history, before October 2026)
has those steps.
