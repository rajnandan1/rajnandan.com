+++
title = "How I built this personal website"
description = "A guide on how I set up my personal website using codedoc and GitHub Pages."
date = 2024-05-16
updated = 2024-05-16

[taxonomies]
tags = ["web", "tutorial", "personal"]

[extra]
toc = true
comment = false
+++

> **Note:** This post was originally written when my site was built with Codedoc. The site has since been migrated to [Zola](https://www.getzola.org/) with the [Serene](https://github.com/isunjn/serene) theme.

## Creating a GitHub repository

Sign up for a GitHub account and create a new repository. Keep the repository public. My username is rajnandan1 and I created a repository called rajnandan.

## Setting up the site

For this migration, I chose Zola - a fast static site generator written in Rust. Combined with the Serene theme, it provides:

-   Fast build times
-   Clean, minimal design
-   Dark/light mode support
-   RSS feeds
-   Syntax highlighting
-   Mobile responsive layout

## Running locally

```bash
# Install Zola (macOS)
brew install zola

# Serve the site locally
zola serve
```

The site should be running on [http://localhost:1111](http://localhost:1111)

## Adding content

### Home Page

The home page is located at `content/_index.md` and uses the Serene theme's home template.

### Adding a new page

To add a new page, create a new markdown file in the `content` folder with proper frontmatter.

### Adding blog posts

Blog posts go in `content/posts/`. Each post needs frontmatter with title, date, description, and optionally tags.

Example:

```markdown
+++
title = "My New Post"
description = "A brief description"
date = 2024-05-16

[taxonomies]
tags = ["tag1", "tag2"]

[extra]
toc = true
+++

Your content here...
```

## Building the site

```bash
zola build
```

## Deploying to GitHub Pages

The site uses GitHub Actions for automatic deployment. Every push to the main branch triggers a build and deploy to GitHub Pages.

## Custom Domain

1. Add a CNAME file in the `static` folder with your domain name
2. Configure DNS records to point to GitHub Pages
3. Enable custom domain in repository settings

That's it! The site is now live at your custom domain.
