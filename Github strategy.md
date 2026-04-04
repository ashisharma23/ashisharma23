Here's a comprehensive GitHub repository management strategy built on industry best practices. Let me first lay out the architecture visually, then walk through every command and step.

The core answer to your main question: **separate repos for each project type**. One master repo creates lifecycle conflicts — a sandbox experiment doesn't need the same branching protection as a production project, and templates need a completely different versioning philosophy. Separate repos with a consistent naming convention is the right call.Now let's walk through the complete setup — from GitHub profile to local machine — step by step.

---

## Part 1 — Naming convention & repo taxonomy

The naming convention is `{type}-{descriptor}`, always lowercase with hyphens. This makes repos sortable, scannable, and immediately clear in purpose.

| Prefix | Purpose | Visibility | Lifecycle |
|---|---|---|---|
| `personal-` | dotfiles, CV, home configs | Private | Long-lived, maintained |
| `lab-` | experiments, spikes, POCs | Private or Public | Short-lived, archive when done |
| `sandbox-` | play, tutorials, scratch | Private | Ephemeral, delete freely |
| `template-` | boilerplates, starter kits | Public | Versioned with releases |
| `project-` | real apps, OSS, products | Public | Full SDLC, protected branches |
| `course-` | learning, certs, study notes | Private | Chapter-based branches |

---

## Part 2 — Set up your local directory structure

Run these commands once to create your local mirror:

```bash
# Create the root workspace
mkdir -p ~/github/{personal,lab,sandbox,templates,projects,courses}

# Verify the structure
tree ~/github
# or if tree isn't installed:
ls -la ~/github/
```

Now configure git globally (one-time setup):

```bash
# Identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Sensible defaults
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global core.autocrlf input        # mac/linux; use 'true' on Windows
git config --global core.editor "code --wait"  # or vim, nano, etc.

# Useful aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --decorate --all"
```

---

## Part 3 — Create repos on GitHub via CLI (recommended)

Install the GitHub CLI if you haven't:

```bash
# macOS
brew install gh

# Windows
winget install --id GitHub.cli

# Linux
sudo apt install gh   # Debian/Ubuntu
```

Authenticate once:

```bash
gh auth login
# Follow prompts → GitHub.com → HTTPS → Login with browser
```

---

## Part 4 — Create each repo type with correct settings

### `personal-` repos

```bash
cd ~/github/personal

# Create and clone in one step
gh repo create personal-dotfiles \
  --private \
  --description "My dotfiles, shell configs, and dev environment setup" \
  --clone

cd personal-dotfiles
git commit --allow-empty -m "chore: initial commit"
git push -u origin main
```

### `lab-` repos (experiment lifecycle)

```bash
cd ~/github/lab

gh repo create lab-rust-async-spike \
  --private \
  --description "Spike: async Rust patterns with Tokio" \
  --clone

cd lab-rust-async-spike

# Add a README that documents the experiment goal
cat > README.md << 'EOF'
# lab-rust-async-spike

**Status:** 🧪 Active experiment  
**Goal:** Evaluate Tokio async patterns for project-myapp  
**Started:** 2025-01-15  
**Conclusion:** (fill when done)

## Hypothesis
...

## Results
...
EOF

git add README.md
git commit -m "docs: experiment brief"
git push -u origin main

# When experiment is done — archive it (never delete; GitHub archive is free)
# Go to repo Settings → Archive repository
# Or via CLI:
gh repo archive lab-rust-async-spike
```

### `sandbox-` repos (ephemeral)

```bash
cd ~/github/sandbox

gh repo create sandbox-next14-play \
  --private \
  --description "Playing with Next.js 14 app router" \
  --clone

cd sandbox-next14-play
# Work freely, no PR discipline needed
# Delete when done:
# gh repo delete sandbox-next14-play --yes
```

### `template-` repos (enable Template Repository)

```bash
cd ~/github/templates

gh repo create template-react-ts \
  --public \
  --description "Production-ready React + TypeScript starter" \
  --clone

cd template-react-ts

# Mark as a GitHub Template Repository (makes the "Use this template" button appear)
gh repo edit template-react-ts --template=true

# Template repos use semantic versioning
git tag v1.0.0
git push origin v1.0.0

# Create a GitHub Release from that tag
gh release create v1.0.0 \
  --title "v1.0.0 – Initial release" \
  --notes "First stable version of the React TypeScript template"
```

### `project-` repos (full SDLC)

```bash
cd ~/github/projects

gh repo create project-myapp-api \
  --public \
  --description "REST API for MyApp — Node.js + Express + PostgreSQL" \
  --clone

cd project-myapp-api

# Set up branch protection on main (requires push rights)
gh api repos/:owner/project-myapp-api/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":[]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null

# Create standard long-lived branches
git checkout -b develop
git push -u origin develop

# Add a .gitignore
gh api repos/:owner/project-myapp-api/contents/.gitignore \
  --method PUT \
  --field message="chore: add gitignore" \
  --field content=$(curl -s https://raw.githubusercontent.com/github/gitignore/main/Node.gitignore | base64)
```

**Branch workflow for projects:**

```bash
# Feature work — always branch from develop
git checkout develop
git pull origin develop
git checkout -b feat/user-authentication

# Work, commit with conventional commits
git commit -m "feat: add JWT middleware"
git commit -m "test: add auth route tests"
git commit -m "docs: update API docs for /auth"

# Push and open PR into develop
git push -u origin feat/user-authentication
gh pr create \
  --base develop \
  --title "feat: user authentication" \
  --body "Closes #12"

# Release: merge develop → main via PR only
gh pr create \
  --base main \
  --head develop \
  --title "release: v1.2.0"
```

### `course-` repos (chapter-based branching)

```bash
cd ~/github/courses

gh repo create course-aws-saa-2024 \
  --private \
  --description "AWS Solutions Architect Associate study notes and labs" \
  --clone

cd course-aws-saa-2024

# One branch per chapter/module
git checkout -b ch01-cloud-fundamentals
git push -u origin ch01-cloud-fundamentals

git checkout -b ch02-iam-security
git push -u origin ch02-iam-security

# Tag when you complete the full course
git checkout main
git merge ch01-cloud-fundamentals --no-ff -m "complete: chapter 01"
git tag completed-ch01
git push origin --tags
```

---

## Part 5 — Clone any existing repo into the right folder

```bash
# Always clone into the matching category folder
cd ~/github/projects
gh repo clone your-username/project-myapp-api

cd ~/github/lab
gh repo clone your-username/lab-old-experiment
```

---

## Part 6 — Maintain a profile README

Your GitHub profile (`github.com/username`) can have a pinned README that acts as a nav for visitors:

```bash
gh repo create your-username \
  --public \
  --description "GitHub profile README" \
  --clone

cd your-username
cat > README.md << 'EOF'
## Hi, I'm [Name]

| Type | Prefix | What's here |
|---|---|---|
| 🔧 Personal | `personal-*` | Dotfiles & configs |
| 🧪 Lab | `lab-*` | Experiments & spikes |
| 📦 Templates | `template-*` | Reusable starters |
| 🚀 Projects | `project-*` | Production work |
| 📚 Courses | `course-*` | Learning notes |
EOF

git add README.md && git commit -m "docs: profile README"
git push -u origin main
```

---

## Part 7 — Maintenance habits

```bash
# Weekly: check for stale lab/sandbox repos
gh repo list --limit 100 --json name,updatedAt \
  | jq '.[] | select(.name | startswith("lab-")) | {name, updatedAt}'

# Archive any lab repo older than 3 months with no commits
gh repo archive lab-old-experiment

# Monthly: update template repos
cd ~/github/templates/template-react-ts
git pull && npm update
git commit -am "chore: update dependencies"
git tag v1.x.x && git push origin --tags

# Sync all project repos at once (run from ~/github/projects)
for dir in ~/github/projects/*/; do
  echo "Pulling $dir..."
  git -C "$dir" pull --rebase 2>/dev/null || echo "  ⚠ skipped (not a git repo)"
done
```

---

## Summary — Decision rules

- **One repo per concern** — lifecycles differ, so separation prevents branch policy conflicts and keeps your profile clean.
- **Prefix-first naming** — your repos sort into logical groups automatically on GitHub.
- **Archive, don't delete** — lab and sandbox repos have archaeological value; GitHub archiving is free and keeps them searchable.
- **Template repos get releases** — use semantic versioning so consumers know when to upgrade.
- **Project repos get branch protection** — never push directly to `main`; PRs are the gate.
- **Course repos use branches as chapters** — merge to `main` only on completion and tag it.