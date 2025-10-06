# SonarCloud + GitHub Actions Submission Report


## Snyk container scan fix

The Snyk container scan previously failed with `SNYK-CLI-0000: Could not detect package manager for file: dockerfile`. I adjusted the container-scan steps in `.github/workflows/enhanced-security.yml` to:

- Print the repository `Dockerfile` (debug) so you can confirm the file name and content seen by the runner.
- Remove `--file` and the `--exclude-base-image-vulns` flag, allowing Snyk to scan the built image for base-image vulnerabilities and application deps when present.

This change prevents Snyk from failing when the Dockerfile contains no application package manager layers (common for JAR-based images). To re-run the container scan and validate:

1. Ensure the workflow builds the image `cicd-demo:latest` (it does via `docker build -t cicd-demo:${{ github.sha }}` then tags to `latest`).
2. Re-run the workflow; Snyk will scan the image and generate `snyk-container.sarif`.
3. If you prefer to scan application dependencies inside the image, change the `args` to include `--file=Dockerfile` only if the Dockerfile uses a supported package manager and leaves metadata in the image layers.

Example of robust args to scan the image:

```yaml
args: >
  --severity-threshold=high
  --sarif-file-output=snyk-container.sarif
```

If you want me to push these changes and re-run the workflow for you, say so and I'll push the commit to `main` (or create a PR).

## SARIF upload collision fix

During upload of SARIF results we encountered the error: "The CodeQL Action does not support uploading multiple SARIF runs with the same category." This happens when multiple scan jobs attempt to upload SARIF files with the identical `category` value (e.g., `snyk-container`) in the same repository run history.

Fix applied:

- Updated `.github/workflows/enhanced-security.yml` to set a unique category per run using `${{ github.run_id }}`. Example:

```yaml
with:
  sarif_file: snyk-container.sarif
  category: snyk-container-${{ github.run_id }}
```

This ensures each SARIF upload is treated as a distinct category by the CodeQL upload action and avoids the collision error.

Note: the workflow runs Snyk in a matrix (dependencies, code). I updated both matrix SARIF uploads to include `${{ github.run_id }}` and the `matrix.scan-type` in the category so each matrix axis produces a unique category (for example `snyk-dependencies-123456789` and `snyk-code-123456789`).

## Slack disabled

Slack notifications have been intentionally disabled for this repository's CI to avoid external dependencies. The workflow will still create a GitHub issue when critical vulnerabilities are detected. If you want Slack notifications later, add the secret `SLACK_WEBHOOK_URL` and I can re-enable the Slack step.
