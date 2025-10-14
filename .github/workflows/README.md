# GitHub Workflows

This directory contains automated workflows for the pipelines repository.

## Kustomize Diff Review Workflow

### Overview

The `kustomize-diff.yaml` workflow automatically reviews pull requests that modify Konflux configuration files. It compares the generated Kustomize manifests between the PR branch and target branch, helping reviewers understand exactly what changes will be applied to the cluster.

### Triggers

The workflow runs automatically when:
- A pull request is opened, synchronized, or reopened
- The PR includes changes to files under `konflux-configs/**`

### What It Does

1. **Installs dyff** - Semantic YAML diff tool for Kubernetes manifests
2. **Checks out both branches** - Gets both the PR branch (HEAD) and target branch (BASE)
3. **Builds manifests** - Uses Buildah with `konflux-configs/Dockerfile` (same as production)
4. **Extracts manifests** - Uses `podman save` + tar to extract `/manifests.yaml` from images
5. **Generates diff** - Uses `dyff --output github` for colorized, document-aware diffs
6. **Extracts summary** - Parses dyff output to count added/removed/modified documents
7. **Posts PR comment** - Shows summary and full diff with all details
8. **Uploads artifacts** - Saves manifests and diff output for 30 days

> **Important:** This workflow uses **dyff** for semantic YAML comparison, which understands Kubernetes resource structure and accurately identifies changes (unlike line-based diff that can be confused by resource modifications). It also uses **Buildah** (Red Hat's container building tool) and the same `Dockerfile` that Konflux uses in production to build the manifests. Files are extracted using `podman save` and tar commands, which works reliably in rootless CI environments with locally built images.

### Comment Features

The workflow posts a streamlined comment that includes:

- **Build Status** - Shows if Kustomize builds succeeded or failed
- **Compact Summary** - Colorized count of added/removed/modified documents
- **Detailed Diff** - Full dyff output showing exact changes
- **Artifacts Link** - Direct link to download full outputs

Key features:
- Colorized summary (green for added, red for removed, yellow for modified)
- Document-aware analysis from dyff
- Shows specific field changes with hierarchical paths
- Proper syntax highlighting and markdown formatting

### Example Output

When a PR is opened, you'll see a comment like this:

```markdown
## Configuration Diff

**4** document(s) impacted: ```diff
+ 1 added
- 2 removed
! 1 modified
```

<details>
<summary><strong>Diff</strong></summary>

```diff
     _        __  __
   _| |_   _ / _|/ _|  between base-output.yaml
 / _' | | | | |_| |_       and head-output.yaml
| (_| | |_| |  _|  _|
 \__,_|\__, |_| |_|   
       |___/

- one document removed:
  - kind: Service
    name: old-service

+ one document added:
  + kind: Service
    name: new-service

± value change
  + kind: Deployment
    name: api-server
  
  spec.template.spec.containers.0.image
  - quay.io/app:v1.0
  + quay.io/app:v2.0
```

</details>

---

📦 **Artifacts:** [base-output.yaml, head-output.yaml, dyff-output.txt](...)
```

### Understanding the Diff

The workflow uses **dyff with GitHub output format** for optimal display:

**Summary format:**
- Uses diff syntax for colorization: `+ added` (green), `- removed` (red), `! modified` (yellow)
- Compact single-line format without lists or emojis
- Shows total impacted documents at a glance

**Diff output:**
- Document-aware: Understands YAML documents (Kubernetes resources)
- Shows added documents: `+ one document added:`
- Shows removed documents: `- one document removed:`
- Shows modifications: `± value change` with hierarchical field paths
- Example path: `spec.template.spec.containers.0.image`
- Colorized output with syntax highlighting
- Much more accurate than line-based diff for YAML

The `--output github` flag generates markdown-formatted output that GitHub renders beautifully. dyff automatically detects added, removed, and modified YAML documents.

### Artifacts

The workflow uploads these artifacts for debugging:
- `base-output.yaml` - Kustomize output from target branch
- `head-output.yaml` - Kustomize output from PR branch
- `dyff-output.txt` - Complete dyff semantic diff output
- Error logs (if builds fail)

Access artifacts from the workflow run page in GitHub Actions or via the link in the PR comment.

### Permissions

The workflow requires:
- `contents: read` - To checkout repository code
- `pull-requests: write` - To post comments on PRs

### Troubleshooting

**Workflow doesn't run:**
- Ensure your PR modifies files under `konflux-configs/`
- Check that the workflow file is on the base branch

**Build fails:**
- Review the error log in the PR comment
- Test locally using the same commands as the workflow (see Local Testing section)
- Verify all referenced files exist in `base/` and `overlay/` directories
- Check that the Konflux test image is accessible: `quay.io/konflux-ci/konflux-test:latest`

**Extraction fails:**
- The workflow uses `podman save` + tar to extract files from images
- Verify the image was built successfully: `podman images | grep konflux-config`
- Test extraction manually: `podman save image-name -o test.tar && tar -xOf test.tar --wildcards '*/layer.tar' | tar -xOf - manifests.yaml`
- Check that manifests.yaml exists in the image by inspecting the Dockerfile
- The multi-tier fallback tries different tar formats for maximum compatibility

**Comment not posted:**
- Check workflow logs in GitHub Actions
- Ensure the GITHUB_TOKEN has sufficient permissions
- Verify the repository settings allow workflow comments

**Diff is truncated:**
- For very large diffs (>500 lines), only the first 500 lines are shown in the comment
- Download the full diff from workflow artifacts

**Changes not detected:**
- The workflow relies entirely on dyff's exit code
- If dyff exits with 0, no changes are reported
- If dyff exits with non-zero, changes are shown
- Download artifacts to manually inspect if needed

### Local Testing

To test changes locally before pushing (using Buildah + dyff, same as the workflow):

```bash
# Install dyff (semantic YAML diff tool)
# Linux:
curl -sL https://github.com/homeport/dyff/releases/download/v1.9.0/dyff_1.9.0_linux_amd64.tar.gz | tar xz
sudo mv dyff /usr/local/bin/

# macOS:
brew install dyff

# Save current branch
CURRENT_BRANCH=$(git branch --show-current)

# Build from base branch using Buildah
git checkout main
cd konflux-configs
buildah bud --build-arg ENVIRONMENT=prod -t konflux-config-base -f Dockerfile .
podman save konflux-config-base -o /tmp/base.tar
tar -xOf /tmp/base.tar --wildcards '*/layer.tar' | tar -xOf - manifests.yaml > /tmp/base.yaml
rm /tmp/base.tar

# Build from your branch using Buildah
git checkout $CURRENT_BRANCH
buildah bud --build-arg ENVIRONMENT=prod -t konflux-config-head -f Dockerfile .
podman save konflux-config-head -o /tmp/head.tar
tar -xOf /tmp/head.tar --wildcards '*/layer.tar' | tar -xOf - manifests.yaml > /tmp/head.yaml
rm /tmp/head.tar

# Compare using dyff (recommended - semantic diff)
dyff between /tmp/base.yaml /tmp/head.yaml

# Or use GitHub output format (same as CI)
dyff between --output github /tmp/base.yaml /tmp/head.yaml

# Or use traditional diff for quick check
diff -u /tmp/base.yaml /tmp/head.yaml | less

# Or use a visual diff tool
code --diff /tmp/base.yaml /tmp/head.yaml

# Clean up
buildah rmi konflux-config-base konflux-config-head
```

> **Note:** This approach uses `dyff` for semantic YAML comparison (same as CI) and `podman save` to export images. The workflow uses the same Dockerfile as production, ensuring the diff reflects the actual build output.
> 
> **Requirements:** You need `buildah`, `podman`, and optionally `dyff`:
> ```bash
> # macOS
> brew install buildah podman dyff
> 
> # Linux (Fedora/RHEL/CentOS)
> sudo dnf install buildah podman
> # Install dyff from GitHub releases
> 
> # Linux (Ubuntu/Debian)
> sudo apt install buildah podman
> # Install dyff from GitHub releases
> ```

### Integration with PR Workflow

This workflow complements the existing Konflux pipelines:

1. **PR opened** → This workflow generates diff for review
2. **Reviewer examines diff** → Understands cluster impact
3. **PR approved & merged** → Konflux build pipeline runs
4. **Build succeeds** → Konflux release pipeline deploys changes

### Best Practices

**For PR Authors:**
- Review the generated diff before requesting review
- Ensure changes match your expectations
- Add explanations for complex changes in PR description
- Test locally if the build fails

**For Reviewers:**
- Always check the diff comment before approving
- Verify namespace, RBAC, and resource names
- Look for unintended resource deletions
- Confirm changes align with PR description

### Customization

To modify the workflow:

1. **Change target overlay:**
   ```yaml
   # In kustomize-diff.yaml, modify:
   kustomize build overlay/dev  # Instead of overlay/prod
   ```

2. **Add additional checks:**
   ```yaml
   # Add new steps after "Generate Diff Report"
   - name: Custom Validation
     run: |
       # Your validation logic
   ```

3. **Change diff format:**
   ```yaml
   # Modify the diff command in "Generate Diff Report"
   diff -y /tmp/base-output.yaml /tmp/head-output.yaml  # Side-by-side
   ```

4. **Add more overlays:**
   - Duplicate the build and diff steps
   - Run for both `overlay/prod` and `overlay/dev`
   - Combine results in the PR comment

### Related Documentation

- [Konflux Configuration as Code](../konflux-configs/README.md)
- [Kustomize Documentation](https://kustomize.io/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Pull Request Workflow](../konflux-configs/README.md#pull-request-workflow)

