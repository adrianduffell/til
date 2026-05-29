# Update node engine version with GitHub Actions

I pin the node engine version in `package.json` to an exact number to avoid differences between local development and CI. For example:

```js
"engines": {
	"node": "24.15.0"
}
```

The same version is controlled in GitHub Actions using `node-version-file: package.json` in `actions/setup-node`:

```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version-file: package.json
```

TIL to automate updating the node engine version using a scheduled cron to create the PR. This workflow runs weekly with the latest node version available in GitHub Actions. If this version is larger than the one pinned in `package.json`, it creates a PR to bump it to the new version. From there, it can be reviewed and tested like normal.

```yaml
name: Update Node Engine

on:
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 1'

permissions:
  contents: write
  pull-requests: write

jobs:
  update-node:
    runs-on: ubuntu-latest
    timeout-minutes: 1

    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          ref: main

      - name: Set up latest Node 24
        uses: actions/setup-node@v5
        with:
          node-version: 24
          cache: npm

      - name: Get current Node version
        id: latest
        run: |
          echo "version=$(node -p 'process.versions.node')" >> "$GITHUB_OUTPUT"

      - name: Read package engine version
        id: current
        run: |
          echo "version=$(node -p 'require("./package.json").engines.node')" >> "$GITHUB_OUTPUT"

      - name: Update package files
        if: steps.latest.outputs.version != steps.current.outputs.version
        run: |
          npm pkg set engines.node="${{ steps.latest.outputs.version }}"
          npm install --package-lock-only

      - name: Check for changes
        id: changes
        run: |
          if git diff --quiet package.json package-lock.json; then
            echo "changed=false" >> "$GITHUB_OUTPUT"
          else
            echo "changed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Output diff
        if: steps.changes.outputs.changed == 'true'
        run: |
          git diff -- package.json package-lock.json

      - name: Check for existing update branch
        if: steps.changes.outputs.changed == 'true'
        id: branch
        run: |
          BRANCH="update-node-${{ steps.latest.outputs.version }}"
          echo "branch=$BRANCH" >> "$GITHUB_OUTPUT"

          if git ls-remote --exit-code --heads origin "$BRANCH" > /dev/null; then
            echo "exists=true" >> "$GITHUB_OUTPUT"
            echo "Branch $BRANCH already exists; skipping."
          else
            echo "exists=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Commit and push
        if: steps.changes.outputs.changed == 'true' && steps.branch.outputs.exists == 'false'
        run: |
          BRANCH="${{ steps.branch.outputs.branch }}"

          git config user.name "github-actions"
          git config user.email "github-actions@github.com"

          git checkout -b "$BRANCH"

          git add package.json package-lock.json
          git commit -m "Update node engine to ${{ steps.latest.outputs.version }}"

          git push --set-upstream origin "$BRANCH"

      - name: Create pull request
        if: steps.changes.outputs.changed == 'true' && steps.branch.outputs.exists == 'false'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh pr create \
            --base main \
            --head "${{ steps.branch.outputs.branch }}" \
            --title "Update node engine to ${{ steps.latest.outputs.version }}" \
            --body 'Updates the node engine in `package.json` to ${{ steps.latest.outputs.version }} and regenerates `package-lock.json`.

          Release notes:
          https://nodejs.org/en/blog/release/v${{ steps.latest.outputs.version }}'
```
