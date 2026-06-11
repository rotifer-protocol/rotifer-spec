# Contributing to rotifer-spec

Thank you for your interest in contributing to the Rotifer Protocol.

For an overview of this repository, see [`README.md`](./README.md). For the
broader Rotifer Protocol ecosystem, visit [rotifer.dev](https://rotifer.dev).

## Workflow

### Branch Naming

- `feat/description` — new features
- `fix/description` — bug fixes
- `docs/description` — documentation changes
- `refactor/description` — code refactoring
- `test/description` — test changes

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(scope): add new capability
fix(scope): correct edge case in X
docs(readme): update install section
```

### Pull Requests

1. Fork the repository
2. Create a feature branch from `main`
3. Make your changes
4. Ensure all checks pass (tests / lint / type-check, as applicable)
5. **Sign off your commits** (see DCO section below)
6. Open a PR using the provided template

## Developer Certificate of Origin (DCO)

All contributions to this project must be **signed off**, certifying
compliance with the [Developer Certificate of Origin v1.1](https://developercertificate.org/).

### How to Sign Off

Sign your commits with the `-s` flag:

```bash
git commit -s -m "your commit message"
```

This automatically appends a `Signed-off-by:` line to the commit message:

```
Signed-off-by: Your Name <your@email.com>
```

If you forgot to sign off and need to amend the last commit:

```bash
git commit --amend -s --no-edit
```

For multiple commits, use rebase with the `--signoff` flag:

```bash
git rebase HEAD~N --signoff   # N = number of commits to sign off
```

### Why DCO

By signing off, you certify that:

1. You wrote the contribution (or have the right to submit it under the
   project's license);
2. You understand the contribution is public and the sign-off is recorded
   permanently in the project's git history;
3. You agree to the [Developer Certificate of Origin v1.1](https://developercertificate.org/) (full text).

### Enforcement

Pull requests with any unsigned commits will be **blocked** by the automated
DCO check (`.github/workflows/dco.yml`). Please sign all commits before
opening a PR.

## License

By contributing, you agree that your contributions will be licensed under
**CC BY-SA 4.0** — see [`LICENSE`](./LICENSE) for the full text.

For AGPL+dual repositories: commercial licensing inquiries go to
`dev@rotifer.dev`, who can provide the commercial license template and terms.
