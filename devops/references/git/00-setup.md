# Initialize Local Repo & Connect to Remote

```bash
cd /path/to/my-repo
git init
git branch -M main  # set default branch to main
```

Create project files then:
```bash
git add .
git commit -m "initial commit"
```

Add the remote and push:
```bash
git remote add origin git@github.com:USERNAME/remote-repo.git
git push -u origin main
```
- The `-u` flag links `main` to `origin/main` for future pushes


# One-Time SSH Setup

## 1. Check Existing Setup

```bash
ssh -T git@github.com   # if authenticated, it's all good
ls -al ~/.ssh           # look for id_ed25519 and id_ed25519.hub
```

- If `id_ed25519` already exists and SSH auth works, skip the following steps

## 2. Create a New SSH Key (ed25519)

```bash
ssh-keygen -t ed25519 -C "email@address.com"
```

- Accept the default location: `~/.ssh/id_ed25519`
- Set a passphrase if desired

Start the SSH agent and add the key:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Print the public key
```bash
cat ~/.ssh/id_ed25519.pub
```

## 3. Add Key to GitHub

1. Copy the printed key
2. GitHub → Settings → SSH and GPG keys → New SSH key
3. Paste the key and save

Verify
```bash
ssh -T git@github.com
```

# Other Modifications

## Changed Repo Name

Update the remote URL locally once:
```bash
git remote -v
git remote set-url origin https://github.com/<user>/<new-repo-name>.git
```

`git remote -v`
- lists each remote name (e.g., `origin`)
- AND the URLs associated with it
- AND whether each URL is used for **fetch** or **push**

# 🆕 Multi-Environment SSH Setup (Advanced)

> Use this when working across multiple systems (e.g., company_1, company_2, Personal) with separate SSH keys and clean boundaries.

## 🧠 Key Difference from Current Workflow

Your current setup assumes:
> One SSH key → one GitHub connection → all repos use git@github.com

This new setup changes that to:
> Multiple SSH keys → mapped via aliases → each repo explicitly selects its key

## 🔑 1. Create Separate SSH Keys

Instead of a single default key:
```bash
ssh-keygen -t ed25519 -C "email@address.com"
```

Create one key per environment:
```bash
ssh-keygen -t ed25519 -C "zhe@company_1" -f ~/.ssh/id_ed25519_company_1
ssh-keygen -t ed25519 -C "zhe@company_2" -f ~/.ssh/id_ed25519_company_2
ssh-keygen -t ed25519 -C "zhe@personal" -f ~/.ssh/id_ed25519_personal
```


## 🔑 2. Add Each Key to GitHub

Same as before:
```bash
cat ~/.ssh/id_ed25519_company_1.pub
```
- Add each `.pub` key to GitHub
- Name clearly:
    - `company_1-laptop`
    - `company_2-laptop`
    - `personal-laptop`

## ⚙️ 3. Configure SSH Aliases

Create/edit:
```bash
notepad ~/.ssh/config
```

Add:
```
Host github-company_1
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_company_1
  IdentitiesOnly yes

Host github-company_2
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_company_2
  IdentitiesOnly yes

Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  IdentitiesOnly yes
```

## 🔗 4. Clone Repos Using Alias

❌ Old workflow (single key)
```bash
git clone git@github.com:USERNAME/remote-repo.git
```
✅ New workflow (explicit key selection)
```bash
git clone git@github-company_1:company_1/company_1-platform.git
```

## 🔄 5. Update Existing Repo Remote

Check current remote:
```bash
git remote -v
```
Update to use alias:
```bash
git remote set-url origin git@github-company_1:company_1/company_1-platform.git
```

## 👤 6. Set Repo-Specific Identity

SSH controls authentication.  
Git config controls commit identity.

Set per repo:
```bash
git config user.name "Zhe Rao"
git config user.email "your-company_1-email@example.com"
```


## 🧠 Mental Model
| Component               | Responsibility                 |
| ----------------------- | ------------------------------ |
| SSH key                 | proves device identity         |
| SSH config alias        | selects which key to use       |
| Git remote URL          | determines which alias is used |
| Git config (name/email) | controls commit identity       |


## ⚠️ Important Rules
- Never reuse SSH keys across environments
- Never copy keys between machines
- Always use alias-based remotes for company repos
- Keep each environment isolated

## 🧭 Summary: What Changed from Original Setup
| Aspect            | Original              | New                      |
| ----------------- | --------------------- | ------------------------ |
| SSH keys          | Single (`id_ed25519`) | Multiple per environment |
| GitHub connection | `git@github.com`      | `git@github-<env>`       |
| Key selection     | Automatic/default     | Explicit via alias       |
| Repo isolation    | Implicit              | Strongly enforced        |
| Risk of cross-use | Medium                | Very low                 |


## 🧭 One-line takeaway

> In multi-environment setups, **the remote URL chooses the SSH key**, not Git.

