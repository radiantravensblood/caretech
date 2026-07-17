# Deploy Care-Tech to GitHub Pages

This prototype is a plain static site. Publish the repository's `main` branch from `/(root)`.

## Required repository layout

Keep these files at the top level of the repository:

```text
index.html
dashboard.html
companion.html
ARCHITECTURE.md
README.md
DEPLOY_GITHUB_PAGES.md
.nojekyll
```

`index.html` must be in the repository root. Do not upload the containing
`care-tech-github-pages-ready/` folder as a nested directory unless you
intentionally want the site under that path.

## Command-line deployment

Create an empty GitHub repository first, then run from inside the extracted
prototype directory:

```bash
git init
git branch -M main
git add .
git commit -m "Publish Care-Tech consent-first prototype"
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPOSITORY.git
git push -u origin main
```

For later updates:

```bash
git add .
git commit -m "Refine Care-Tech prototype"
git push
```

Never commit API keys, patient data, credentials, environment files, or real
clinical records.

## Enable GitHub Pages

1. Open the repository on GitHub.
2. Select **Settings**.
3. In **Code and automation**, select **Pages**.
4. Under **Build and deployment**, set **Source** to **Deploy from a branch**.
5. Choose branch **main** and folder **/(root)**.
6. Select **Save**.

The project-site address will normally be:

```text
https://YOUR-USERNAME.github.io/YOUR-REPOSITORY/
```

A user or organization site instead uses a repository named exactly
`YOUR-USERNAME.github.io` and publishes at:

```text
https://YOUR-USERNAME.github.io/
```

## Verification pass

After deployment:

- Open the landing page.
- Open the dashboard and return to the landing page.
- Open the companion preview and return to the landing page.
- Test Day, Night, and Chocolate modes.
- Test the dashboard client selector, provenance disclosures, and safety-plan dialog.
- Test at a narrow mobile width.
- Confirm the browser console has no errors.
- Confirm every screen still says the data are synthetic and not for clinical use.

GitHub Pages is static hosting. The companion interactions in this build are
scripted demonstrations; no live model, database, notification service, or
consent backend is deployed.
