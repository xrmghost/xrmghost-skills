# Contributing to xrmghost-skills

Thank you for contributing! Please read this guide before submitting changes.

## Prerequisites

- Git 2.34+
- A GitHub account with access to the `xrmghost` organisation

---

## Commit Signing

All commits merged to `main` **must be cryptographically signed**. Branch protection has `required_signatures: true` enabled — unsigned commits will be rejected.

You can sign with either **GPG** or **SSH**. Choose one and configure it once.

---

### Option A — GPG Signing

#### 1. Generate a GPG key (skip if you already have one)

```bash
gpg --full-generate-key
# Choose: RSA and RSA (default), 4096 bits, no expiry (or a date you'll remember)
# Enter your name and the email address registered with your GitHub account
```

#### 2. Get your key ID

```bash
gpg --list-secret-keys --keyid-format=long
```

Example output:
```
sec   rsa4096/3AA5C34371567BD2 2024-01-01 [SC]
```

The key ID is the part after the slash: `3AA5C34371567BD2`.

#### 3. Export the public key and add it to GitHub

```bash
gpg --armor --export 3AA5C34371567BD2
```

Copy the output. In GitHub → **Settings → SSH and GPG keys → New GPG key**, paste it and save.

#### 4. Configure Git to use the key

```bash
git config --global user.signingkey 3AA5C34371567BD2
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

#### 5. Verify

```bash
echo "test" | gpg --clearsign
git commit --allow-empty -m "test: verify gpg signing"
git log --show-signature -1
```

You should see `gpg: Good signature from ...` in the log output.

---

### Option B — SSH Signing (Git ≥ 2.34)

SSH signing is simpler if you already authenticate to GitHub with an SSH key.

#### 1. Ensure you have an SSH key

```bash
ls ~/.ssh/*.pub
# If none: ssh-keygen -t ed25519 -C "your@email.com"
```

#### 2. Add the key to GitHub as a **Signing Key**

In GitHub → **Settings → SSH and GPG keys → New SSH key**, set **Key type** to **Signing Key** and paste the public key.

> Note: a key used for authentication and one used for signing can be the same key, but GitHub requires it to be added separately under the "Signing Key" type.

#### 3. Configure Git

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub   # adjust path as needed
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

To allow Git to verify SSH signatures locally, create an allowed-signers file:

```bash
echo "your@email.com $(cat ~/.ssh/id_ed25519.pub)" >> ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

#### 4. Verify

```bash
git commit --allow-empty -m "test: verify ssh signing"
git log --show-signature -1
```

You should see `Good "git" signature for your@email.com` in the output.

---

### Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `error: gpg failed to sign the data` | GPG agent not running | `gpg-agent --daemon` or restart terminal |
| `secret key not available` | Wrong key ID configured | Re-run `gpg --list-secret-keys` and update `user.signingkey` |
| `SSH key not found` | Wrong path in `user.signingkey` | Use the full absolute path to the `.pub` file |
| GitHub rejects signed commit | Signing key not added to GitHub account | Add public key in GitHub Settings |

---

## Future Work

- **Build provenance attestation** (`attest-build-provenance`) via GitHub Actions — not currently implemented because xrmghost-skills has no formal release artefacts. When a release pipeline is added, SLSA provenance attestation should be wired into the workflow.

---

## Pull Request Guidelines

- Target `main`.
- One logical change per PR.
- Ensure all commits in the PR are signed (see above).
- Reference the relevant ADO work item in the PR description.
