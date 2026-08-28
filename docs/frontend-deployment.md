# Frontend Deployment, Backup & Recovery (P1-C)

The two production frontends are static HTML served by **GitHub Pages**. GitHub Pages is a
production dependency: if the repo or its Pages build is broken, the owner cannot record or confirm
a sale. This document is the single procedure for changing, backing up, and recovering them.

**Golden rules**

- The **canonical source is the Git repo**, not any local folder and not the GitHub web editor.
- **Deployment = `git commit` + `git push` to `main`.** No drag-and-drop uploads, no editing files
  in the GitHub web UI — *except* the emergency rollback in §7/§9.
- **Compare SHA-256 (or `git diff`) before and after every production change** (§4, §6, §10).
- Rollback is always to a **known-good Git commit**, never hand-retyped from memory.
- **No secrets in this repo, ever** — see §11.

---

## 1. Canonical repo and branch

- Repo: **`https://github.com/karenplataaiwork-ctrl/siigo-voiceinvoicing.git`** (public)
- Branch: **`main`** (only branch; `origin/HEAD` → `main`)
- GitHub Pages source: **Deploy from a branch → `main` / `(root)`**
  - Inferred from repo structure (all files at root, no `gh-pages` branch, no `/docs` publish
    folder, no `.github/workflows`) and live behaviour (`server: GitHub.com`, files served at
    `https://karenplataaiwork-ctrl.github.io/siigo-voiceinvoicing/<file>`).
  - **Confirm once** in GitHub → repo → *Settings → Pages*: Source = "Deploy from a branch",
    Branch = `main`, Folder = `/ (root)`. There is no custom domain (no `CNAME`), so the site
    stays on `*.github.io`.

## 2. Canonical local directory

**`/Users/karenplata/Documents/AI-Automation-Projects/siigo-voiceinvoicing/`**

- This is a plain `git clone` of the repo above. `main` tracks `origin/main`.
- It contains **only** the frontend repo — nothing else.
- **Do not** git-connect the larger project folder
  `/Users/karenplata/Documents/AI-Automation-Projects/siigo/` to this repo. That folder holds
  internal docs and `.mcp.json` (which contains a **live n8n API key**); pushing it to this public
  repo would leak the key. The HTML copies still sitting in `…/siigo/` are **historical / not
  canonical** — do not edit them and do not deploy from them.

## 3. Production URLs

| Page | URL | Purpose |
|---|---|---|
| Recorder | `https://karenplataaiwork-ctrl.github.io/siigo-voiceinvoicing/phase9-recorder.html` | owner records a sale by voice |
| Confirmation | `https://karenplataaiwork-ctrl.github.io/siigo-voiceinvoicing/phase7-confirm.html` | owner reviews + confirms/rejects; polls invoice status |
| Landing | `https://karenplataaiwork-ctrl.github.io/siigo-voiceinvoicing/index.html` | index |
| Privacy | `https://karenplataaiwork-ctrl.github.io/siigo-voiceinvoicing/privacy.html` | privacy notice |
| STT benchmark recorder | `…/phase9-stt-benchmark-recorder.html` | non-production test aid |

Other repo files in `main`: none. The pages call only the n8n webhook host
`https://aiautomationswork.app.n8n.cloud/webhook/…` — that is a public endpoint, not a secret.

## 4. Pre-deploy verification (before you change anything)

```sh
cd /Users/karenplata/Documents/AI-Automation-Projects/siigo-voiceinvoicing
git fetch origin
git status                 # must be: clean, "up to date with origin/main"
git log --oneline -5       # note the current HEAD commit — this is your rollback anchor
shasum -a 256 *.html       # record hashes BEFORE the change
```

Confirm the live site currently matches `main` (should always be true):

```sh
for f in phase9-recorder.html phase7-confirm.html index.html privacy.html; do
  curl -sS "https://karenplataaiwork-ctrl.github.io/siigo-voiceinvoicing/$f" | shasum -a 256 | sed "s|-|$f|"
done
```

If HEAD, working tree, and live do not all agree, **stop** and resolve that first (§9).

## 5. Exact deployment procedure

1. `cd /Users/karenplata/Documents/AI-Automation-Projects/siigo-voiceinvoicing`
2. `git fetch origin && git status` → clean and up to date (§4).
3. Edit the HTML file(s) **in this directory only**.
4. Review the change: `git diff` — read every line. Confirm no secret, no unintended edit.
5. `shasum -a 256 <changed file>` — record the **new** hash.
6. `git add <files>` then `git commit -m "Update <file>: <what changed and why>"`
7. `git push origin main`
8. Wait for the Pages build (usually < 1–2 min). Then run §6.

Never skip straight to editing on github.com. Never `git push --force`.

## 6. Post-deploy verification

```sh
# 1. repo HEAD is your new commit
git log --oneline -3

# 2. live site now serves the new bytes (allow ~1–2 min for the Pages build)
for f in <changed files>; do
  echo "$f"
  curl -sS "https://karenplataaiwork-ctrl.github.io/siigo-voiceinvoicing/$f" | shasum -a 256
  shasum -a 256 "$f"
done
# the two hashes for each file MUST be identical
```

3. Open the page in a browser (hard-refresh / cache-bypass) and click through the real flow:
   recorder → record → **Abrir confirmación**; confirmation page → status polling behaves.
   Do **not** create a real invoice just to test the frontend — use a throwaway/again-rejected
   sale, or stop at the confirmation screen.
4. If the live hash does **not** match after ~5 min, the Pages build failed → check
   *Settings → Pages* and the repo's *Actions*/*Deployments* tab for the failed build; fix or
   roll back (§7).

## 7. Rollback procedure (using Git history — never manual reconstruction)

Use this the moment a deploy is found to be bad.

```sh
cd /Users/karenplata/Documents/AI-Automation-Projects/siigo-voiceinvoicing
git fetch origin
git log --oneline -15        # find the last KNOWN-GOOD commit hash (see the table below)
```

**Preferred — revert the bad commit(s), keeping history:**

```sh
git revert --no-edit <bad_commit_hash>      # repeat / use a range for multiple commits
git push origin main
```

**If several recent commits are bad — roll the branch back to a known-good commit:**

```sh
git reset --hard <known_good_commit_hash>
git push --force-with-lease origin main      # the only sanctioned force-push, rollback only
```

Then run §6 to confirm the live site matches the known-good bytes.

**Emergency only — no working local checkout available:** on github.com, open the repo → the file →
"History" → the known-good version → "..." → *Revert* / copy its raw content into the web editor and
commit. Do this **only** to stop active production breakage, then immediately reconcile the local
checkout (`git fetch && git reset --hard origin/main`) so local matches again.

### Known-good rollback anchors (as of 2026-08-28)

| Commit | Date | Meaning |
|---|---|---|
| `38917a7` | 2026-08-27 | **current `main` HEAD** — all 5 pages, matches live |
| `c82f449` | 2026-08-25 | last change to `phase7-confirm.html` (1-second polling + `FAILED` / `RECONCILIATION_REQUIRED` handling) |
| `e45e4d9` | 2026-08-21 | last change to `phase9-recorder.html` (Pausar/Reanudar controls) |

Known-good SHA-256 (repo `main` @ `38917a7` == live, verified 2026-08-28):

```
43393a34506d23d1c4d1b7400ab42e5587c902f3efb2afa82f8fcda3aa617909  phase9-recorder.html
aa99eb224c0146141376c2f6493ff63d6c01e7b07260afba445ead4717d6eb13  phase7-confirm.html
64d5c5cf1691e5a0b503e7bc605be70f5fdbb6ff59d32e3c626541bdaabf7670  index.html
53cfc2d61c894b83a5a925374599a7c29d301333d7b039d3efda083622cdc3a4  privacy.html
ecd521d0b103ff1f1addf850d9a5c1c6d6b3aad080796dacbe3aaa43dd913bdd  phase9-stt-benchmark-recorder.html
```

## 8. Recovery if the local machine is lost

The repo on GitHub is the source of truth; nothing local is unique once §5 has been followed.

```sh
git clone https://github.com/karenplataaiwork-ctrl/siigo-voiceinvoicing.git \
  /Users/<user>/Documents/AI-Automation-Projects/siigo-voiceinvoicing
cd .../siigo-voiceinvoicing
git log --oneline -5
shasum -a 256 *.html          # compare against §7's known-good list (or the latest recorded set)
```

Then re-establish auth for pushing (§ "Authentication", below). The live site keeps serving
throughout — losing the laptop does not take production down.

To also recover the internal project docs (this file included), they live in a **separate**
location — `…/siigo/docs/` — which is backed up independently; that folder is not part of this repo.

## 9. Recovery if a frontend file is accidentally changed

**Not yet committed / pushed** (local edit only):

```sh
cd /Users/karenplata/Documents/AI-Automation-Projects/siigo-voiceinvoicing
git status
git checkout -- <file>          # discard the local change, restore to HEAD
shasum -a 256 <file>            # confirm it matches the known-good hash (§7)
```

**Committed locally but not pushed:**

```sh
git reset --hard origin/main    # after: git fetch origin
```

**Already pushed (live is now wrong):** treat as a bad deploy → §7 rollback.

**Someone edited on github.com directly:** `git fetch origin` then, if the change is unwanted,
§7 rollback; if wanted, `git reset --hard origin/main` locally to resync, then never do that again
(§ golden rules).

## 10. SHA-256 / git-diff discipline (mandatory)

Every production change must have a recorded **before** and **after**:

- **Before:** `git log --oneline -1` (HEAD) + `shasum -a 256 *.html`.
- **Change:** `git diff` reviewed line-by-line.
- **After push + build:** live hash (`curl … | shasum -a 256`) == local file hash == committed
  blob. Record all three matching in the change note / commit message or a deploy log.

If any pair fails to match, the deploy is not complete — investigate or roll back. A green
"pushed" is not "deployed"; only a matching live hash is.

## 11. Secrets

- This repo is **public**. It must contain **only** static HTML/CSS/JS meant to be world-readable.
- **Never** commit: n8n API keys, the `.mcp.json` file, Siigo `username` / `access_key` / bearer
  tokens / `Partner-Id`, OAuth client secrets, Gmail credentials, `.env` files.
- The pages legitimately contain the **public** n8n webhook URLs
  (`…/webhook/phase9-audio-intake`, `…/webhook/phase7-*`) — those are not secrets.
- A `.gitignore` in the repo blocks the common offenders (`.env*`, `.mcp.json`, `*.pem`, key
  files). It is a safety net, not a licence to keep secrets in the working tree.
- If a secret is ever committed: rotate it immediately (assume it is compromised the moment it
  hits a public repo), then scrub history — a plain revert does **not** remove it from history.

## Authentication (one-time setup for pushing)

`git push` to this repo needs a credential for the `karenplataaiwork-ctrl` GitHub account. As of
2026-08-28 the machine has **no** `gh` CLI, **no** stored git credential helper, and **no** global
git identity. Set up one of:

- **GitHub CLI:** install `gh`, run `gh auth login` (HTTPS, this account), then
  `gh auth setup-git`.
- **Personal Access Token (fine-grained, repo-scoped, contents:read+write):** use it as the
  password on the first `git push` and let the macOS keychain helper store it
  (`git config --global credential.helper osxkeychain`).
- **SSH key:** add a key to the `karenplataaiwork-ctrl` account and set the remote to
  `git@github.com:karenplataaiwork-ctrl/siigo-voiceinvoicing.git`.

Also set a commit identity once: `git config --global user.name "…"` and
`git config --global user.email "…"` (or per-repo).

---

## Provenance

- **Audited & written:** 2026-08-28 (P1-C), read-only.
- **State at audit:** local project `…/siigo/` was **not** a Git repo and had no clone of the
  Pages repo; production HTML in `…/siigo/`, in `origin/main` (`38917a7`), and on the live Pages
  site were **byte-for-byte identical** (SHA-256, all 5 files). Deployment history in the repo is
  all "Add files via upload" / "Update …" web-UI commits — consistent with the known "partly
  manual" deployment issue.
- **This procedure supersedes** any manual upload workflow. Re-verify §1 (Pages settings) and the
  §7 hashes after the first real deploy through this process.
