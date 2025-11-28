# rajnandan.com

Personal website of Raj Nandan Sharma, built with [Zola](https://www.getzola.org/) and the [Serene](https://github.com/isunjn/serene) theme.

🌐 **Live site:** [https://rajnandan.com](https://rajnandan.com)

## Features

-   ⚡ Fast static site generation with Zola
-   🎨 Clean, minimal design with Serene theme
-   🌙 Dark/Light mode support
-   📱 Mobile responsive
-   📰 RSS feed for blog posts
-   🔍 Syntax highlighting for code blocks
-   🚀 Automatic deployment via GitHub Actions

## Prerequisites

-   [Zola](https://www.getzola.org/documentation/getting-started/installation/) (v0.19.0+)

### Installation

**macOS (Homebrew):**

```bash
brew install zola
```

**Windows (Scoop):**

```bash
scoop install zola
```

**Linux:**

```bash
# Snap
snap install zola --edge

# Or download from releases
# https://github.com/getzola/zola/releases
```

## Local Development

1. Clone the repository with submodules:

    ```bash
    git clone --recurse-submodules https://github.com/rajnandan1/rajnandan.com.git
    cd rajnandan.com
    ```

2. If you already cloned without submodules:

    ```bash
    git submodule update --init --recursive
    ```

3. Start the development server:

    ```bash
    zola serve
    ```

4. Open [http://localhost:1111](http://localhost:1111) in your browser

The site will automatically reload when you make changes.

## Project Structure

```
rajnandan.com/
├── config.toml           # Zola configuration
├── content/              # All content files
│   ├── _index.md         # Home page
│   ├── posts/            # Blog posts
│   │   ├── _index.md     # Blog section config
│   │   └── *.md          # Individual posts
│   ├── projects/         # Projects page
│   ├── books/            # Books page
│   ├── games/            # Games page
│   └── contact/          # Contact page
├── static/               # Static assets
│   ├── img/              # Images (avatar, favicon)
│   ├── icon/             # Custom icons
│   └── CNAME             # Custom domain config
├── themes/
│   └── serene/           # Serene theme (git submodule)
└── .github/
    └── workflows/
        └── deploy.yml    # GitHub Actions deployment
```

## Adding New Content

### Adding a Blog Post

1. Create a new markdown file in `content/posts/`:

    ```bash
    touch content/posts/my-new-post.md
    ```

2. Add frontmatter at the top:

    ```markdown
    +++
    title = "My New Post"
    description = "A brief description of the post"
    date = 2024-01-15
    
    [taxonomies]
    tags = ["tag1", "tag2"]
    
    [extra]
    toc = true
    comment = false
    +++

    Your content here...
    ```

### Adding a New Page

1. Create a new directory in `content/`:

    ```bash
    mkdir content/newpage
    ```

2. Add an `_index.md` file:

    ```markdown
    +++
    title = "Page Title"
    description = "Page description"
    template = "prose.html"
    
    [extra]
    lang = "en"
    title = "Page Title"
    subtitle = "Optional subtitle"
    +++

    Page content...
    ```

3. Add the page to `config.toml` sections:
    ```toml
    sections = [
      # ... existing sections
      { name = "newpage", path = "/newpage", is_external = false },
    ]
    ```

## Building for Production

```bash
zola build
```

The built site will be in the `public/` directory.

## Deployment

The site automatically deploys to GitHub Pages when you push to the `main` branch.

### Manual Deployment

1. Build the site:

    ```bash
    zola build
    ```

2. The GitHub Actions workflow will:
    - Checkout the code with submodules
    - Install Zola
    - Build the site
    - Deploy to GitHub Pages

### Custom Domain Setup

1. The `CNAME` file in `static/` contains `rajnandan.com`
2. Configure DNS records for your domain to point to GitHub Pages
3. Enable "Enforce HTTPS" in repository settings

## Theme Customization

To customize the theme, you can:

1. Override templates by copying them from `themes/serene/templates/` to a local `templates/` directory
2. Add custom CSS by creating `templates/_custom_css.html`
3. Add custom fonts via `templates/_custom_font.html`

See the [Serene theme documentation](https://github.com/isunjn/serene/blob/latest/USAGE.md) for more details.

## Updating the Theme

```bash
git submodule update --remote themes/serene
```

Check the [theme changelog](https://github.com/isunjn/serene/blob/main/CHANGELOG.md) for breaking changes before updating.

## License

Content © Raj Nandan Sharma. Theme by [isunjn](https://github.com/isunjn/serene).
