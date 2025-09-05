---
layout: post
title: "Guide to Setting Up a Free Personal Website"
date: 2025-08-29
author: Network Engineer Grandfather
tags: [Cloudflare, GitHub Pages, Website Setup, Custom Domain, Jekyll]
---
##### Created using Perplixity AI

## Guide to Setting Up Custom Domain on Cloudflare Pages with GitHub Pages

### 1. Account Creation

- **Create a Cloudflare account:** Sign up for a free account on Cloudflare by providing your email and setting a password.
- **Create a GitHub account:** If you do not have one, create a GitHub account to host your website repository.

### 2. Prepare Your Website on GitHub Pages (Updated)

- **Use Your Free Personal Website:** Every GitHub user gets one free personal website at `username.github.io`, tied to a repository named `username.github.io`. Content pushed to this repository publishes directly to your personal GitHub Pages domain.
- **Create Your Repository:** Use your personal website repository or create a separate repository for project sites.
- **Use a Template Generator:** It is highly recommended to use static site generators or templates for easier site management:
  - **Jekyll:** Natively supported by GitHub Pages; allows blog and static site use with Markdown and Liquid templates.
  - Other popular generators: Hugo, Gatsby, or simple HTML/CSS templates.
- **Add Your Content:** Push website files or Jekyll source (including `_config.yml`) to the repository.
- **Enable GitHub Pages:** In repository settings, enable GitHub Pages to publish your site. Your site is then available at `https://username.github.io` or `https://username.github.io/repository-name`.

### 3. Connect Cloudflare Pages to GitHub

- Log in to Cloudflare dashboard.
- Navigate to **Workers & Pages > Pages**, select **Create a Project**.
- Connect to your GitHub repository with your website files.
- Set build commands if needed (e.g., `exit 0` for no build).
- Deploy to get your Cloudflare Pages subdomain (`<your-site>.pages.dev`).

### 4. Add a Custom Domain on Cloudflare Pages

- In your Cloudflare Pages project, go to **Custom domains > Setup a custom domain**.
- Enter your custom domain (e.g., `www.example.com`).
- For apex domains, add your domain to Cloudflare and update nameservers at your domain registrar as provided by Cloudflare.
- For subdomains, add CNAME record pointing to your Cloudflare Pages subdomain if you keep DNS with another provider.

### 5. DNS Setup If Domain Is Not Hosted on Cloudflare

- At your current domain registrar:
  - For full Cloudflare hosting (apex domains), update your nameservers to Cloudflare’s.
  - For subdomains, add CNAME to point subdomain (e.g., `www`) to your Cloudflare Pages domain (`<your-site>.pages.dev`).
- Allow time for DNS propagation (up to 24-48 hrs).

### 6. Verify and Activate

- Verify domain ownership in Cloudflare Pages.
- Activate the custom domain.
- Cloudflare will provision SSL for HTTPS.
- Your custom domain now serves your website hosted on Cloudflare Pages backed by your GitHub repository.

---

This setup ensures automated deployment from GitHub with Cloudflare delivering content securely and efficiently.

