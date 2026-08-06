# GitHub NHI — Full lab setup guide

Use this guide to build a **test environment from scratch** on GitHub and verify that Securden discovers every identity type correctly.

It answers the question: **Is this a code bug or a configuration gap?**

| Symptom | Usually configuration | Usually code |
|---------|:---------------------:|:------------:|
| Repo scan works, secrets found | ✓ | |
| Webhook resync works | ✓ | |
| `discovered 1 GitHub App installation via App JWT` in logs | ✓ (personal install) | |
| No fine-grained PAT / OAuth identities | ✓ (need org install + org permissions) | |
| `GET /user` HTTP 403 with GitHub App | ✓ (expected — not a broken token) | |
| Scan crashes / ImportError / 404 webhook | | ✓ |
| Identities created in DB but UI empty | | ✓ (check UI filter first) |

Replace `<host>` with your Securden URL (e.g. `https://pam.company.com` or your ngrok URL).

---

## What Securden discovers (reference)

After a successful **organization** scan with a correctly permissioned **GitHub App** data source, you should see rows like these under **NHI → Identities**:

| What you create on GitHub | Securden identity type | Metadata `kind` |
|---------------------------|------------------------|-----------------|
| Repository | Repository | `github_repository` |
| GitHub App (Securden connector) | Bot | `github_app_installation` |
| Another GitHub App on the org | Bot | `github_app_installation` |
| Fine-grained PAT with org access | Repository Access Token | `github_fine_grained_pat` |
| OAuth app authorized for org | Repository Access Token | `github_oauth_app` |
| Classic PAT authorized for org (if API returns it) | Repository Access Token | `github_classic_pat` |
| Deploy keys, webhooks, Actions runners, etc. | Various | See scan logs |

**Secrets** (Actions secrets, Dependabot secrets, variables, leaked tokens in code) appear under **NHI → Secrets**, not Identities.

---

## Part 0 — Prerequisites

1. **Securden** with NHI enabled and **SecurdenNhiDetector.exe** installed.
2. **GitHub account** with permission to create an organization (free tier is fine).
3. **Public HTTPS URL** for Securden (ngrok is OK for labs):
   - Webhook: `https://<host>/nhi/git_repositories/github/webhook/`
4. Copy latest code to your Django server and restart after deploy.

---

## Part 1 — Create a GitHub Organization

> **Required for PAT and OAuth programmatic identity discovery.**  
> Personal-user-only installs discover Securden’s own GitHub App only.

1. GitHub → your profile menu → **Your organizations** → **New organization**.
2. Choose **Free** plan.
3. Organization name example: `securden-nhi-lab`.
4. Contact email: your email → **Create organization**.

### Enable fine-grained PAT policy (org)

1. Org → **Settings** → **Personal access tokens** → **Settings**.
2. Enable **Require approval of fine-grained personal access tokens** (or allow them for the org).
3. Save.

Without this, `GET /orgs/{org}/personal-access-tokens` may return empty or 403.

---

## Part 2 — Add and manage people

### Invite org members

1. Org → **People** → **Invite member**.
2. Enter GitHub username or email → role **Member** (or **Owner** for full admin APIs).
3. Invitee accepts the email invitation.

### Org roles (quick reference)

| Role | Can approve PAT policy | Can see credential-authorizations API |
|------|:------------------------:|:-------------------------------------:|
| Owner | Yes | Yes (org admin) |
| Member | No | Limited |

For lab testing, use an **Owner** account for the Securden scan credential owner.

### Repository access (after repo exists)

- **Org-owned repo, private:** Org members need explicit team/repo access unless default permissions grant it.
- **Collaborator (outside org):** Repo → **Settings** → **Collaborators** → **Add people**.

---

## Part 3 — Create a test repository

1. Org home → **Repositories** → **New repository**.
2. Suggested values:

| Field | Value |
|-------|-------|
| Owner | `securden-nhi-lab` (your org) |
| Name | `nhi-test-repo` |
| Visibility | Private (recommended) |
| Initialize | Add README ✓ |

3. Click **Create repository**.

### Add sample content (optional, for secret scan)

```bash
git clone https://github.com/securden-nhi-lab/nhi-test-repo.git
cd nhi-test-repo
mkdir -p .github/workflows
echo 'name: CI' > .github/workflows/ci.yml
echo 'on: push' >> .github/workflows/ci.yml
echo 'jobs: { build: { runs-on: ubuntu-latest, steps: [{ uses: actions/checkout@v4 }] } }' >> .github/workflows/ci.yml
git add .
git commit -m "Add workflow"
git push origin main
```

### Add an Actions secret (optional)

Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**  
Name: `LAB_TEST_SECRET`, Value: any string → Securden should list it under **Secrets** after scan.

---

## Part 4 — Create the Securden GitHub App (connector)

This is the app Securden uses to scan. **Not** the same as an OAuth App.

Detailed reference: [GITHUB_APP_SETUP.md](GITHUB_APP_SETUP.md)

### 4.1 Create the app

1. GitHub → **Settings** (your user) → **Developer settings** → **GitHub Apps** → **New GitHub App**.
2. Fill in:

| Field | Example |
|-------|---------|
| GitHub App name | `Securden NHI Lab` |
| Homepage URL | `https://<host>` |
| Webhook | Active ✓ |
| Webhook URL | `https://<host>/nhi/git_repositories/github/webhook/` |
| Webhook secret | Long random string — save for Securden |

**Where can this GitHub App be installed?** → **Any account** (recommended for labs).

### 4.2 Repository permissions

Set for **discovery + remediation** (use Read-only everywhere if you only need inventory):

| Permission | Discovery | Remediation |
|------------|:---------:|:-----------:|
| Contents | Read | Read and write |
| Metadata | Read | Read |
| Actions | Read | Read and write |
| Secrets | Read | Read and write |
| Dependabot secrets | Read | Read |
| Environments | Read | Read and write |
| Webhooks | Read | Read |
| Administration | Read | Read and write |
| Pull requests | Read | Read |
| Codespaces | Read | Read |

### 4.3 Organization permissions (critical for PAT / OAuth discovery)

Install target must be the **org**. Set:

| Permission | Level | Why |
|------------|-------|-----|
| **Administration** | **Read-only** | List other GitHub Apps on the org |
| **Personal access tokens** | **Read-only** | List fine-grained PATs with org access |
| Secrets | Read (or R/W) | Org Actions secrets |
| Actions variables | Read (or R/W) | Org Actions variables |
| Members | Read-only | Org member identities |

### 4.4 Account permissions

| Permission | Level |
|------------|-------|
| Codespaces | Read-only |
| SSH signing keys | Read-only (optional) |

### 4.5 Events

Subscribe to **Push** only.

### 4.6 Private key and App ID

1. **Create GitHub App**.
2. Note **App ID** (numeric, e.g. `123456`).
3. **Generate a private key** → save the `.pem` file.

### 4.7 Install on the organization

1. App page → **Install App** → choose **`securden-nhi-lab`** (your org).
2. **All repositories** or select `nhi-test-repo`.
3. **Install** → if GitHub shows **Review permissions**, approve org access.
4. Copy **Installation ID** from the URL:
   ```
   https://github.com/organizations/securden-nhi-lab/settings/installations/12345678
   ```
   Use `12345678` only — **not** the OAuth Client ID (`Iv23...`).

---

## Part 5 — Create a fine-grained PAT (discovered identity)

This PAT is **inventory only** — Securden lists it as an identity; it is not the scan credential.

1. GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
2. Token name: `Securden lab FG PAT`.
3. **Resource owner:** `securden-nhi-lab` (your org) — **not** your personal user.
4. Repository access: **Only select repositories** → `nhi-test-repo` (or All repositories).
5. Permissions (minimum for lab):

| Category | Permission |
|----------|------------|
| Repository → Metadata | Read-only |
| Repository → Contents | Read-only |

6. **Generate token** → copy once (you will not see it again).

7. If org requires approval: Org → **Settings** → **Personal access tokens** → **Pending requests** → **Approve**.

**Expected after scan:** Identity type **Repository Access Token**, kind `github_fine_grained_pat`, name like `org:securden-nhi-lab / PAT Securden lab FG PAT (...)`.

---

## Part 6 — Create a classic PAT (optional, limited discovery)

Classic PATs (`ghp_...`) **cannot be fully enumerated** by GitHub’s API. Securden may list them only if they appear in `GET /orgs/{org}/credential-authorizations` (org admin, plan-dependent).

To create one anyway (e.g. as a **Securden scan credential** or for manual API tests):

1. **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)** → **Generate new token**.
2. Note: `Securden lab classic PAT`.
3. Scopes: `repo`, `read:org`, `read:user`, `read:public_key`, `read:ssh_signing_key`.
4. Generate and copy.

**Using classic PAT in Securden:** Data source auth = **Token based**, enter username + PAT. Good for SSH key inventory; **no webhook auto-sync**.

---

## Part 7 — Create an OAuth App

Two different uses:

| Use | Purpose |
|-----|---------|
| **A. Securden data source (OAuth auth)** | Securden scans as the authorizing user; good for SSH keys, `/user/installations`. |
| **B. Lab OAuth app on the org** | Authorize it against the org so it may appear in credential-authorizations discovery. |

### 7A — OAuth App for Securden connector

1. **Developer settings** → **OAuth Apps** → **New OAuth App**.
2. Fill in:

| Field | Value |
|-------|-------|
| Application name | `Securden NHI OAuth Lab` |
| Homepage URL | `https://<host>` |
| Authorization callback URL | `https://<host>/nhi/github_oauth_callback` |

3. **Register application** → note **Client ID** and generate **Client secret**.

**In Securden:** Add GitHub data source → **OAuth** → Client ID + Secret → **Scan** → approve in popup.

Securden requests scopes: `repo`, `read:user`, `read:public_key`, `admin:public_key`, `read:ssh_signing_key`, `admin:ssh_signing_key`, `codespace`.

### 7B — OAuth App to be discovered (org inventory)

1. Create a **second** OAuth App (e.g. `Lab Third-Party OAuth`).
2. As an **org owner**, open the app’s authorization URL in browser:
   ```
   https://github.com/login/oauth/authorize?client_id=YOUR_CLIENT_ID&scope=read:org repo
   ```
3. Authorize for the org when prompted.

**Expected after org scan (GitHub App data source):** May appear as **Repository Access Token**, kind `github_oauth_app`, if `GET /orgs/{org}/credential-authorizations` succeeds.

---

## Part 8 — Install a second GitHub App (optional)

To verify Securden lists **other** apps on the org (not only Securden’s app):

1. Install any public GitHub App on your org (e.g. a free CI or security tool), **or** create a second test GitHub App and install it on the org.
2. Requires Securden connector app to have org **Administration → Read-only**.

**Expected:** Extra **Bot** identity, kind `github_app_installation`, name like `org:securden-nhi-lab / GitHub App ...`.

---

## Part 9 — Configure Securden

1. **Admin → NHI → Data Sources → Add → GitHub**.
2. Authentication: **GitHub App** (recommended for full lab).
3. Fields:

| Field | Source |
|-------|--------|
| Profile name | `GitHub Lab Org` |
| App ID | Step 4.6 |
| Installation ID | Step 4.7 (org install, numeric) |
| Private Key | Full `.pem` contents |
| Webhook secret | Same as GitHub App webhook secret |

4. Click **Scan** → wait for task to complete.
5. **Identities** filter → **All**.

---

## Part 10 — Verify discovery (checklist)

### Django logs (success patterns)

```
GitHub NHI discovered N GitHub App installation(s) via App JWT.
GitHub NHI discovered ...   (no "programmatic access ... was not queried" if org repos exist)
```

Should **not** see (for org setup):

```
programmatic access ... was not queried: no organization-owned repos
orgs/{org}/personal-access-tokens returned HTTP 403
orgs/{org}/installations returned HTTP 403
```

`Could not verify token (HTTP 403)` on `GET /user` is **OK** with GitHub App — ignore it.

### UI checklist

| # | Created on GitHub | Found in Securden Identities? |
|---|-------------------|:-----------------------------:|
| 1 | Org repo `nhi-test-repo` | Repository |
| 2 | Securden GitHub App on org | Bot (`github_app_installation`) |
| 3 | Fine-grained PAT (Part 5) | Repository Access Token |
| 4 | Second GitHub App (Part 8) | Bot |
| 5 | OAuth authorized to org (Part 7B) | Repository Access Token (if API allows) |
| 6 | Actions workflow in repo | CI/CD identity |
| 7 | Repo webhook / deploy key | Webhook / deploy key types |

### If something is missing

| Missing | Check |
|---------|-------|
| All org programmatic types | App installed on **org**, not personal user only |
| Fine-grained PAT | Org PAT policy enabled; token **resource owner = org**; PAT approved; app has **Personal access tokens Read** |
| Other org apps | **Administration Read** on GitHub App; re-approve org permissions |
| OAuth / classic PAT | Org **owner** running scan context; credential-authorizations may 403 on some plans |
| Only 1 GitHub App identity | Expected on personal install; move to org for full set |

---

## Part 11 — Code vs configuration decision tree

```
Scan completes, repos listed?
├─ NO  → Installation ID, PEM, App ID, app installed on correct account
└─ YES → Webhook / secrets work?
         ├─ NO  → Permissions (Contents, Secrets), ngrok URL, webhook secret
         └─ YES → Programmatic identities missing?
                  ├─ Log: "not queried: no organization-owned repos"
                  │        → CONFIG: org install + org-owned repo
                  ├─ Log: "orgs/.../personal-access-tokens HTTP 403"
                  │        → CONFIG: org permission + PAT policy
                  ├─ Log: "discovered 1 ... via App JWT" only
                  │        → CONFIG: expected on user install; add org for PAT/OAuth
                  └─ Log: discovery OK but UI empty
                           → CONFIG: Identities filter = All; correct data source
                           → CODE:  check DB / nhi_views filter (rare)
```

---

## Part 12 — Optional extras (more identities & secrets)

| Action on GitHub | Securden finds |
|------------------|----------------|
| Repo → Settings → Deploy keys → Add | Deploy key identity |
| Repo → Settings → Webhooks → Add | Repository webhook identity |
| Org → Settings → Webhooks | Org webhook identity |
| Actions → Runners → New self-hosted runner | Actions runner identity |
| Org → Settings → Secrets → Actions | Org secret (under **Secrets**) |
| Push AWS key in a file on `main` | Leaked secret (detector) |

---

## Quick copy-paste: GitHub App permission checklist

**Repository:** Contents R, Metadata R, Actions R, Secrets R, Dependabot secrets R, Environments R, Webhooks R, Administration R, Pull requests R, Codespaces R  

**Organization:** Administration R, Personal access tokens R, Secrets R, Actions variables R, Members R  

**Account:** Codespaces R  

**Events:** Push  

**Install on:** Organization with org-owned repos  

---

## Related docs

| Document | Contents |
|----------|----------|
| [GITHUB_APP_SETUP.md](GITHUB_APP_SETUP.md) | Securden GitHub App permissions, webhook, troubleshooting |
| [../README.md](../README.md) | PAT / OAuth data source summary, sync behavior |
