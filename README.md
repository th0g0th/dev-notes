# Git Multi-Identity SSH

Auto-switch SSH keys and git commit identity per GitHub org. No manual switching needed.

## The Problem

You have multiple GitHub accounts (personal, work, client). Each needs its own SSH key and commit identity. By default, git uses one key and one identity for everything.

## The Solution: 3 Files

### File 1: `~/.ssh/config`

Maps host aliases to SSH keys. Both point to `github.com`, but SSH picks a different key based on which **Host** alias is used.

```
# Default — personal key
Host github.com
  HostName github.com
  IdentityFile ~/.ssh/personal-key
  IdentitiesOnly yes

# Alias — work key
Host github.com-work
  HostName github.com
  IdentityFile ~/.ssh/work-key
  IdentitiesOnly yes
```

### File 2: `~/.gitconfig`

Default identity + URL rewrites.

```gitconfig
# Default identity for all repos
[user]
    name = Your Name
    email = personal@email.com

# Force SSH for all GitHub (no HTTPS)
[url "git@github.com:"]
    insteadOf = https://github.com/

# Route Work-Org repos to work SSH key
[url "git@github.com-work:Work-Org/"]
    insteadOf = git@github.com:Work-Org/

# Auto-switch commit identity for Work-Org repos
[includeIf "hasconfig:remote.*.url:*Work-Org*"]
    path = ~/.gitconfig-work
```

The `insteadOf` chain:

```
https://github.com/Work-Org/repo.git
  → git@github.com:Work-Org/repo.git       (rule 1: force SSH)
  → git@github.com-work:Work-Org/repo.git  (rule 2: reroute to work key)
```

The `includeIf` loads a different `[user]` section when the remote URL contains `Work-Org`. Requires Git 2.36+.

### File 3: `~/.gitconfig-work`

Work identity override. Only loaded when `includeIf` matches.

```gitconfig
[user]
    name = Your Full Name
    email = you@company.com
```

## Result

| You clone / push to | SSH key used | Commits as |
|---|---|---|
| `git@github.com:Work-Org/repo.git` | `~/.ssh/work-key` | Your Full Name `<you@company.com>` |
| `git@github.com:personal/repo.git` | `~/.ssh/personal-key` | Your Name `<personal@email.com>` |

Fully automatic — clone, push, fetch all just work.

## Adding Another Org

Every new org = 3 additions:

**1.** Add to `~/.ssh/config`:

```
Host github.com-client
  HostName github.com
  IdentityFile ~/.ssh/client-key
  IdentitiesOnly yes
```

**2.** Add to `~/.gitconfig`:

```gitconfig
[url "git@github.com-client:Client-Org/"]
    insteadOf = git@github.com:Client-Org/

[includeIf "hasconfig:remote.*.url:*Client-Org*"]
    path = ~/.gitconfig-client
```

**3.** Create `~/.gitconfig-client`:

```gitconfig
[user]
    name = Your Name
    email = you@client.com
```

## Verification

```bash
# Test SSH keys
ssh -T git@github.com          # → Hi personal-user!
ssh -T git@github.com-work     # → Hi work-user!

# Test identity in a repo
cd ~/work-project
git config user.name            # → Your Full Name
git config user.email           # → you@company.com

# Test URL rewriting
git ls-remote --heads git@github.com:Work-Org/some-repo.git
# Should authenticate with work key automatically
```

## Requirements

- Git 2.36+ (for `hasconfig:remote` in `includeIf`)
- SSH keys already added to respective GitHub accounts

## License

MIT
