# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this repository, please report it responsibly.

**DO NOT** create a public GitHub issue for security vulnerabilities.

## Secret Protection

This repository has multiple layers of protection against secret leakage:

### 1. `.gitignore` Rules

The following are automatically ignored:
- `.env` files (all environment files with secrets)
- `config.json` (generated configs with credentials)
- `init-mongo.js` (database initialization with passwords)
- SSL/TLS certificates and keys
- Backup files and database dumps
- Token files and session data

### 2. Pre-commit Hooks

Install pre-commit hooks to catch secrets before they're committed:

```bash
# Install pre-commit
pip install pre-commit

# Install hooks
pre-commit install

# Run manually on all files
pre-commit run --all-files
```

### 3. Git Secrets

Additional protection via git-secrets:

```bash
# macOS
brew install git-secrets

# Configure for this repo
cd /path/to/eMeshCentral
git secrets --install
git secrets --register-aws

# Add custom patterns
git secrets --add 'mongodb://[^:]+:[^@]+@'
git secrets --add 'sessionKey.*[=:]["'"'"'][A-Za-z0-9]{20,}'
```

### 4. Template Files

Sensitive configuration files have `.template` versions that are safe to commit:
- `deploy/config.json.template` ✅ Safe
- `deploy/config.json` ❌ Ignored
- `deploy/env.example` ✅ Safe
- `deploy/.env` ❌ Ignored

## Security Checklist for Contributors

Before committing, verify:

- [ ] No `.env` files in commit
- [ ] No hardcoded passwords, tokens, or API keys
- [ ] No MongoDB connection strings with real credentials
- [ ] No SSL/TLS private keys
- [ ] No database dumps or backups
- [ ] Pre-commit hooks pass

## Sensitive Data Patterns

The following patterns should NEVER appear in commits:

| Pattern | Example | Risk |
|---------|---------|------|
| Real MongoDB URLs | `mongodb://user:pass@host` | Database compromise |
| Session keys | `sessionKey: "abc123..."` | Session hijacking |
| API tokens | `token: "sk_live_..."` | API abuse |
| Invite codes | `meshinstall=xyz123` | Unauthorized access |
| Private keys | `-----BEGIN PRIVATE KEY-----` | TLS compromise |

## What to Do If Secrets Are Committed

If you accidentally commit secrets:

1. **Immediately** rotate the compromised credentials
2. Use BFG Repo-Cleaner or `git filter-branch` to remove from history
3. Force push the cleaned history
4. Notify the security team

```bash
# Example: Remove file from entire git history
bfg --delete-files .env
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

## Deployment Security

When deploying MeshCentral:

1. Generate all secrets using `setup.sh` (never use defaults)
2. Store `.env` securely on the server (mode 600)
3. Use NinjaOne secure variables for agent tokens
4. Rotate tokens monthly
5. Enable MFA for all accounts
6. Review auth.log regularly

## Contact

For security concerns, contact the repository maintainers directly.
