# Responsive Content Guide

This guide explains how to make images and tables responsive in your blog posts.

## Responsive Images

### Method 1: Using HTML directly
```html
<div class="responsive-image-container">
    <img src="your-image.png" alt="Description" class="responsive-image"/>
</div>
```

### Method 2: Using Jekyll include (recommended)
```liquid
{% include responsive_image.html src="your-image.png" alt="Description" caption="Optional caption" %}
```

## Responsive Tables

### Method 1: HTML with responsive classes
```html
<div class="responsive-table-container">
<table class="responsive-table">
<thead>
<tr>
<th>Column 1</th>
<th>Column 2</th>
</tr>
</thead>
<tbody>
<tr>
<td data-label="Column 1">Data 1</td>
<td data-label="Column 2">Data 2</td>
</tr>
</tbody>
</table>
</div>
```

### Method 2: For very small screens (stacked layout)
Add the `stack-on-mobile` class to the table:
```html
<table class="responsive-table stack-on-mobile">
```

**Important:** When using the stacked layout, make sure each `<td>` has a `data-label` attribute that matches the column header.

## Features

### Images
- Automatically scale to fit container width
- Maintain aspect ratio
- Add subtle shadow and rounded corners
- Center alignment
- Support for captions

### Tables
- Horizontal scrolling on small screens
- Hover effects
- Sticky headers
- Optional stacked layout for very small screens
- Scroll indicator on mobile devices

## CSS Classes Reference

### Image Classes
- `.responsive-image`: Main responsive image class
- `.responsive-image-container`: Container with proper spacing
- `.image-caption`: Styling for image captions

### Table Classes
- `.responsive-table`: Main responsive table class
- `.responsive-table-container`: Container with horizontal scrolling
- `.stack-on-mobile`: Additional class for mobile stacking

### Content Classes
- `.responsive-content`: General responsive content wrapper

## Browser Support
- Works on all modern browsers
- Mobile-first responsive design
- Supports screen sizes from 320px to desktop
- Graceful degradation on older browsers