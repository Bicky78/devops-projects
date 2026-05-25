# Bitbucket & Advanced Git Interview Questions

Advanced Git concepts, Bitbucket integration, and real-world scenarios for DevOps interviews — covering workflows, CI/CD integration, and troubleshooting.

---

## Table of Contents

- [Bitbucket Integration Questions](#bitbucket-integration-questions)
- [Advanced Git Workflows](#advanced-git-workflows)
- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [Git Best Practices](#git-best-practices)

---

## Bitbucket Integration Questions

### 1. What is the difference between Bitbucket and GitHub?

**Answer:**

| Feature | Bitbucket | GitHub |
|---------|-----------|--------|
| **Owner** | Atlassian | Microsoft |
| **Free private repos** | Yes (unlimited) | Limited (free tier) |
| **Built-in CI/CD** | Bitbucket Pipelines | GitHub Actions |
| **Jira integration** | Native | Limited |
| **Code review** | Pull requests | Pull requests |
| **Wiki** | Yes | Yes |
| **Issue tracking** | Yes (integrated with Jira) | Yes |

**Choose Bitbucket when:**
- Already using Atlassian ecosystem (Jira, Confluence)
- Need unlimited private repos
- Want built-in CI/CD with Pipelines

---

### 2. How do you set up Git with Bitbucket?

**Answer:**

```bash
# 1. Configure Git user
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 2. Generate SSH key (if not exists)
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"

# 3. Add SSH key to ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa

# 4. Copy public key
cat ~/.ssh/id_rsa.pub

# 5. Add SSH key to Bitbucket
# Bitbucket → Settings → SSH Keys → Add Key

# 6. Clone repository
git clone git@bitbucket.org:workspace/repo.git
```

---

### 3. What are Bitbucket Pipelines?

**Answer:** Bitbucket Pipelines is an integrated CI/CD service that runs builds directly in Bitbucket. It uses YAML configuration files stored in your repository.

**Key features:**
- Integrated with Bitbucket (no external service needed)
- Docker-based build environment
- Parallel steps and stages
- Artifacts and test reports
- Branch-specific configurations
- Deployment variables and secrets

```yaml
# bitbucket-pipelines.yml
pipelines:
  default:
    - step:
        name: Build and Test
        image: maven:3.8.4
        script:
          - mvn clean package
          - mvn test
        artifacts:
          - target/*.jar
```

---

### 4. How do you create a pull request in Bitbucket?

**Answer:**

```bash
# 1. Push feature branch to Bitbucket
git checkout -b feature-x
# Make changes
git add .
git commit -m "Add feature X"
git push origin feature-x

# 2. Create PR via Bitbucket UI
# Bitbucket → Create → Pull request
# Select source: feature-x
# Select destination: main
# Add description, reviewers
```

**Pull request features:**
- Code review with inline comments
- Approvals workflow
- Automatic merge checks
- Build status integration
- Conflict resolution UI

---

### 5. How do you configure Bitbucket Pipelines with multiple environments?

**Answer:**

```yaml
# bitbucket-pipelines.yml
pipelines:
  branches:
    main:
      - step:
          name: Build and Deploy to Production
          image: node:16
          script:
            - npm install
            - npm run build
            - npm run deploy:prod
          deployment: production
    
    develop:
      - step:
          name: Build and Deploy to Staging
          image: node:16
          script:
            - npm install
            - npm run build
            - npm run deploy:staging
          deployment: staging
    
    feature/*:
      - step:
          name: Build and Test
          image: node:16
          script:
            - npm install
            - npm run test
```

---

### 6. What is Bitbucket's branch permissions model?

**Answer:** Bitbucket allows fine-grained branch permissions:

| Permission | Description |
|------------|-------------|
| **Read** | View branch and pull requests |
| **Write** | Push commits and create pull requests |
| **Admin** | Delete branch, force push, edit settings |

**Configuration:**
```bash
# Via UI:
# Repository → Settings → Branch permissions
# Apply restrictions to:
# - Main branch (requires PR approval)
# - Release branches (no force push)
# - Feature branches (anyone can push)
```

---

### 7. How do you integrate Bitbucket with Jira?

**Answer:**

**1. Link repository to Jira project:**
- Bitbucket → Settings → Linked accounts
- Connect Jira instance
- Select Jira project

**2. Use smart commits:**
```bash
# Commit message format
git commit -m "PROJ-123: Fix login issue"

# This automatically:
# - Creates Jira issue link
# - Updates issue status
# - Adds commit to issue activity
```

**3. Branch naming:**
```bash
# Branch format: feature/PROJ-123-description
git checkout -b feature/PROJ-123-login-fix
```

**4. Pull request integration:**
- PR automatically linked to Jira issues
- Build status shown in Jira
- Deployment status updates in Jira

---

### 8. What are Bitbucket deployment variables?

**Answer:** Deployment variables are encrypted environment variables for Pipelines, used for secrets and configuration.

**Types:**
- **Repository variables** - Available to all pipelines
- **Deployment variables** - Available to specific environments
- **Secured variables** - Hidden in logs (for passwords, API keys)

```yaml
# Using variables in pipelines
pipelines:
  default:
    - step:
        script:
          - echo $API_KEY
          - docker login -u $DOCKER_USER -p $DOCKER_PASS
```

**Best practices:**
- Use secured variables for secrets
- Use descriptive names
- Scope variables to specific environments
- Rotate keys regularly

---

## Advanced Git Workflows

### 9. What is the Git Flow workflow?

**Answer:** Git Flow is a branching model for release management with strict branching strategy:

```
main (production)
└── develop (integration)
    ├── feature/* (new features)
    ├── release/* (prepare release)
    └── hotfix/* (critical fixes)
```

**Branch types:**
- **main**: Production-ready code
- **develop**: Integration branch for features
- **feature**: New features (from develop)
- **release**: Prepare release (from develop)
- **hotfix**: Critical fixes (from main)

```bash
# Start feature
git checkout develop
git pull
git checkout -b feature/new-feature
# Work...
git checkout develop
git merge feature/new-feature
git branch -d feature/new-feature

# Start release
git checkout develop
git checkout -b release/v1.2.0
# Prepare release...
git checkout main
git merge release/v1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git checkout develop
git merge release/v1.2.0
```

---

### 10. What is GitHub Flow?

**Answer:** GitHub Flow is a simpler workflow with just two long-lived branches:

```
main (always deployable)
└── feature/* (short-lived branches)
```

**Rules:**
- `main` is always deployable
- Create feature branches from `main`
- Never merge to `main` without tests
- Merge to `main` with pull request
- Deploy immediately after merge

```bash
# GitHub Flow
git checkout main
git pull
git checkout -b feature/authentication
# Work...
git checkout main
git merge feature/authentication
git branch -d feature/authentication
git push origin main
# Deploy immediately
```

---

### 11. What is GitLab Flow?

**Answer:** GitLab Flow combines Git Flow's environment branches with GitHub Flow's simplicity:

```
main (production)
├── develop (staging)
│   └── feature/* (development)
└── production/* (production environments)
```

**Key concepts:**
- Environment branches reflect deployment stages
- Feature branches merge to `develop`
- `develop` deploys to staging
- `main` deploys to production
- Can have multiple production branches

---

### 12. How do you implement a release process with Git tags?

**Answer:**

```bash
# 1. Create release branch
git checkout -b release/v2.1.0 develop

# 2. Update version files
# Update package.json, CHANGELOG.md, etc.

# 3. Commit version bump
git commit -m "Bump version to 2.1.0"

# 4. Merge to main
git checkout main
git merge release/v2.1.0

# 5. Create annotated tag
git tag -a v2.1.0 -m "Release version 2.1.0"

# 6. Merge back to develop
git checkout develop
git merge main

# 7. Push and deploy
git push origin main --tags
git push origin develop

# 8. Deploy from tag
git checkout v2.1.0
```

---

### 13. What is the difference between `git merge --squash` and `git rebase`?

**Answer:**

| Feature | `git merge --squash` | `git rebase` |
|---------|----------------------|--------------|
| **History** | Single commit (squashed) | Linear history (preserves individual commits) |
| **Branch** | Source branch remains | Source branch moves |
| **Conflict resolution** | Once at merge time | For each commit |
| **Original commits** | Lost in target branch | Preserved in source branch |

```bash
# Squash merge
git checkout main
git merge --squash feature-branch
git commit -m "Squashed feature"

# Rebase
git checkout feature-branch
git rebase main
```

---

### 14. How do you implement a hotfix workflow?

**Answer:**

```bash
# 1. Create hotfix branch from main
git checkout main
git pull
git checkout -b hotfix/critical-bug

# 2. Fix the bug
# Make changes...
git commit -m "Fix critical security issue"

# 3. Test thoroughly
# Run tests, manual verification

# 4. Merge to main
git checkout main
git merge hotfix/critical-bug
git tag -a v1.2.1 -m "Hotfix v1.2.1"

# 5. Merge to develop (if needed)
git checkout develop
git merge hotfix/critical-bug

# 6. Delete hotfix branch
git branch -d hotfix/critical-bug

# 7. Deploy immediately
git push origin main --tags
```

---

### 15. What is a Git subtree and how does it differ from submodules?

**Answer:**

| Feature | Submodule | Subtree |
|---------|-----------|---------|
| **Storage** | Separate repository | Merged into main repository |
| **Cloning** | Requires `--recurse-submodules` | No special flags needed |
| **Updates** | `git submodule update` | `git subtree pull` |
| **History** | Separate | Integrated |
| **Size** | Smaller main repo | Larger main repo |

```bash
# Submodule
git submodule add https://github.com/user/library.git libs/library

# Subtree
git subtree add --prefix=libs/library https://github.com/user/library.git main

# Update subtree
git subtree pull --prefix=libs/library https://github.com/user/library.git main
```

---

## Troubleshooting Scenarios

### 16. You accidentally committed sensitive data to a repository. How do you remove it permanently?

**Answer:**

```bash
# 1. Remove sensitive file from history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secrets.txt' \
  --prune-empty --tag-name-filter cat -- --all

# 2. Add to .gitignore
echo "secrets.txt" >> .gitignore
git add .gitignore
git commit -m "Add secrets.txt to .gitignore"

# 3. Force push to overwrite history
git push origin --force --all

# 4. Notify all collaborators
# Everyone must re-clone or reset their local repos
```

**Alternative with BFG Repo-Cleaner:**
```bash
# Download BFG tool
java -jar bfg.jar --delete-files secrets.txt repo.git
cd repo.git
git reflog expire --expire=now --all && git gc --prune=now --force
git push origin --force --all
```

---

### 17. Your local branch is behind remote and you have uncommitted changes. How do you sync?

**Answer:**

```bash
# Option 1: Stash changes, pull, then apply
git stash
git pull origin main
git stash pop

# Option 2: Commit changes, then pull with merge
git add .
git commit -m "WIP"
git pull origin main
# Resolve conflicts if any

# Option 3: Rebase changes onto latest
git add .
git commit -m "WIP"
git fetch origin
git rebase origin/main

# Option 4: Use merge tool for complex conflicts
git mergetool
```

---

### 18. You pushed to wrong branch. How do you fix it?

**Answer:**

```bash
# 1. Move commit to correct branch
git checkout wrong-branch
git reset --hard HEAD~1  # Remove last commit
git checkout correct-branch
git cherry-pick wrong-branch@{1}  # Apply the commit

# 2. Force push to fix remote
git push origin wrong-branch --force
git push origin correct-branch

# Alternative: Move multiple commits
git checkout wrong-branch
git log --oneline  # Find commit hash
git checkout correct-branch
git cherry-pick abc1234..def5678
git checkout wrong-branch
git reset --hard abc1233  # Before the wrong commits
```

---

### 19. You need to undo a commit that was already pushed to shared repository.

**Answer:**

```bash
# Option 1: Revert (safest)
git revert HEAD  # Creates new commit that undoes previous
git push origin main

# Option 2: Reset and force push (dangerous)
git reset --hard HEAD~1
git push origin main --force
# Only if you're sure no one else has pulled

# Option 3: Create new branch from old commit
git checkout -b fixed-version HEAD~1
git push origin fixed-version
# Create PR to merge back
```

---

### 20. How do you recover a deleted branch?

**Answer:**

```bash
# 1. Check reflog for the branch
git reflog

# 2. Find the branch tip
git reflog show --all | grep branch-name

# 3. Recreate branch
git checkout -b branch-name abc1234

# 4. Push to remote
git push origin branch-name

# If branch was deleted from remote only:
git checkout branch-name
git push origin branch-name
```

---

## Git Best Practices

### 21. What are Git best practices for team collaboration?

**Answer:**

**Branch naming:**
```bash
feature/user-authentication
bugfix/login-issue
hotfix/security-patch
release/v2.1.0
```

**Commit messages:**
```bash
# Format: type(scope): description
feat(auth): add OAuth2 integration
fix(api): resolve null pointer exception
docs(readme): update installation instructions
```

**Workflow practices:**
- Pull requests for all changes
- Code review required
- Automated tests must pass
- No direct pushes to main/develop
- Use feature flags for incomplete features
- Regular branch cleanup

---

### 22. How do you optimize Git repository performance?

**Answer:**

**Repository size:**
```bash
# Remove large files from history
git filter-branch --tree-filter 'rm -rf large-files' --prune-empty HEAD

# Use Git LFS for large files
git lfs install
git lfs track "*.zip"
git add .gitattributes

# Clean up old branches
git remote prune origin
```

**Clone performance:**
```bash
# Shallow clone (latest only)
git clone --depth 1 https://github.com/user/repo.git

# Single branch clone
git clone --single-branch --branch main https://github.com/user/repo.git

# Partial clone (Git 2.19+)
git clone --filter=blob:none https://github.com/user/repo.git
```

**Network optimization:**
```bash
# Use SSH instead of HTTPS
git remote set-url origin git@github.com:user/repo.git

# Configure compression
git config --global core.compression 9
```

---

### 23. What security practices should you follow with Git?

**Answer:**

**Repository security:**
- Use SSH keys instead of HTTPS with passwords
- Enable two-factor authentication on GitHub/Bitbucket
- Use signed commits (GPG)
- Regularly rotate SSH keys
- Use branch permissions to protect main branches

**Code security:**
- Never commit secrets (passwords, API keys)
- Use environment variables or secret management
- Scan for sensitive data with tools like `git-secrets`
- Use `.gitignore` properly

**Example signed commit:**
```bash
# Configure GPG signing
git config --global commit.gpgsign true
git config --global user.signingkey ABC12345

# Sign commit
git commit -S -m "Signed commit"
```

---

### 24. How do you handle merge conflicts in large teams?

**Answer:**

**Prevention:**
- Small, frequent commits
- Clear feature boundaries
- Regular integration (avoid long-lived branches)
- Use feature flags to isolate incomplete work
- Automated tests to catch conflicts early

**Resolution process:**
```bash
# 1. Pull latest changes before starting
git pull origin main

# 2. Use merge tools
git mergetool

# 3. Communicate with conflicting author
# Discuss resolution approach

# 4. Test thoroughly after merge
# Run full test suite

# 5. Document conflict resolution
# Add comments for future reference
```

---

### 25. What is the recommended workflow for releasing software with Git?

**Answer:**

**Release process:**
```bash
# 1. Prepare release
git checkout develop
git pull
git checkout -b release/v2.1.0

# 2. Update version and changelog
# Update version files
# Add release notes

# 3. Test thoroughly
# Run full test suite
# Manual QA

# 4. Create release
git checkout main
git merge release/v2.1.0
git tag -a v2.1.0 -m "Release v2.1.0"

# 5. Deploy
git push origin main --tags
# Deploy to production

# 6. Update develop
git checkout develop
git merge main

# 7. Clean up
git branch -d release/v2.1.0
```

**Best practices:**
- Semantic versioning (MAJOR.MINOR.PATCH)
- Annotated tags with release notes
- Automated testing before release
- Rollback plan
- Release notes for stakeholders
- Post-release monitoring
