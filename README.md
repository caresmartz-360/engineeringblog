# CS360 Engineering Blog

Welcome to the CS360 Engineering Blog repository! This is where we share insights, updates, and technical deep dives from our engineering team.

## 🚀 Quick Start for Contributors

### Prerequisites

- Ruby 3.x or higher
- Bundler gem installed (`gem install bundler`)

### Local Development Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/caresmartz-360/engineeringblog.git
   cd engineeringblog
   ```

2. **Install dependencies**
   ```bash
   bundle install
   ```

3. **Run the local server**
   ```bash
   bundle exec jekyll serve
   ```

4. **View the blog**
   Open your browser to `http://127.0.0.1:4000/`

The site will automatically regenerate when you make changes to files (except `_config.yml`, which requires a server restart).

## ✍️ Contributing a New Blog Post

### Step 1: Create a New Post File

Create a new file in the `_posts/` directory with the following naming convention:

```
_posts/YYYY-MM-DD-title-of-your-post.markdown
```

**Example:**
```
_posts/2025-12-11-introducing-our-new-feature.markdown
```

### Step 2: Add Front Matter

Every post must start with YAML front matter between `---` markers:

```markdown
---
layout: default
title: "Your Post Title Here"
date: 2025-12-11 14:30:00 +0530
categories: engineering tutorial
---

Your post content starts here...
```

**Front Matter Fields:**
- `layout`: Always use `default`
- `title`: The title of your post (use quotes if it contains special characters)
- `date`: Publication date and time (format: `YYYY-MM-DD HH:MM:SS +TIMEZONE`)
- `categories`: Space-separated list of categories (optional)

### Step 3: Write Your Content

Write your post content using Markdown. Here are some common formatting examples:

```markdown
## Headings

Use `##` for main sections, `###` for subsections.

## Code Blocks

Use triple backticks with language specification:

```python
def hello_world():
    print("Hello, World!")
```

## Lists

- Bullet point 1
- Bullet point 2

1. Numbered item 1
2. Numbered item 2

## Links and Images

[Link text](https://example.com)

![Alt text](/path/to/image.png)

## Emphasis

*italic* or _italic_
**bold** or __bold__
```

### Step 4: Preview Your Post

1. Save your file
2. The local server will automatically regenerate
3. Refresh your browser to see the changes
4. Check that your post appears on the homepage

### Step 5: Submit Your Post

1. **Commit your changes**
   ```bash
   git add _posts/YYYY-MM-DD-your-post-title.markdown
   git commit -m "Add blog post: Your Post Title"
   ```

2. **Push to a new branch**
   ```bash
   git checkout -b post/your-post-title
   git push origin post/your-post-title
   ```

3. **Create a Pull Request**
   - Go to the repository on GitHub
   - Click "New Pull Request"
   - Select your branch
   - Add a description of your post
   - Request review from the team

## 📝 Writing Guidelines

### Best Practices

- **Keep it focused**: One main topic per post
- **Use clear headings**: Help readers scan your content
- **Add code examples**: Show, don't just tell
- **Include images**: Visual aids improve understanding
- **Proofread**: Check for typos and grammar
- **Test locally**: Always preview before submitting

### Post Categories

Use relevant categories to help organize content:
- `engineering` - General engineering topics
- `tutorial` - Step-by-step guides
- `architecture` - System design and architecture
- `devops` - Infrastructure and deployment
- `update` - Product updates and announcements
- `culture` - Team culture and processes

### Excerpt Control

By default, Jekyll uses the first paragraph as the excerpt shown on the homepage. To control this:

```markdown
Your intro paragraph here.

<!--more-->

The rest of your post continues here...
```

## 🎨 Theme and Styling

This blog uses the **Hacker** theme, which provides a clean, terminal-inspired dark design. The theme is managed through the `github-pages` gem for compatibility with GitHub Pages.

## 🔧 Configuration

Site configuration is in `_config.yml`. Most contributors won't need to modify this file, but if you do:

- Changes to `_config.yml` require a server restart
- Test thoroughly before committing
- Coordinate with the team for major changes

## 📚 Additional Resources

- [Jekyll Documentation](https://jekyllrb.com/docs/)
- [Markdown Guide](https://www.markdownguide.org/)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)

## 🤝 Getting Help

If you have questions or run into issues:

1. Check existing posts in `_posts/` for examples
2. Review the Jekyll documentation
3. Ask in the team chat
4. Open an issue in this repository

## 📄 License

[Add your license information here]

---

Happy blogging! 🎉
