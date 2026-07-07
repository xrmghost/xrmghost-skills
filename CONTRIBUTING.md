# Contributing to xrmghost-skills

Thank you for contributing! Please read this guide before submitting changes.

Before touching `skills/*/SKILL.md`, it helps to know what these skills are actually standing in for. They encode the local `xg run` authoring/debugging loop described at **[docs.xrmghost.tech/skills/using-skills/](https://docs.xrmghost.tech/skills/using-skills/)**, against the command surface documented in the **[CLI reference](https://docs.xrmghost.tech/cli/reference/)** — a change here that drifts from either is a behavior bug even if the linter still passes. If your contribution is a product bug or feature idea rather than skill wording, file it against the product itself via **[Report a Bug](https://docs.xrmghost.tech/community/report-a-bug/)** or **[Request a Feature](https://docs.xrmghost.tech/community/request-a-feature/)** instead of a PR here.

## Prerequisites

- Git 2.34+
- A GitHub account with access to the `xrmghost` organisation

---

## Spec Linting

All pull requests targeting `main` are validated by the `skill-lint` GitHub Actions workflow.

The workflow uses [`agent-ecosystem/skill-validator`](https://github.com/agent-ecosystem/skill-validator) in structure-validation mode so that valid skills pass cleanly and invalid specs fail as a blocking status check.

### Run the linter locally

```bash
mkdir -p tools && \
curl -sSL https://github.com/agent-ecosystem/skill-validator/releases/download/v1.5.6/skill-validator_1.5.6_linux_amd64.tar.gz \
  | tar -xz -C tools && \
chmod +x tools/skill-validator && \
./tools/skill-validator validate structure -o json skills > skill-lint-results.json
```

To inspect the generated JSON quickly:

```bash
python3 -m json.tool skill-lint-results.json
```

### Generate and read SARIF locally

The CI workflow converts `skill-lint-results.json` into `results.sarif` before uploading it to GitHub code scanning.

- `ruleId` identifies the violated validator rule.
- `message.text` contains the human-readable error.
- `locations[].physicalLocation` points to the affected skill file and line used for inline annotations.

### Where to see results in GitHub

- **Pull request checks:** the required `skill-lint` check shows pass/fail status and inline annotations.
- **Security tab:** historical SARIF uploads are published under **Security → Code scanning** with category `skill-spec`.

---

## Commit Signing

All commits merged to `main` **must be cryptographically signed**. Branch protection has `required_signatures: true` enabled — unsigned commits will be rejected.

You can sign with either **GPG** or **SSH**. Choose one and configure it once.

### Option A — GPG Signing

```bash
# 1. Generate key
gpg --full-generate-key

# 2. Get key ID
gpg --list-secret-keys --keyid-format=long

# 3. Export and add to GitHub Settings → SSH and GPG keys → New GPG key
gpg --armor --export <KEY_ID>

# 4. Configure git
git config --global user.signingkey <KEY_ID>
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# 5. Verify
git log --show-signature -1
```

### Option B — SSH Signing (Git ≥ 2.34)

```bash
# 1. Add existing SSH key to GitHub Settings → SSH keys → New SSH key (type: Signing Key)

# 2. Configure git
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# 3. Allowed signers (for local verification)
echo "your@email.com $(cat ~/.ssh/id_ed25519.pub)" >> ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

### Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `gpg failed to sign the data` | GPG agent not running | `gpg-agent --daemon` |
| `secret key not available` | Wrong key ID | Re-run `gpg --list-secret-keys` |
| GitHub rejects signed commit | Key not added to GitHub | Add public key in GitHub Settings |

---

## Future Work

- **Build provenance attestation** (`attest-build-provenance`) — not yet implemented (no formal releases). Planned for when a release pipeline is added.

---

## Pull Request Guidelines

- Target `main`.
- One logical change per PR.
- Ensure all commits in the PR are signed.
- Reference the relevant ADO work item in the PR description.
