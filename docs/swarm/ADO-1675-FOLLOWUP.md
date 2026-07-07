# ADO-1675-FOLLOWUP

## Goal

Close the remaining edges of the ADO-1669 public discovery backlink mesh on the
`xrmghost-skills` README that were out of scope for the prior ADO-1682 task
(which only added `docs.xrmghost.tech` links). Add first-party links to:

- `www.xrmghost.tech` (product marketing site)
- `github.com/xrmghost/xrmghost` (base repo / suite index)
- `github.com/xrmghost/xrmghost-docs` (docs source repo)
- `github.com/xrmghost/xrmghost-attributes` (sibling library)
- `nuget.org/packages/XrmGhost.Attributes` (the pairing NuGet package)

## Scope and modifications

- Only `README.md` was touched.
- Added a new `## Related resources` section immediately before `## License`,
  after the existing `SECURITY.md` reference, so it doesn't disturb the
  docs.xrmghost.tech links already present from the prior cycle.
- Each of the 5 links has distinct, non-duplicated copy explaining why a
  reader would want to follow it (marketing overview vs. suite index vs.
  docs source vs. sibling library vs. packaged NuGet distribution).
- No skill prompts, triggers, `CONTRIBUTING.md`, `SECURITY.md`, or
  `docs/adr/**` files were modified.

## Commands executed

```
git diff -- README.md
rg -n 'https://' README.md
git status --porcelain
```

Additionally, verified all 5 new links resolve with HTTP 200 via `curl -L`:

```
200 https://www.xrmghost.tech
200 https://github.com/xrmghost/xrmghost
200 https://github.com/xrmghost/xrmghost-docs
200 https://github.com/xrmghost/xrmghost-attributes
200 https://www.nuget.org/packages/XrmGhost.Attributes
```

## Acceptance criteria verified

- [x] README.md now includes first-party links to www, base repo, docs repo,
      attributes repo, and the Attributes NuGet package.
- [x] Copy is differentiated per link (no duplicated wording).
- [x] No skill behavior or other tracked files changed (`git status --porcelain`
      shows only `README.md` modified).

## Residual risks

- None identified for this change; it is purely editorial/link-topology.
  Link liveness was verified at time of writing but is not continuously
  monitored (same caveat as the prior ADO-1682 cycle).

## Notes for review

- The new section sits after the Security Policy line and before License,
  keeping the existing docs.xrmghost.tech-focused paragraphs untouched.
- Task was scoped strictly to README.md; no other files in
  `files_not_to_touch` were opened or modified.
