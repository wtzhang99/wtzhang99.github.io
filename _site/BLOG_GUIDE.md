# Blog Management Guide

## Adding New Blog Posts

### 1. Create a new post file
Create a new markdown file in the `_posts` directory with the naming convention:
```
YYYY-MM-DD-title-with-hyphens.md
```

### 2. Post Front Matter
Each post should start with YAML front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD
author: Your Name
tags: [tag1, tag2, tag3]
excerpt: "Brief description of your post (optional)"
---
```

### 3. Writing Content
- Use standard Markdown syntax
- Add `<!--more-->` to create custom excerpts
- Include code blocks with syntax highlighting
- Use headings to structure your content

### 4. Tags and Categories
- Keep tags consistent across posts
- Use relevant tags for better organization
- Common tags for academic blogs: Research, Machine Learning, NLP, Publications, Conference

## Blog Features

### Pagination
- Posts are paginated (5 per page by default)
- Configure in `_config.yml` with `paginate` setting

### Navigation
- Previous/Next post navigation on individual post pages
- Blog index accessible via main navigation

### SEO
- Automatic sitemap generation
- SEO-friendly URLs
- Meta tags for social sharing

## Local Development

1. Install dependencies:
```bash
bundle install
```

2. Serve locally:
```bash
bundle exec jekyll serve
```

3. View at: `http://localhost:4000`

## Best Practices

### Writing Tips
- Keep posts focused on a single topic
- Use clear, descriptive titles
- Include relevant code examples
- Add personal insights and experiences
- Link to related papers or resources

### Technical Tips
- Optimize images before uploading
- Use relative links for internal content
- Test links before publishing
- Preview posts locally before committing

### Content Ideas for Academic Blogs
- Research insights and findings
- Conference experiences
- Tutorial posts on your area of expertise
- Industry vs. academia perspectives
- Collaboration stories
- Technical deep-dives
- Career advice and reflections

## File Structure
```
_posts/
├── 2025-09-06-journey-into-llms.md
├── 2025-08-15-text-rich-networks-industry.md
└── [your-new-posts].md

_layouts/
├── post.html          # Individual post layout
├── default.html       # Base layout
└── project.html       # Project layout

blog.html              # Blog index page
```

## Customization

### Styling
Blog styles are in `libs/custom/my_css.css` under the "BLOG STYLES" section.

### Layout
Modify `_layouts/post.html` to change the post layout structure.

### Pagination
Adjust pagination settings in `_config.yml`:
```yaml
paginate: 5                    # Posts per page
paginate_path: "/blog/page:num/"
```
