# CLAUDE.md — EDS/AEM Boilerplate

## Project Overview

AEM Edge Delivery Services (EDS) boilerplate. Content is authored in Google Drive (mounted at `/` via `fstab.yaml`) and served through Adobe's Helix CDN. The codebase is purely front-end: blocks, styles, and scripts. No build step — files are served directly.

## Directory Structure

```
blocks/       # One folder per block, each with {name}.js + {name}.css
scripts/
  aem.js      # Core AEM runtime & utility exports
  scripts.js  # Page decoration lifecycle (loadEager/loadLazy/loadDelayed)
  delayed.js  # Non-critical code, loaded 3s after page load
styles/
  styles.css       # Global styles & CSS custom properties
  fonts.css        # @font-face declarations (loaded lazily)
  lazy-styles.css  # Post-LCP global styles
fonts/        # .woff2 font files
icons/        # SVG icons referenced as <span class="icon icon-{name}">
head.html     # Custom <head> additions (viewport, script/style imports)
fstab.yaml    # Content mountpoint config
```

## Block Architecture

Each block lives in `blocks/{name}/` and consists of exactly two files:
- `{name}.js` — must export a default `decorate` function
- `{name}.css` — scoped to `.{name}` selector

### Block JS template

```javascript
// blocks/myblock/myblock.js
export default function decorate(block) {
  // block is the DOM element with class="myblock"
  // Restructure block.children, add classes, fetch data, etc.
}
```

Use `async` when the block needs to fetch content or load fragments.

### Import patterns

```javascript
// Core utilities
import { createOptimizedPicture, getMetadata, buildBlock } from '../../scripts/aem.js';

// Page-level helpers
import { decorateMain, loadFonts } from '../../scripts/scripts.js';

// Sibling block utilities
import { loadFragment } from '../fragment/fragment.js';
```

Always include the `.js` extension in imports (ESLint rule).

### Block CSS conventions

```css
/* 4-space indentation */
/* Scope all rules to the block */
.myblock { }
.myblock > ul { }
.myblock .myblock-item { }

/* Mobile-first, single breakpoint */
@media (width >= 900px) {
    .myblock { }
}
```

No IDs — classes only. Class names use kebab-case.

## Key Utilities (`scripts/aem.js`)

| Export | Purpose |
|--------|---------|
| `createOptimizedPicture(src, alt, eager, breakpoints)` | Responsive `<picture>` with WebP + fallback. Use for all block images. |
| `getMetadata(name)` | Read `<meta name="...">` or `<meta property="...">` from the page. |
| `loadCSS(href)` | Dynamically load a CSS file. |
| `loadScript(src, attrs)` | Dynamically load a non-module JS file. |
| `buildBlock(name, content)` | Create a block DOM element from a 2D array. |
| `decorateBlock(block)` | Add `data-block-name`, status attributes to a block element. |
| `decorateIcons(element)` | Replace `<span class="icon icon-X">` with `<img src="/icons/X.svg">`. |
| `toClassName(text)` | Convert string to kebab-case CSS class name. |
| `toCamelCase(text)` | Convert string to camelCase. |
| `readBlockConfig(block)` | Parse a metadata/config block into a plain object. |

## Page Lifecycle (`scripts/scripts.js`)

```
loadEager()   — decorate main, show body, load first section + LCP image
loadLazy()    — load header/footer, remaining sections, lazy-styles, fonts
loadDelayed() — import delayed.js after 3000ms (analytics, chat widgets, etc.)
```

`decorateMain()` runs the full decoration pipeline: icons → auto-blocks (hero, fragments) → sections → blocks → buttons.

## CSS Custom Properties

Defined in `styles/styles.css` `:root`. Key variables:

```css
--background-color, --light-color, --dark-color, --text-color
--link-color: #3b63fb, --link-hover-color: #1d3ecf
--body-font-family: roboto, roboto-fallback, sans-serif
--heading-font-family: roboto-condensed, roboto-condensed-fallback, sans-serif
--body-font-size-m/s/xs   (desktop: 18/16/14px, mobile: 22/19/17px)
--heading-font-size-xxl/xl/l/m/s/xs
--nav-height: 64px
```

## Section & Wrapper Classes

The framework automatically adds:
- `.{blockname}-container` on the section containing the block
- `.{blockname}-wrapper` on the immediate wrapper div around the block

Sections can have metadata classes: `.section.light`, `.section.highlight` → applies `--light-color` background.

## Fragments

Reusable content pieces fetched as `.plain.html`:

```javascript
import { loadFragment } from '../fragment/fragment.js';
const fragment = await loadFragment('/path/to/fragment');
block.append(fragment);
```

Used for header, footer, and shared components. Fragment links (`<a href="/fragments/...">`) are auto-converted to fragments by `buildAutoBlocks()`.

## Metadata System

```javascript
// Reads <meta name="nav" content="/nav">
const navPath = getMetadata('nav') || '/nav';

// Common metadata keys: nav, footer, template, theme
```

## Image Optimization

Always use `createOptimizedPicture()` for images inside blocks — never use `<img>` directly:

```javascript
const picture = createOptimizedPicture(img.src, img.alt, false, [{ width: '750' }]);
img.closest('picture').replaceWith(picture);
```

Set the third argument (`eager`) to `true` only for the LCP (above-the-fold) image.

## Linting & Dev Commands

```bash
npm run lint        # ESLint + Stylelint
npm run lint:fix    # Auto-fix linting errors
npm run lint:js     # JS only
npm run lint:css    # CSS only (blocks/**/*.css + styles/*.css)
```

**ESLint:** airbnb-base, `.js` extensions required in imports, unix line endings, `no-param-reassign` allows property mutation.

**Stylelint:** stylelint-config-standard.

**Indentation:** 2 spaces in JS, 4 spaces in CSS.

## Accessibility Patterns

- Use semantic HTML: `<nav>`, `<header>`, `<footer>`, `<main>`, `<ul>`/`<li>` for lists
- Add `aria-expanded`, `aria-controls`, `aria-label` to interactive elements
- Handle keyboard events: Escape to close menus, Enter/Space to activate
- Icon `<img>` elements need descriptive `alt` text
