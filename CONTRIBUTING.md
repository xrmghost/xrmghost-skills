# Contributing to xrmghost-skills

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

If you want to inspect the SARIF shape locally, use the same workflow in GitHub Actions or reuse the conversion step from `.github/workflows/skill-lint.yml` after generating `skill-lint-results.json`.

### Where to see results in GitHub

- **Pull request checks:** the required `skill-lint` check shows pass/fail status and inline annotations.
- **Security tab:** historical SARIF uploads are published under **Security → Code scanning** with category `skill-spec`.
