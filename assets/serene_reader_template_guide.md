# Serene Reader: Blog Template & Design System

This document provides the structural and stylistic foundation for the "Serene Reader" blog design. It's designed to be used by an AI agent to generate consistent, highly-legible blog pages.

## 1. Design Tokens (CSS Variables)

Use these tokens to maintain visual consistency across the site. The design relies on **Tailwind CSS** classes, but these variables define the core theme.

```css
:root {
  /* Colors */
  --color-surface: #fdf8f8;
  --color-surface-dim: #ddd9d8;
  --color-on-surface: #1d1b1a;
  --color-primary: #006a6a; /* Teal accent */
  --color-primary-container: #6ff7f6;
  --color-on-primary-container: #002020;
  --color-secondary: #4a6363;
  --color-outline: #6f7979;
  
  /* Typography */
  --font-serif: 'Newsreader', serif;
  --font-sans: 'Inter', sans-serif;
  
  /* Spacing & Layout */
  --max-width-content: 720px;
  --line-height-reading: 1.75;
}
```

## 2. Core Layout Shell (Shared Components)

Every page follows this basic structural shell.

### Header (TopNavBar)
- **Logo**: Serif, italic, positioned on the left.
- **Navigation**: Centered links (Essays, Technical, Archive, About).
- **Interactions**: Subtle teal underlines for active states; backdrop-blur on scroll.

### Footer
- Simple text-based layout with copyright on the left and utility links on the right.

## 3. Page Templates

### A. Blog Index Template
The index page uses a hierarchy-focused layout:
1.  **Featured Essay**: A large, full-width image followed by a bold title and a concise summary.
2.  **Article Grid**: A clean, multi-column grid for other posts. Each entry includes:
    - Thumbnail image.
    - Category label (all caps, small).
    - Title (serif, readable size).
    - Short excerpt.
    - Reading time or date.

### B. Article View Template
Designed for deep focus and long-form reading.
- **Header**: Centrally aligned title, author metadata, and date.
- **Reading Corridor**: Constrained to 720px for optimal line length.
- **Typography**: Large serif body text with generous line-height (`leading-relaxed`).
- **Code Blocks**: Soft-toned background, monospace font, with syntax highlighting.
- **Blockquotes**: Elegant left-border styling with italicized text.

## 4. Implementation Guidance for the AI Agent

1.  **Prioritize Legibility**: Never let decorative elements interfere with the text. Maintain high contrast.
2.  **Generous Whitespace**: Use significant padding and margins between sections to create an "airy" feel.
3.  **Responsive Fluidity**: Ensure the content width scales down gracefully for mobile devices while maintaining the 720px max-width on desktop.
4.  **Serif-First**: Use the Serif font for all long-form content; Sans-serif can be used for small labels or UI elements.
