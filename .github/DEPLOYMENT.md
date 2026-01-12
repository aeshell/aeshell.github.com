# GitHub Pages Deployment Setup

This repository uses GitHub Actions to automatically build and deploy the Hugo site to GitHub Pages.

## Setup Instructions

To enable automatic deployment, you need to configure GitHub Pages in your repository settings:

### 1. Enable GitHub Pages

1. Go to your repository on GitHub: `https://github.com/aeshell/aeshell.github.com`
2. Click on **Settings** (top menu)
3. In the left sidebar, click on **Pages** (under "Code and automation")
4. Under **Build and deployment**:
   - **Source**: Select "GitHub Actions"
   - This allows the workflow to deploy to GitHub Pages

### 2. Trigger Deployment

The workflow will automatically run when:
- You push commits to the `main` or `web` branch
- You manually trigger it from the Actions tab

You can also manually trigger the workflow:
1. Go to the **Actions** tab in your repository
2. Click on "Deploy Hugo site to Pages" workflow
3. Click **Run workflow** button

### 3. View Your Site

After the workflow completes successfully:
- Your site will be available at: `https://aeshell.github.io/` (or your custom domain if configured)
- You can find the URL in the workflow run under the "deploy" job

## Workflow Details

The workflow (`.github/workflows/hugo.yml`):
- Uses Hugo v0.152.2 Extended
- Installs Go 1.25 for Hugo modules
- Builds the site with minification and garbage collection
- Deploys to GitHub Pages using the official GitHub Actions

## Custom Domain (Optional)

If you want to use a custom domain:
1. Add a `CNAME` file to the `static/` directory with your domain name
2. Configure your DNS provider to point to GitHub Pages
3. Enable HTTPS in repository settings > Pages

## Troubleshooting

If the deployment fails:
1. Check the Actions tab for error logs
2. Ensure GitHub Pages is enabled with "GitHub Actions" as the source
3. Verify the repository has Pages deployment permissions enabled
