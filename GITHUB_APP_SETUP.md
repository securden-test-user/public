# How to create a GitHub App for Securden NHI

This guide walks through creating a GitHub App with the permissions Securden uses to:

- Discover repositories, identities, and secrets
- Resync automatically when code is pushed to the default branch
- Run live remediation (rotate CI/CD secrets, edit files, manage deploy keys, etc.)

Replace `<host>` with your Securden server URL (must be reachable from the public internet for webhooks).

---

## Before you start

You need:

- GitHub **Organization owner** or **personal account** admin access (to create the app and install it)
- Securden with **SecurdenNhiDetector.exe** installed
- A stable HTTPS URL for Securden, e.g. `https://pam.company.com`

---

## Step 1 — Create the GitHub App

1. Log in to GitHub.
2. Open **Settings** (your profile or organization).
3. Go to **Developer settings → GitHub Apps → New GitHub App**.

### Basic details

| Field | Example |
|-------|---------|
| **GitHub App name** | `Securden NHI` |
| **Homepage URL** | `https://<host>` |
| **Webhook** | **Active** ✓ |
| **Webhook URL** | `https://<host>/nhi/git_repositories/github/webhook/` |
| **Webhook secret** | Generate a long random string and **save it** — you enter the same value in Securden |

Leave **Callback URL** empty unless you also use OAuth (GitHub App auth in Securden does not need it).

### Where can this app be installed?

- **Only on this account** — personal GitHub user
- **Any account** — if you plan to install on customer orgs (typical for vendors)

---

## Step 2 — Repository permissions

Set these under **Repository permissions**. Use **Read and write** where noted if you want **live remediation**; otherwise **Read-only** is enough for discovery-only.

| Permission | Discovery (scan) | Remediation (live fix) | What Securden uses it for |
|------------|:----------------:|:----------------------:|---------------------------|
| **Contents** | Read | **Read and write** | List/read repo files for secret scanning; commit file fixes |
| **Metadata** | Read | Read | Required (automatic) — repo list, basic repo info |
| **Actions** | Read | **Read and write** | Workflows, runners, Actions permissions; update CI/CD variables |
| **Secrets** | Read | **Read and write** | GitHub Actions secrets (repo + environment scope) |
| **Dependabot secrets** | Read | Read | Dependabot secrets inventory |
| **Environments** | Read | **Read and write** | Deployment environments; delete environment remediation |
| **Webhooks** | Read | Read | Repository webhook identities |
| **Administration** | Read | **Read and write** | Deploy keys, collaborators, rulesets, repo visibility, rename, default branch |
| **Pull requests** | Read | Read | Actions “approve PR reviews” policy |
| **Codespaces** | Read | Read | Codespaces and Codespaces secrets |

### Discovery-only (minimum)

If you only need scan + webhook resync (no live fixes from the Securden UI):

- Set every permission above to **Read only**, except **Metadata** (Read).
- **Contents: Read** is mandatory for scanning repository files.

---

## Step 3 — Organization permissions

Required when the app is **installed on an organization** (not just a personal account).

| Permission | Discovery | Remediation | What Securden uses it for |
|------------|:---------:|:-----------:|---------------------------|
| **Administration** | Read | Read | List **other GitHub Apps** installed on the org |
| **Personal access tokens** | Read | Read | List **fine-grained PATs** with org access |
| **Secrets** | Read | **Read and write** | Organization-level Actions secrets |
| **Actions variables** | Read | **Read and write** | Organization-level Actions variables |
| **Members** | Read | Read | Organization member / bot identities |

If you install only on a **personal user account**, organization permissions can stay at **No access** — but **fine-grained PAT and OAuth/classic PAT identities will not be discovered** (only Securden’s own GitHub App installation via App JWT).

**Full lab walkthrough** (create org, repo, PAT, OAuth app, people, verify discovery): [GITHUB_NHI_LAB_SETUP.md](GITHUB_NHI_LAB_SETUP.md)

---

## Step 4 — Account permissions

Securden also calls a few **user-level** APIs (SSH auth keys, SSH signing keys, user Codespaces):

| Permission | Recommended | Notes |
|------------|-------------|-------|
| **Codespaces** | Read | User Codespaces listing |
| **SSH signing keys** | Read and write | Only if you need signing-key remediation |
| **Git SSH keys** | — | Not a separate GitHub App permission; user SSH keys (`/user/keys`) may be **partially unavailable** with installation tokens. Use **GitHub OAuth** in Securden if SSH key inventory is critical. |

If account-level keys do not appear after a scan, check Securden logs — discovery continues for repositories even when `/user/keys` is denied.

---

## Step 5 — Subscribe to events

Under **Subscribe to events**, enable:

| Event | Required | Why |
|-------|:--------:|-----|
| **Push** | ✓ Yes | Triggers automatic resync when code lands on `main` / `master` / default branch |

Other events are not required for NHI.

---

## Step 6 — Create and download the private key

1. Click **Create GitHub App**.
2. On the app page, click **Generate a private key**.
3. Save the downloaded `.pem` file securely — Securden needs the full PEM contents once; GitHub will not show it again.

Note the **App ID** on the same page (numeric, e.g. `123456`).

---

## Step 7 — Install the app

1. On the GitHub App page, click **Install App** (left sidebar).
2. Choose the **organization or user** that owns the repositories you want to scan.
3. Choose **All repositories** or select specific repos.
4. Click **Install**.

### Find the Installation ID

After install, the browser URL looks like:

```
https://github.com/settings/installations/12345678
```

or for an org:

```
https://github.com/organizations/my-org/settings/installations/12345678
```

The number at the end (`12345678`) is the **Installation ID**. It is **numeric only**.

> **Common mistake:** Do **not** paste the OAuth **Client ID** here (values like `Iv23lirpquvLE1adlhyM`).  
> Client ID is for OAuth apps, not GitHub App installation.

---

## Step 8 — Configure Securden

1. In Securden: **Admin → NHI → Data Sources → Add → GitHub**.
2. Authentication: **GitHub App**.
3. Fill in:

| Securden field | Value |
|----------------|-------|
| Profile name | Any label, e.g. `GitHub Production` |
| App ID | From GitHub App settings |
| Installation ID | From install URL (Step 7) |
| Private Key | Paste entire `.pem` file contents |
| Webhook secret | Same string you set in Step 1 |

4. Click **Scan**.

Securden validates the app, stores credentials encrypted, and starts the first repository scan.

---

## Step 9 — Verify webhooks

1. On the GitHub App settings page, open **Advanced → Recent Deliveries**.
2. After a push to `main`, you should see deliveries to  
   `https://<host>/nhi/git_repositories/github/webhook/`
3. **Ping** events should return `200` when you first save the webhook URL.

If deliveries fail:

- Confirm Securden is reachable from the internet (not localhost-only).
- Confirm the URL path is exactly `/nhi/git_repositories/github/webhook/`.
- Confirm Django is running and the NHI URL routes are deployed.

---

## Permission summary (copy-paste checklist)

Use this when filling the GitHub App form.

### Full scan + remediation

**Repository permissions**

- Contents → **Read and write**
- Metadata → **Read-only**
- Actions → **Read and write**
- Secrets → **Read and write**
- Dependabot secrets → **Read-only**
- Environments → **Read and write**
- Webhooks → **Read-only**
- Administration → **Read and write**
- Pull requests → **Read-only**
- Codespaces → **Read-only**

**Organization permissions** (org install only)

- **Administration** → **Read-only** (list other GitHub App installations)
- **Personal access tokens** → **Read-only** (fine-grained PATs with org access)
- Secrets → **Read and write**
- Actions variables → **Read and write**
- Members → **Read-only**

**Account permissions**

- Codespaces → **Read-only**
- SSH signing keys → **Read and write** (optional; for signing-key actions)

**Events**

- Push → **Subscribed**

### Scan + webhook only (no live remediation)

Use the same list but set **Contents, Actions, Secrets, Environments, Administration** to **Read-only** only.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Scan starts but no repos | App not installed on target org/user | Install app (Step 7) |
| `Could not get GitHub App token` | Wrong App ID, Installation ID, or PEM | Re-check all three fields |
| No SSH keys in Identities | Installation token cannot read `/user/keys` | Add OAuth data source or accept repo-only coverage |
| No GitHub App / PAT / OAuth identities | Personal-user install only, or missing org permissions | Install app on an **organization**; grant **Administration (Read)** and **Personal access tokens (Read)**; ensure org-owned repos are scanned; re-approve org permissions after changes |
| `user/installations` HTTP 403 in logs | Expected with GitHub App token | Securden uses App JWT (`GET /app/installations`) instead; update `discovery.py` and re-scan |
| `orgs/.../personal-access-tokens` HTTP 403 | Org PAT policy or permission missing | Enable org fine-grained PAT approval policy; grant **Personal access tokens → Read** on the GitHub App |
| `orgs/.../credential-authorizations` HTTP 403 | Org admin / plan limitation | Requires org admin; OAuth/classic PAT listing may be empty on some GitHub plans |
| Webhook never fires | Push was not to default branch | Only `main` / `master` / default branch triggers resync |
| Webhook 403 in Securden logs | Webhook secret mismatch | Use the same secret in GitHub and Securden |
| Actions secrets missing | **Secrets** permission too low | Set Secrets → Read (or Read and write) |
| Live secret rotate fails | **Secrets** or **Actions** not Read and write | Upgrade permissions and reinstall if GitHub prompts |

---

## Related Securden endpoints

| Purpose | URL |
|---------|-----|
| GitHub App webhook | `POST https://<host>/nhi/git_repositories/github/webhook/` |
| Add data source (UI) | Admin → NHI → Data Sources → GitHub → GitHub App |

For PAT or OAuth setup (no app webhook), see the main [git_repositories README](../README.md).

**End-to-end lab guide** (org, repo, PAT, OAuth, people, discovery checklist): [GITHUB_NHI_LAB_SETUP.md](GITHUB_NHI_LAB_SETUP.md)
