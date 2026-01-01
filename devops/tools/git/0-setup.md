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

