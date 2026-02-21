# Day 26 – GitHub CLI: Manage GitHub from Your Terminal

## Task 1: Install and Authenticate

### Installation & Setup

# Linux (Debian/Ubuntu)
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh


# Authenticate
gh auth login

# Check auth status
gh auth status

# View current user
gh auth status | grep "Logged in"
```

### What authentication methods does gh support?

1. **OAuth via web browser** (recommended)
   - Opens browser, you authorize the app
   - Most user-friendly method

2. **Personal Access Token (PAT)**
   - Generate token from GitHub settings
   - Paste it when prompted
   - Useful for servers/CI environments

3. **SSH keys**
   - Uses existing SSH keys
   - Good if you already use SSH for Git

4. **GitHub App authentication**
   - For organization-level automation
   - More granular permissions

---

## Task 2: Working with Repositories

### Commands Used
```bash
# Create new repo
gh repo create my-test-repo --public --description "Testing gh CLI" --clone

# Alternative: create with README
gh repo create my-test-repo --public --readme

# Clone using gh (interactive picker if no repo specified)
gh repo clone username/repo-name

# View repo details
gh repo view username/repo-name

# List all your repos
gh repo list

# List repos with filters
gh repo list --limit 10
gh repo list --visibility public
gh repo list --language python

# Open repo in browser
gh repo view --web
gh repo view username/repo-name --web

# Delete repo
gh repo delete username/my-test-repo --yes
```

### Observations

- `gh repo create` automatically sets up remote and can clone in one command
- Much faster than going to GitHub web interface
- `gh repo view` shows stars, description, README preview right in terminal
- Delete requires confirmation unless you use `--yes` flag

---

## Task 3: Issues

### Commands Used
```bash
# Create issue
gh issue create --title "Bug: Login fails" --body "User cannot login with valid credentials" --label bug

# Interactive create (prompts for title, body, etc.)
gh issue create

# List all open issues
gh issue list

# List with filters
gh issue list --state all
gh issue list --label bug
gh issue list --assignee @me

# View specific issue
gh issue view 42

# View issue in browser
gh issue view 42 --web

# Close issue
gh issue close 42

# Close with comment
gh issue close 42 --comment "Fixed in PR #43"

# Reopen issue
gh issue reopen 42

# Edit issue
gh issue edit 42 --add-label "help wanted"
```

### How could you use gh issue in a script or automation?

1. **Automated bug reporting from logs**
```bash
   # Script that parses error logs and creates issues
   if grep -q "ERROR" app.log; then
     gh issue create --title "Error detected in logs" \
       --body "$(tail -n 50 app.log)" \
       --label "automated,bug"
   fi
```

2. **CI/CD failure notifications**
```bash
   # Create issue when build fails
   gh issue create --title "Build failed: $BUILD_ID" \
     --body "Build logs: $BUILD_URL" \
     --label "ci-failure"
```

3. **Scheduled issue creation**
```bash
   # Weekly reminder to update dependencies
   gh issue create --title "Weekly: Update dependencies" \
     --body "Run npm update and test" \
     --assignee devops-team
```

4. **Bulk issue management**
```bash
   # Close all issues with specific label
   gh issue list --label "wontfix" --json number --jq '.[].number' | \
     xargs -I {} gh issue close {}
```

5. **Issue metrics/reporting**
```bash
   # Count open bugs
   gh issue list --label bug --json number --jq 'length'
```

---

## Task 4: Pull Requests

### Complete PR Workflow from Terminal
```bash
# Create branch and make changes
git checkout -b feature/add-documentation
echo "# New docs" >> README.md
git add .
git commit -m "docs: add new documentation section"
git push -u origin feature/add-documentation

# Create PR
gh pr create --title "Add documentation" --body "This PR adds missing docs"

# Or use --fill to auto-populate from commits
gh pr create --fill

# Interactive PR creation
gh pr create

# List all open PRs
gh pr list

# List with filters
gh pr list --state all
gh pr list --author @me
gh pr list --label "needs-review"

# View PR details
gh pr view 15

# View PR in browser
gh pr view 15 --web

# Check PR status (checks, reviews, etc.)
gh pr checks

# View diff
gh pr diff 15

# Checkout PR locally
gh pr checkout 15

# Review PR
gh pr review 15 --approve
gh pr review 15 --comment --body "LGTM"
gh pr review 15 --request-changes --body "Please fix the tests"

# Merge PR
gh pr merge 15

# Merge with specific method
gh pr merge 15 --merge    # Standard merge commit
gh pr merge 15 --squash   # Squash and merge
gh pr merge 15 --rebase   # Rebase and merge

# Auto-merge and delete branch
gh pr merge 15 --squash --delete-branch
```

### What merge methods does gh pr merge support?

1. **--merge** (default)
   - Creates a merge commit
   - Preserves all commits from the branch
   - Shows full history in the log

2. **--squash**
   - Combines all commits into one
   - Cleaner main branch history
   - Loses individual commit details

3. **--rebase**
   - Replays commits on top of base branch
   - Linear history
   - No merge commit

4. **--admin**
   - Override branch protection rules
   - Requires admin permissions

### How would you review someone else's PR using gh?
```bash
# View the PR
gh pr view 42

# Check out the PR locally to test
gh pr checkout 42

# Run tests locally
npm test

# View the diff
gh pr diff 42

# Review options:

# 1. Approve
gh pr review 42 --approve --body "Looks good, tests pass"

# 2. Request changes
gh pr review 42 --request-changes --body "Please add unit tests"

# 3. Comment only
gh pr review 42 --comment --body "Consider refactoring the error handling"

# Add single comment to specific code
gh pr comment 42 --body "This function could be simplified"
```

---

## Task 5: GitHub Actions & Workflows

### Commands Used
```bash
# List workflow runs
gh run list

# List runs for specific workflow
gh run list --workflow=ci.yml

# View specific run
gh run view 123456

# View run logs
gh run view 123456 --log

# Watch a running workflow
gh run watch

# Rerun failed jobs
gh run rerun 123456

# List workflows
gh workflow list

# View workflow details
gh workflow view ci.yml

# Enable/disable workflow
gh workflow enable ci.yml
gh workflow disable ci.yml

# Trigger workflow manually
gh workflow run ci.yml
```

### How could gh run and gh workflow be useful in a CI/CD pipeline?

1. **Monitoring build status**
```bash
   # Check if latest CI run passed
   if gh run list --workflow=ci.yml --limit 1 --json conclusion --jq '.[0].conclusion' | grep -q "success"; then
     echo "Build passed, proceeding with deployment"
   else
     echo "Build failed, aborting"
     exit 1
   fi
```

2. **Automated retries**
```bash
   # Auto-retry failed runs
   gh run list --status failure --json databaseId --jq '.[].databaseId' | \
     xargs -I {} gh run rerun {}
```

3. **Deployment orchestration**
```bash
   # Trigger production deployment after staging passes
   gh workflow run deploy-prod.yml
```

4. **Build notifications**
```bash
   # Send Slack notification when build completes
   STATUS=$(gh run view --json conclusion --jq '.conclusion')
   curl -X POST $SLACK_WEBHOOK -d "{\"text\": \"Build $STATUS\"}"
```

5. **Quality gates**
```bash
   # Block deployment if tests fail
   gh run list --workflow=tests.yml --limit 1 --json conclusion --jq '.[0].conclusion'
```

---

## Task 6: Useful gh Tricks

### gh api - Raw API Calls
```bash
# Get rate limit status
gh api rate_limit

# Get user info
gh api user

# List repos with custom query
gh api /user/repos --paginate

# Get specific repo data
gh api repos/username/repo-name

# Create label via API
gh api repos/username/repo/labels -f name="priority" -f color="ff0000"
```

**Use case:** Access GitHub features not yet in gh CLI, custom automation scripts.

---

### gh gist - Manage Gists
```bash
# Create gist from file
gh gist create script.sh

# Create private gist
gh gist create --secret notes.md

# Create gist from stdin
echo "Quick note" | gh gist create -

# List your gists
gh gist list

# View gist
gh gist view abc123

# Edit gist
gh gist edit abc123

# Delete gist
gh gist delete abc123
```

**Use case:** Quick code sharing, save snippets, share logs with team.

---

### gh release - Manage Releases
```bash
# Create release
gh release create v1.0.0 --title "Version 1.0.0" --notes "Initial release"

# Create release with assets
gh release create v1.0.0 ./dist/*.zip --notes "Release notes"

# List releases
gh release list

# View release
gh release view v1.0.0

# Download release assets
gh release download v1.0.0

# Delete release
gh release delete v1.0.0
```

**Use case:** Automate version releases in CI/CD, distribute binaries.

---

### gh alias - Create Shortcuts
```bash
# Create alias
gh alias set pv 'pr view'
gh alias set issues 'issue list --label bug'

# Now you can use
gh pv 42    # Instead of gh pr view 42
gh issues   # Lists all bugs

# List aliases
gh alias list

# Delete alias
gh alias delete pv
```

**Use case:** Speed up repeated commands, create team-specific shortcuts.

---

### gh search - Search GitHub
```bash
# Search repos
gh search repos "devops tools" --language=python

# Search with filters
gh search repos --stars=">1000" --language=go

# Search issues across GitHub
gh search issues "authentication bug" --state=open

# Search code
gh search code "import pandas" --language=python

# Search users
gh search users "location:India" --followers=">100"
```

**Use case:** Find libraries, research solutions, discover projects.

---

## Key Takeaways

1. **Context switching eliminated** - Everything from terminal, no browser needed
2. **Scriptable** - All commands support `--json` for automation
3. **Faster workflows** - Create PR, review, merge in seconds
4. **CI/CD integration** - Trigger workflows, check status, automate releases
5. **Team collaboration** - Review PRs, manage issues, assign work without leaving terminal

### Most useful commands for daily work:
```bash
gh pr create --fill           # Quick PR creation
gh pr list                    # Check team's PRs
gh pr checkout 42             # Test someone's PR
gh issue create               # Report bugs fast
gh repo view --web            # Jump to browser when needed
```

### Pro tips:

- Use `gh pr status` to see your PRs and assigned reviews
- `gh pr create --draft` for work-in-progress PRs
- `gh issue list --assignee @me` shows your tasks
- Combine with `jq` for powerful automation: `gh pr list --json number,title`

---
