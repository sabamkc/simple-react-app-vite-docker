# FAQs — GitHub Pages deployment for this repo

Summary
- This document records the full debugging timeline for deploying the Vite + React app to GitHub Pages, the problems encountered, and the fixes that resolved them.

Timeline & Problems

1) Symptom: header rendered but body content not loaded in Pages; browser console showed 404s for JS assets (example: `/assets/index-*.js` or `/simple-react-app-vite-docker/assets/index-*.js`).
   - Root cause: the Pages workflow uploaded the repository root (source files) instead of the built production output. `index.html` in the repo referenced the source-built paths (via Vite dev references) that are not present unless the project is built.
   - Fix: Update the Actions workflow to run install + build and upload the generated `dist` directory. See `.github/workflows/static.yml` changes: install Node, run `npm install`, `npm run build`, then upload `dist` with `actions/upload-pages-artifact@v3`.

2) Symptom: GitHub Actions failed at the install step with `npm ci` error: "The `npm ci` command can only install with an existing package-lock.json".
   - Root cause: `npm ci` requires a lockfile. This repository had no `package-lock.json` (or shrinkwrap), so `npm ci` fails.
   - Fix: Use `npm install` in the workflow (or alternatively add a `package-lock.json` to the repo). The workflow was updated to `npm install` so the build can proceed when there is no lockfile. Optionally, use a conditional in the workflow to run `npm ci` when `package-lock.json` exists.

3) Symptom: After building and uploading `dist`, Pages still returned 404 for the JS asset at `/simple-react-app-vite-docker/assets/index-*.js` (or for other repo-hosted path variants).
   - Root cause: Vite's `base` config defaults to `'/'`. When GitHub Pages serves the site at `https://<user>.github.io/<repo>/`, the built asset URLs must include the repo base path. Without `base` configured, built HTML points to `/assets/...` (site root) while Pages serves under `/repo/` (404).
   - Fix: Set `base` in `vite.config.js` to the repository path: `base: '/simple-react-app-vite-docker/'`. Rebuild and redeploy so assets are referenced under `/simple-react-app-vite-docker/assets/...`.

Interim issues while verifying locally

- Local environment lacked `npm`: an attempted local `npm ci && npm run build` failed because the environment used by this automation/testing agent did not have Node/npm installed.
  - Resolution: Use Docker (multi-stage Dockerfile present) to run the build stage and extract `dist`, or run the build locally on a machine with Node installed.

- Docker attempt in the automated environment failed because the Docker daemon was not available in that environment.
  - Resolution: Run the Docker commands on your machine where Docker Desktop/daemon is available. Example commands are included below.

Key fixes applied in repo

- `.github/workflows/static.yml` — changed to:
  - Checkout repo
  - Setup Node (`actions/setup-node@v4`)
  - `npm install` (or conditional `npm ci` when lockfile exists)
  - `npm run build`
  - `actions/configure-pages@v5`
  - `actions/upload-pages-artifact@v3` with `path: dist`
  - `actions/deploy-pages@v4`

- `vite.config.js` — added `base: '/simple-react-app-vite-docker/'` so built assets use repo path.

Commands to reproduce locally (recommended)

1) Build locally (if Node is installed):
```bash
cd /path/to/simple-react-app-vite-docker
npm install
npm run build
ls -la dist
```

2) Build using Docker (works without Node locally):
```bash
cd /path/to/simple-react-app-vite-docker
# Build the build stage
docker build --target build -t simple-react-build:latest .
docker create --name tmp_build simple-react-build:latest
docker cp tmp_build:/app/dist ./dist
docker rm tmp_build
ls -la dist
```

3) Serve `dist` locally to inspect the production build:
```bash
# Python simple server
python3 -m http.server 8000 --directory dist
# Open http://localhost:8000 in your browser
```

Deployment steps (already applied)

- Commit & push the workflow and `vite.config.js` changes to `main` and open the Actions tab. The updated workflow will:
  - Install dependencies
  - Run the Vite build
  - Upload `dist` as the artifact for Pages
  - Deploy the built static site

Optional improvements

- Add `actions/cache` for Node modules in the workflow to speed up installs.
- Add a workflow conditional to run `npm ci` when `package-lock.json` exists and `npm install` otherwise.
- Add a `homepage` or environment variable for the base so you can reuse the repo in different hosts (optional pattern).

FAQs

Q: Why did I see a 404 for `/assets/index-*.js` on GitHub Pages?
A: The build produced asset paths relative to site root (`/assets/...`) but Pages hosts the site at `/your-repo/`. Configure Vite `base` to include the repo path or build and deploy to the site root.

Q: Should I use `npm ci` or `npm install` in CI?
A: `npm ci` is preferred for reproducible installs in CI, but it requires a lockfile (`package-lock.json` or `npm-shrinkwrap.json`). If you don't have a lockfile, use `npm install` or add a lockfile to the repo and use `npm ci` for faster, deterministic installs.

Q: How can I test the production build without installing Node locally?
A: Use Docker to run the `build` stage (this repo's Dockerfile has a `build` stage) and copy the `dist` output from the container, or run the build on any machine with Node installed.

Q: After changing `vite.config.js` base, do I need to rebuild?
A: Yes — changing `base` affects the generated asset URLs in `dist`. Re-run `npm run build` and redeploy.

If anything here is unclear or you want me to add the optional improvements (cache, conditional `npm ci`), say which one and I'll patch the workflow.

-- End of FAQs
