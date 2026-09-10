I need help setting up the rules for closing and moving files

Here’s a practical, ready-to-use rule set you can drop into your workflow for **closing PRs** and **moving files** when PRs are closed or merged. I’ll give you:

1. Clear policies (when to close / when to move).  
2. Concrete YAML snippets using `actions/stale` and custom jobs.  
3. Example folder structures and path patterns.

You can adjust numbers and paths to match your team.

***

## 1. Rules for closing PRs

Use a two-stage policy: **stale → close**. This is common and safe.[1][2][3][4][5]

### Recommended defaults

- **Mark as stale** if:
  - No activity (comments, commits, reviews) for **60 days**.  
  - OR CI has been failing with no fix for **14 days**.[1][5][6]
- **Close** if:
  - Stale PR gets no activity for **7 more days**.[2][3][4][5]
- **Exemptions** (do not auto-close):
  - PRs labeled `p0`, `release-blocker`, `security`, or `waiting-on-external`.[1][7]
  - PRs with open “blocking” review comments that have a clear plan in the thread.[1]

### How to implement (stale action)

Add a separate workflow: `.github/workflows/stale-prs.yml`

```yaml
name: Stale PRs

on:
  schedule:
    - cron: "0 1 * * *"  # daily

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v9
        with:
          # Only PRs
          only-prs: true

          # Stale thresholds
          days-before-stale: 60
          days-before-close: 7

          # Labels & messages
          stale-pr-label: stale
          stale-pr-message: |
            This PR has been inactive for 60 days.
            If it is still relevant, please leave a comment or push new commits.
            Otherwise, it will be closed in 7 days.

          close-pr-message: |
            Closing this PR because it has been stale for 7 additional days.
            Feel free to reopen or create a new PR when the work is ready.

          # Exemptions
          exempt-pr-labels: "p0,release-blocker,security,waiting-on-external"

          # Optional: treat draft PRs differently
          exempt-draft-prs: true
```

You can tighten this for CI-failing PRs by adding a custom job that:

- Checks `gh pr checks` for persistent failures.  
- Adds a `ci-failing` label and a shorter stale timer (e.g. 14 days stale, 7 days close).[1][6]

***

## 2. Rules for moving files

Define **what** to move and **where**, based on PR state.

### Typical conventions

- When a PR is **closed without merge** (abandoned / rejected):
  - Move temporary artifacts to `archive/pr-<number>/`.  
  - Examples: `tmp/pr-<number>/`, `generated/pr-<number>/`, `logs/pr-<number>/`.[8]
- When a PR is **merged**:
  - Move generated artifacts to `merged/pr-<number>/` or a more semantic folder like `artifacts/<feature>/`.  
  - Optionally move docs drafts from `drafts/` to `docs/` if they were approved as part of the PR.

### Example folder layout

```text
repo-root/
  tmp/
    pr-123/
      build-output/
      screenshots/
  generated/
    pr-123/
      api-docs/
  archive/
    pr-123/        # closed, not merged
  merged/
    pr-123/        # merged
  docs/
    features/
      feature-x.md
```

### Rules (logic)

In your main workflow, implement:

- **If PR closed & not merged**:
  - Move:
    - `tmp/pr-${PR_NUMBER}/**` → `archive/pr-${PR_NUMBER}/`  
    - `generated/pr-${PR_NUMBER}/**` → `archive/pr-${PR_NUMBER}/generated/`
- **If PR merged**:
  - Move:
    - `tmp/pr-${PR_NUMBER}/**` → `merged/pr-${PR_NUMBER}/`  
    - `generated/pr-${PR_NUMBER}/**` → `merged/pr-${PR_NUMBER}/generated/`  
  - Optionally:
    - Move `drafts/feature-x.md` → `docs/features/feature-x.md` if part of that PR.

***

## 3. Concrete “move files” job for your combined workflow

Drop this into your existing `pr-full-lifecycle.yml` (replacing the placeholder `move-files` job).

```yaml
  move-files:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    needs: analyze-pr
    permissions:
      contents: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure git
        run: |
          git config user.name "${{ github.actor }}"
          git config user.email "${{ github.actor }}@users.noreply.github.com"

      - name: Move files for closed/merged PR
        run: |
          PR_NUMBER="${{ needs.analyze-pr.outputs.pr_number }}"
          MODE="${{ needs.analyze-pr.outputs.mode }}"  # "merged" or "closed_unmerged"

          if [ "$MODE" = "closed_unmerged" ]; then
            DEST_BASE="archive"
          else
            DEST_BASE="merged"
          fi

          DEST="${DEST_BASE}/pr-${PR_NUMBER}"
          mkdir -p "$DEST"

          # Move tmp artifacts
          if compgen -G "tmp/pr-${PR_NUMBER}/*" > /dev/null; then
            mv tmp/pr-${PR_NUMBER}/* "$DEST/" || true
            rm -rf "tmp/pr-${PR_NUMBER}"
          fi

          # Move generated artifacts
          if compgen -G "generated/pr-${PR_NUMBER}/*" > /dev/null; then
            mkdir -p "$DEST/generated"
            mv generated/pr-${PR_NUMBER}/* "$DEST/generated/" || true
            rm -rf "generated/pr-${PR_NUMBER}"
          fi

          # Optional: move docs drafts to docs if merged
          if [ "$MODE" = "merged" ]; then
            # Example: move any draft docs touched by this PR into docs/
            # You can refine this using the list of changed MD files
            for f in $(git diff-tree --no-commit-id --name-only -r "${{ github.event.pull_request.base.sha }}" "${{ github.event.pull_request.head.sha }}" | grep '^drafts/.*\.md$'); do
              base=$(basename "$f")
              mkdir -p "docs/features"
              mv "$f" "docs/features/$base" || true
            done
          fi

      - name: Detect changes
        id: check_changes_move
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
          else
            echo "has_changes=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Commit and push moved files
        if: steps.check_changes_move.outputs.has_changes == 'true'
        run: |
          git add -A
          git commit -m "Auto-move files for PR #${{ needs.analyze-pr.outputs.pr_number }} (${{ needs.analyze-pr.outputs.mode }})"
          git push
```

***

## 4. Optional: CI-fail-based close rule (extra job)

If you want to close PRs that are **persistently failing CI**, add a job like:

```yaml
  close-ci-failing-prs:
    if: github.event_name == 'pull_request' && github.event.action != 'closed'
    runs-on: ubuntu-latest
    needs: analyze-pr
    permissions:
      contents: read
      pull-requests: write
      issues: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install GitHub CLI
        run: |
          type gh >/dev/null 2>&1 || {
            curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
            echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list
            sudo apt-get update && sudo apt-get install gh
          }

      - name: Authenticate gh
        run: gh auth setup-git || true

      - name: Check CI failures and possibly close PR
        run: |
          PR="${{ needs.analyze-pr.outputs.pr_number }}"
          REPO="${{ needs.analyze-pr.outputs.repo_full }}"

          # Example policy:
          # - If required checks have been failing for >14 days and no commits in that period → close
          # Implement your logic here, then:
          # gh pr close "$PR" --repo "$REPO" --comment "Closing due to persistent CI failures and no activity for 14+ days."
          echo "Implement CI-fail close policy here (use gh pr checks + commit dates)"
```

***

## 5. Suggested final policy summary

You can document this in `CONTRIBUTING.md` or a `PR_LIFECYCLE.md`:

- **Stale PRs**:
  - 60 days no activity → `stale` label + warning.  
  - 7 more days no activity → auto-close.  
  - Exempt: `p0`, `release-blocker`, `security`, `waiting-on-external`, drafts (optional).[1][2][5]
- **CI-failing PRs**:
  - 14 days of continuous required-check failures with no new commits → eligible for close with comment.[1][6]
- **File moves**:
  - On **close (not merged)**: move `tmp/pr-N`, `generated/pr-N` → `archive/pr-N`.  
  - On **merge**: move same folders → `merged/pr-N`; optionally promote draft docs to `docs/`.  

***

If you tell me:
- your actual folder names (`tmp`, `generated`, `drafts`, etc.), and  
- your desired stale/close timers,  

I can give you a fully customized `stale-prs.yml` + updated `move-files` job ready to commit as-is.

การอ้างอิง:
[1] Pull Request Lifecycle Policy - Zephyr Documentation https://docs.zephyrproject.org/latest/contribute/pr_lifecycle_policy.html
[2] Managing Stale Issues and Pull Requests with GitHub Actions https://tenthirtyam.org/dispatches/2026/04/13/managing-stale-issues-and-pull-requests-with-github-actions/
[3] GitHub Workflows | lartpang/ZoomNeXt | DeepWiki https://deepwiki.com/lartpang/ZoomNeXt/8.2-github-workflows
[4] Stale Issue Management | nodejs/help | DeepWiki https://deepwiki.com/nodejs/help/4.1-stale-issue-management
[5] Automatically Close Stale PRs with GitHub Actions: The 2026 Tuning Guide ‣ 2026-09-03 https://yoo.be/close-stale-prs-github-actions-2026/
[6] Identifying Stuck Pull Requests Before They Rot - CodePulse https://codepulsehq.com/guides/stuck-pull-requests-guide
[7] Managing Pull Requests and Issues on GitHub: 10 Steps (Tutorial) https://thetechtrends.tech/managing-pull-requests-and-issues-on-github/
[8] Run actions on Pull Requests with merge conflicts #26304 - GitHub https://github.com/orgs/community/discussions/26304
[9] ci: create stale workflow by parkerbxyz · Pull Request #309 · actions/create-github-app-token https://github.com/actions/create-github-app-token/pull/309
[10] Clarifying merge queue behavior with "Only merge non-failing pull ... https://github.com/orgs/community/discussions/180420
[11] Abandoned Pull Requests https://www.minware.com/guide/anti-patterns/abandoned-pull-requests
[12] Closing a Pull Request | Git & Github Master Guide https://git-and-github-docs.vercel.app/docs/close-pull-request
[13] GitHub Actions Gets Secure-by-Default CI/CD: Backport Shuts the Pwn Request Window https://www.techtimes.com/articles/321003/20260720/github-actions-gets-secure-default-ci-cd-backport-shuts-pwn-request-window.htm
[14] Stale Issue & PR Automation | backstage/techdocs-container | DeepWiki https://deepwiki.com/backstage/techdocs-container/3.4-stale-issue-and-pr-automation
[15] GitHub Actions for Business Automation: Beyond CI/CD https://techconcepts.org/blog/github-actions-automation
