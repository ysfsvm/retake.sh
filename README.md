# retake.sh

Source code for my personal blog. Built with [Hugo](https://gohugo.io/).

## Structure

- `hugo/`: The Hugo site (content, config, and theme submodule).
- `hosting/`: Docker Compose configuration for Umami analytics and an alternative self-hosted Hugo server.
- `.github/workflows/`: GitHub Actions workflow for automated deployment to GitHub Pages.

## Deployment

The site is hosted on GitHub Pages. Any push to the `main` branch automatically triggers a build and deploy.

### Self-Hosting (Analytics & Fallback)

To run the self-hosted services (like Umami analytics):

```bash
cd hosting
# Make sure to create your .env file here based on your credentials
docker compose up -d
```

## Local Development

Clone the repository with submodules to fetch the theme:

```bash
git clone --recurse-submodules https://github.com/ysfsvm/retake.sh.git
```

To preview the site locally:

```bash
cd hugo
hugo server -D
```
