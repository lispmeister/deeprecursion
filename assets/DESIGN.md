---
name: Serene Reader
colors:
  surface: '#fdf8f8'
  surface-dim: '#ddd9d8'
  surface-bright: '#fdf8f8'
  surface-container-lowest: '#ffffff'
  surface-container-low: '#f7f3f2'
  surface-container: '#f1edec'
  surface-container-high: '#ebe7e7'
  surface-container-highest: '#e5e2e1'
  on-surface: '#1c1b1b'
  on-surface-variant: '#444748'
  inverse-surface: '#313030'
  inverse-on-surface: '#f4f0ef'
  outline: '#747878'
  outline-variant: '#c4c7c7'
  surface-tint: '#5f5e5e'
  primary: '#181919'
  on-primary: '#ffffff'
  primary-container: '#2d2d2d'
  on-primary-container: '#959494'
  inverse-primary: '#c8c6c6'
  secondary: '#346666'
  on-secondary: '#ffffff'
  secondary-container: '#b5e9e9'
  on-secondary-container: '#386b6b'
  tertiary: '#191917'
  on-tertiary: '#ffffff'
  tertiary-container: '#2e2d2b'
  on-tertiary-container: '#979491'
  error: '#ba1a1a'
  on-error: '#ffffff'
  error-container: '#ffdad6'
  on-error-container: '#93000a'
  primary-fixed: '#e4e2e1'
  primary-fixed-dim: '#c8c6c6'
  on-primary-fixed: '#1b1c1c'
  on-primary-fixed-variant: '#474747'
  secondary-fixed: '#b8ecec'
  secondary-fixed-dim: '#9cd0d0'
  on-secondary-fixed: '#002020'
  on-secondary-fixed-variant: '#194e4e'
  tertiary-fixed: '#e6e2de'
  tertiary-fixed-dim: '#c9c6c3'
  on-tertiary-fixed: '#1c1b1a'
  on-tertiary-fixed-variant: '#484644'
  background: '#fdf8f8'
  on-background: '#1c1b1b'
  surface-variant: '#e5e2e1'
typography:
  h1:
    fontFamily: Newsreader
    fontSize: 48px
    fontWeight: '600'
    lineHeight: '1.2'
    letterSpacing: -0.02em
  h2:
    fontFamily: Newsreader
    fontSize: 32px
    fontWeight: '500'
    lineHeight: '1.3'
    letterSpacing: -0.01em
  h3:
    fontFamily: Newsreader
    fontSize: 24px
    fontWeight: '500'
    lineHeight: '1.4'
    letterSpacing: '0'
  body-lg:
    fontFamily: Newsreader
    fontSize: 21px
    fontWeight: '400'
    lineHeight: '1.8'
    letterSpacing: 0.01em
  body-md:
    fontFamily: Newsreader
    fontSize: 18px
    fontWeight: '400'
    lineHeight: '1.7'
    letterSpacing: 0.01em
  caption:
    fontFamily: Inter
    fontSize: 14px
    fontWeight: '400'
    lineHeight: '1.5'
    letterSpacing: 0.02em
  label:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: '600'
    lineHeight: '1'
    letterSpacing: 0.05em
rounded:
  sm: 0.125rem
  DEFAULT: 0.25rem
  md: 0.375rem
  lg: 0.5rem
  xl: 0.75rem
  full: 9999px
spacing:
  max-width-content: 720px
  section-gap: 5rem
  paragraph-gap: 2rem
  margin-wide: auto
  gutter: 2rem
---

## Brand & Style

This design system is built on the principles of **Minimalism** and **Intellectual Calm**. The primary goal is to disappear, leaving only the content and the reader in a quiet, focused environment. It is designed for long-form essays, technical explanations, and literary journals where the cognitive load should be reserved for the ideas, not the interface.

The aesthetic response is one of "breathability"—achieved through a reduction of borders, the elimination of heavy shadows, and an expansive use of negative space. It avoids the clinical coldness of pure white in favor of a softer, paper-like warmth to sustain reading sessions over long durations.

## Colors

The palette is intentionally restrained to prevent visual fatigue. 

- **Primary Text:** A deep charcoal (#2D2D2D) is used instead of pure black to soften the contrast against the background, mimicking high-quality ink on paper.
- **Background:** A soft off-white (#FAFAFA) provides a warm, non-reflective base that reduces glare.
- **Accent:** A muted teal (#4A7C7C) serves as the sole functional color, reserved for links, progress indicators, and subtle call-to-actions.
- **System:** Grays are used sparingly for secondary metadata and borders, ensuring they do not compete with the main narrative flow.

## Typography

The design system prioritizes a **transitional serif** (Newsreader) for all long-form text. The high x-height and classic proportions provide the "intellectual" feel of a physical book while maintaining digital clarity.

The body text uses a generous 1.8 line height to allow the reader’s eye to track easily from line to line without tension. Inter is used as a functional secondary typeface for UI labels, navigation, and metadata to provide a clear distinction between "content" and "interface."

## Layout & Spacing

This design system utilizes a **Fixed Content Grid** focused on a single-column reading experience. 

- **Reading Width:** The central column is strictly limited to 720px (approximately 75-85 characters per line), which is the optimal range for reading speed and comprehension.
- **Negative Space:** Margins are expansive, pushing secondary navigation or side-notes far into the periphery to avoid distracting the eye.
- **Vertical Rhythm:** Large vertical gaps (80px+) separate major sections, signaling a mental pause for the reader.

## Elevation & Depth

The system rejects traditional drop shadows in favor of **Tonal Layering** and **Low-Contrast Outlines**.

- **Surfaces:** Depth is indicated by shifting from the background (#FAFAFA) to a slightly darker surface color (#F2F2F2). 
- **Borders:** When structural separation is required (e.g., code blocks or cards), a subtle 1px border (#E5E5E5) is used.
- **Interactive States:** Lift is communicated via the accent color (teal) rather than physical shadows. For example, a hovered link might gain a thicker underline or a subtle color shift.

## Shapes

The shape language is **Soft and Disciplined**. A consistent 0.25rem (4px) radius is applied to code blocks, buttons, and input fields. This slight rounding removes the "sharpness" of a purely geometric grid without becoming overly playful or bubbly. Images and large containers should follow this subtle rounding to maintain a cohesive, gentle visual flow.

## Components

### Code Blocks
Code blocks should use a monospaced font at a slightly smaller scale (15px) with a background of `#F6F8FA`. Syntax highlighting must use a muted, earthy palette (soft oranges, blues, and greens) to ensure code does not appear visually louder than the surrounding prose.

### Diagrams & Illustrations
SVG diagrams must use light stroke weights (1px to 1.5px) using the primary text color. Fills should be semi-transparent versions of the accent color. Animations should be slow and ease-in-out to prevent jarring the reader's focus.

### Buttons
Primary buttons are outlined with the accent color or have a solid muted teal background with white text. They should feel understated. Secondary buttons use text-only styling with a subtle hover state.

### Lists & Blockquotes
- **Lists:** Use custom bullets (small teal circles or dashes) with significant indentation.
- **Blockquotes:** Indicated by a 2px teal vertical line on the left, set in italicized Newsreader with increased line height to distinguish "guest" voices from the main author.

### Navigation
The header should be transparent and disappear on scroll-down, reappearing only on scroll-up to maximize the reading viewport.