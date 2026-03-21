---
title: "How I Deployed My Blog on GitHub Pages Using Hugo"
date: 2026-03-21
draft: false
tags: ["hugo", "github", "devops", "tutorial"]
description: "A step by step guide on how I built and deployed this blog using Hugo and GitHub Pages — from zero to live in under an hour."
image: "https://wh1t35had0w.github.io/wh1t3-5hvd0w/images/hugo-blog.png"
---

So you want a blog. Not just any blog — a fast, free, fully custom one that you actually own.
That is exactly what I built, and in this post I will walk you through every step I took to
get this site live on GitHub Pages using Hugo.

## What is Hugo?

Hugo is a static site generator. Instead of a heavy CMS like WordPress, Hugo takes your
content written in simple Markdown files and converts them into a full website in milliseconds.
No database. No server. Just fast, clean HTML.

## What You Need Before Starting

- A Windows, Mac or Linux computer
- A GitHub account
- Git installed
- Hugo installed (extended version)
- About 30 to 60 minutes

## Step 1 — Install Git

Head over to [git-scm.com](https://git-scm.com/download/win) and download the installer.
Run it with all the default options. Once done, open your terminal and verify:
```bash
git --version
```

You should see something like `git version 2.53.0`.

## Step 2 — Install Hugo

Go to the [Hugo releases page](https://github.com/gohugoio/hugo/releases/latest) and download
the file ending in `windows-amd64.zip` — make sure it says **extended** in the name.

Extract it and place `hugo.exe` in `C:\Hugo\bin\`. Then add that folder to your system PATH.

Verify in your terminal:
```bash
hugo version
```

## Step 3 — Create Your Site

Navigate to where you want your blog to live and run:
```bash
hugo new site my-blog
cd my-blog
git init
```

## Step 4 — Add a Theme

I used the **Nomad Tech** theme. Add it as a Git submodule:
```bash
git submodule add https://github.com/m03315/nomad-tech themes/nomad-tech
echo 'theme = "nomad-tech"' >> hugo.toml
```

## Step 5 — Customize Your Config

Open `hugo.toml` and set your site title, author name, description and contact details.
This is also where you control your navigation menu and theme parameters.

## Step 6 — Create Your First Post
```bash
hugo new posts/hello-world.md
```

Open the file, set `draft: false` and write your content in Markdown. Add a cover image
by placing a `.jpg` or `.png` in `static/images/` and referencing it in your post header.

## Step 7 — Preview Locally
```bash
hugo server
```

Open your browser at `http://localhost:1313` and you will see your site live. Every change
you save will instantly reload in the browser.

## Step 8 — Push to GitHub

Create a new empty repository on GitHub. Then connect and push your local code:
```bash
git add .
git commit -m "Initial Hugo site"
git remote add origin https://github.com/yourusername/your-repo.git
git branch -M main
git push -u origin main
```

## Step 9 — Set Up GitHub Actions

Create the file `.github/workflows/hugo.yml` with the following content:
```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true
      - name: Build
        run: hugo --minify
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Step 10 — Enable GitHub Pages

Go to your repository on GitHub → **Settings** → **Pages** → set source to
**GitHub Actions** → Save.

## You Are Live

After the Action runs — usually under a minute — your blog will be live at:
```
https://yourusername.github.io/your-repo/
```

Every time you write a new post and push to `main`, GitHub Actions rebuilds and
redeploys everything automatically.

## Final Thoughts

Hugo and GitHub Pages is one of the cleanest setups for a personal blog. It is free,
fast, version controlled and fully yours. No subscription. No ads. No nonsense.

If you got stuck anywhere in this guide, feel free to reach out through the contact
links on this site. Happy hacking. 🖤

