**Caution: All the paths/directories mentioned in this file is for reference only because paths may change depending on different versions of fumadocs.**

## Path Convention

Throughout this guide, `<dir>` stands for the top-level directory of the Fumadocs app you are working in — the folder containing `package.json`, `src/`, and `content/`.

- If the app is at the repository root, `<dir>` is `.` and `<dir>/src/app/global.css` simply means `src/app/global.css`.
- If the app lives in a subfolder (e.g. `template/`, `apps/docs/`), substitute that name: `<dir>/src/app/global.css` → `template/src/app/global.css`.

Resolve `<dir>` once at the start, then apply it consistently to every path below.

**Note:** `<dir>` applies to filesystem paths only. Import specifiers beginning with `@/` (such as `@/lib/navigation`) are TypeScript path aliases resolved relative to the app's own `src/` directory — leave them exactly as written, with no `<dir>` prefix.

# Fumadocs Initial Setup Skills

This guide provides step-by-step instructions for implementing sidebar tabs in Fumadocs with section-specific colors. Complete Task 1 to set up the sidebar structure, then Task 2 to wire up the dynamic coloring system.

## Task 1: Creating Sidebar Tabs

**Objective**: Set up root-level sidebar tabs using meta.json files to organize documentation into sections.

### Step 1: Create Section Folders

Create two section directories under `<dir>/content/docs/`:

```bash
mkdir -p "<dir>/content/docs/(sec-1)"
mkdir -p "<dir>/content/docs/sec-2"
```

**Note**: The parentheses in `(sec-1)` make it a Next.js route group, which means it becomes the default route (accessing `/docs` redirects to this section).

### Step 2: Organize Content Files

Move your content files into the respective sections:

```bash
mv "<dir>/content/docs/index.mdx" "<dir>/content/docs/(sec-1)/index.mdx"
mv "<dir>/content/docs/test.mdx" "<dir>/content/docs/sec-2/test.mdx"
```

### Step 3: Create Section Meta Files

Create `meta.json` in each section folder to define its properties.

**`<dir>/content/docs/(sec-1)/meta.json`:**
```json
{
  "title": "Section 1",
  "description": "Section 1 Description",
  "icon": "Building",
  "root": true,
  "pages": ["index"]
}
```

**`<dir>/content/docs/sec-2/meta.json`:**
```json
{
  "title": "Section 2",
  "description": "Section 2 Description",
  "icon": "PanelsTopLeft",
  "root": true,
  "pages": ["test"]
}
```

**Key properties:**
- `"root": true` - Marks this as a root-level sidebar tab
- `"icon"` - Lucide icon name (e.g., "Building", "PanelsTopLeft")
- `"pages"` - Array of page files in this section (without .mdx extension)

### Step 4: Define Section Order

Create `meta.json` at the docs root to define tab order:

**`<dir>/content/docs/meta.json`:**
```json
{
  "pages": ["(sec-1)", "sec-2"]
}
```

## Task 2: Wiring Section Colors

**Objective**: Set up dynamic color theming so each section displays with its own brand color throughout all UI elements (links, borders, ToC, etc.).

**Prerequisites**: Task 1 must be completed first.

### Step 1: Create Navigation Utility

Create a utility function to extract section names from paths.

**`<dir>/src/lib/navigation.ts`:**
```typescript
export function getSection(path: string | undefined) {
  if (!path) return 'sec-1';
  const [dir] = path.split('/', 1);
  if (!dir) return 'sec-1';
  return (
    {
      '(sec-1)': 'sec-1',
      'sec-2': 'sec-2',
    }[dir] ?? 'sec-1'
  );
}
```

**What it does:**
- Takes a file path like `'(sec-1)/index'` or `'sec-2/test'`
- Extracts the first directory segment
- Maps it to a normalized section name (`'sec-1'` or `'sec-2'`)
- Returns `'sec-1'` as default for undefined/empty paths

### Step 2: Define Section Colors in CSS

Add color variables and section-specific rules to your global CSS.

**`<dir>/src/app/global.css`:**
```css
@import 'tailwindcss';
@import 'fumadocs-ui/css/neutral.css';
@import 'fumadocs-ui/css/preset.css';

:root {
  --sec-1-color: hsl(26, 73%, 51%);
  --sec-2-color: hsl(217, 100%, 58%);
}

.dark {
  --sec-1-color: #fff383;
  --sec-2-color: #a9ceff;
}

.sec-1 {
  --color-fd-primary: var(--sec-1-color);
}

.sec-2 {
  --color-fd-primary: var(--sec-2-color);
}

html {
  scrollbar-gutter: stable;
}

html > body[data-scroll-locked] {
  margin-right: 0px !important;
  --removed-body-scroll-bar-size: 0px !important;
}
```

**How it works:**
- Define color variables for each section in light and dark modes
- CSS classes `.sec-1` and `.sec-2` map these to `--color-fd-primary`
- `--color-fd-primary` is Fumadocs' primary color variable used throughout all UI components

### Step 3: Create Dynamic Body Component

Create a client component that applies section classes dynamically.

**`<dir>/src/app/layout.client.tsx`:**
```typescript
'use client';

import { useParams } from 'next/navigation';
import { type ReactNode } from 'react';
import { getSection } from '@/lib/navigation';

export function Body({ children }: { children: ReactNode }) {
  const { slug = [] } = useParams();
  const section = Array.isArray(slug) ? getSection(slug[0]) : undefined;

  return <body className={`flex flex-col min-h-screen ${section || ''}`}>{children}</body>;
}
```

**Why a client component?**
- `useParams()` is a React hook that needs client-side rendering
- It reads the current URL to determine which section the user is viewing
- Applies the section name as a class to `<body>` (e.g., `<body class="sec-2">`)

### Step 4: Update Root Layout

Replace the static body tag with the dynamic Body component.

**`<dir>/src/app/layout.tsx`:**
```typescript
import { RootProvider } from 'fumadocs-ui/provider/next';
import './global.css';
import { Inter } from 'next/font/google';
import { Body } from './layout.client';

const inter = Inter({
  subsets: ['latin'],
});

export default function Layout({ children }: LayoutProps<'/'>) {
  return (
    <html lang="en" className={inter.className} suppressHydrationWarning>
      <Body>
        <RootProvider>{children}</RootProvider>
      </Body>
    </html>
  );
}
```

### Step 5: Add Tab Icon Color Transform

Update the docs layout to color tab icons based on their section.

**`<dir>/src/app/docs/layout.tsx`:**
```typescript
import { source } from '@/lib/source';
import { DocsLayout } from 'fumadocs-ui/layouts/docs';
import { baseOptions } from '@/lib/layout.shared';
import { getSection } from '@/lib/navigation';
import type { CSSProperties } from 'react';

export default function Layout({ children }: LayoutProps<'/docs'>) {
  return (
    <DocsLayout
      tree={source.getPageTree()}
      {...baseOptions()}
      tabs={{
        transform(option, node) {
          const meta = source.getNodeMeta(node);
          if (!meta || !node.icon) return option;
          const color = `var(--${getSection(meta.path)}-color, var(--color-fd-foreground))`;

          return {
            ...option,
            icon: (
              <div
                className="[&_svg]:size-full rounded-lg size-full text-(--tab-color) max-md:bg-(--tab-color)/10 max-md:border max-md:p-1.5"
                style={
                  {
                    '--tab-color': color,
                  } as CSSProperties
                }
              >
                {node.icon}
              </div>
            ),
          };
        },
      }}
    >
      {children}
    </DocsLayout>
  );
}
```

**What the transform does:**
- For each tab, retrieves its metadata
- Determines the section color using `getSection()`
- Wraps the icon in a div with a CSS custom property `--tab-color`
- Uses Tailwind's arbitrary value syntax `text-(--tab-color)` to apply the color
- Adds subtle background tint on mobile with `max-md:bg-(--tab-color)/10`

---

## How The System Works

Understanding the complete flow helps when implementing this pattern in new projects or troubleshooting issues.

### Complete Flow Explanation

1. **User navigates to a page**: `/docs/sec-2/test`

2. **Body component extracts section**:
   - `useParams()` gets `{ slug: ['sec-2', 'test'] }`
   - `getSection('sec-2')` returns `'sec-2'`
   - Applies class: `<body class="flex flex-col min-h-screen sec-2">`

3. **CSS rule activates**:
   ```css
   .sec-2 {
     --color-fd-primary: var(--sec-2-color);
   }
   ```

4. **Color propagates globally**:
   - All Fumadocs components reference `--color-fd-primary`
   - Links, borders, ToC highlights, buttons all pick up the section color

5. **Tab icons get colored independently**:
   - `tabs.transform` in docs layout colors each tab icon
   - Based on which section that tab belongs to
   - Not affected by the current page

## Expected Results

After completing both tasks, your Fumadocs site will have:

### Visual Result

- **Section 1 pages**: Orange/amber theme (light mode) or yellow (dark mode)
- **Section 2 pages**: Blue theme (light mode) or light blue (dark mode)
- **All UI elements**: Links, active ToC items, borders, hover states use the section color
- **Tab icons**: Always show their section's color regardless of active page

---

## Customization Guide

**Note**: This section is for human readers to understand how to customize the implementation. AI agents should NOT attempt to implement these customizations unless explicitly asked by the user.

### Adding More Sections (sec-3, sec-4, etc.)

If you want to add a third section or more, follow these steps:

1. **Create new section folder**: `<dir>/content/docs/sec-3/`
2. **Add meta.json** with `"root": true` and your desired title/icon
3. **Update `<dir>/content/docs/meta.json`** to include `"sec-3"` in the pages array
4. **Add color mapping** to `<dir>/src/lib/navigation.ts`:
   ```typescript
   return (
     {
       '(sec-1)': 'sec-1',
       'sec-2': 'sec-2',
       'sec-3': 'sec-3',  // Add this
     }[dir] ?? 'sec-1'
   );
   ```
5. **Define colors** in `<dir>/src/app/global.css`:
   ```css
   :root {
     --sec-3-color: hsl(120, 70%, 45%);
   }
   
   .dark {
     --sec-3-color: #90ee90;
   }
   
   .sec-3 {
     --color-fd-primary: var(--sec-3-color);
   }
   ```

### Changing Colors

To customize the colors for existing sections, modify the values in `<dir>/src/app/global.css`:

```css
:root {
  --sec-1-color: hsl(26, 73%, 51%);  /* Change hue, saturation, lightness */
  --sec-2-color: hsl(217, 100%, 58%);
}
```

**Tips:**
- Use HSL format for easier color adjustments
- Maintain adequate contrast for accessibility (test with WCAG tools)
- Always test colors in both light and dark modes

### Using Different Icons

Change the `"icon"` property in any section's meta.json to use a different [Lucide icon](https://lucide.dev/icons/):

```json
{
  "icon": "Rocket"
}
```

Popular choices: `"Book"`, `"Code"`, `"Settings"`, `"Database"`, `"Layers"`, `"Zap"`

---

## Pattern Reference

### Official Fumadocs Implementation

This implementation follows the same pattern used in the official Fumadocs documentation:
- They use `.ui`, `.headless`, `.framework` classes for their three sections
- Each class sets `--color-fd-primary` to a different color value
- The body element receives the appropriate class based on the URL path
- All UI components inherit the color through CSS custom properties

### Key Insight

Fumadocs is designed to be themeable through the `--color-fd-primary` CSS variable. Once you understand this, per-section theming becomes straightforward:

1. Define section-specific color variables
2. Create CSS classes that map those colors to `--color-fd-primary`
3. Dynamically apply the correct class to the body based on the current route
4. All Fumadocs UI components automatically pick up the color

This pattern ensures consistency across all UI elements without manually styling each component.

---

## Task 3: Adding Circular Gradient Navbar Icon

**Objective**: Add a circular gradient icon to the navbar that automatically adapts its colors based on the current section and light/dark mode.

**Prerequisites**: Task 2 must be completed first (color wiring must be in place).

### Step 1: Add FumadocsIcon Component

Add the icon component to your existing layout client file.

**`<dir>/src/app/layout.client.tsx`:**
```typescript
'use client';

import { useParams } from 'next/navigation';
import { type ReactNode, useId } from 'react';
import { getSection } from '@/lib/navigation';

export function Body({ children }: { children: ReactNode }) {
  const { slug = [] } = useParams();
  const section = Array.isArray(slug) ? getSection(slug[0]) : undefined;

  return <body className={`flex flex-col min-h-screen ${section || ''}`}>{children}</body>;
}

export function FumadocsIcon(props: React.SVGProps<SVGSVGElement>) {
  const id = useId();
  return (
    <svg width="80" height="80" viewBox="0 0 180 180" {...props}>
      <circle
        cx="90"
        cy="90"
        r="89"
        fill={`url(#${id}-iconGradient)`}
        stroke="var(--color-fd-primary)"
        strokeWidth="1"
      />
      <defs>
        <linearGradient id={`${id}-iconGradient`} gradientTransform="rotate(45)">
          <stop offset="45%" stopColor="var(--color-fd-background)" />
          <stop offset="100%" stopColor="var(--color-fd-primary)" />
        </linearGradient>
      </defs>
    </svg>
  );
}
```

**What it does:**
- Creates an SVG circle with a linear gradient fill
- Uses `useId()` to generate unique gradient IDs (prevents conflicts if multiple icons render)
- The gradient transitions from `--color-fd-background` (45%) to `--color-fd-primary` (100%)
- Stroke uses `--color-fd-primary` for a subtle outline
- Both CSS variables automatically update based on the current section

### Step 2: Add Icon to Navbar

Update the layout shared configuration to include the icon in the navbar title.

**`<dir>/src/lib/layout.shared.tsx`:**
```typescript
import type { BaseLayoutProps } from 'fumadocs-ui/layouts/shared';
import { appName, gitConfig } from './shared';
import { FumadocsIcon } from '@/app/layout.client';

export function baseOptions(): BaseLayoutProps {
  return {
    nav: {
      title: (
        <>
          <FumadocsIcon className="size-5" />
          {appName}
        </>
      ),
    },
    githubUrl: `https://github.com/${gitConfig.user}/${gitConfig.repo}`,
  };
}
```

**What it does:**
- Replaces the plain text title with a JSX fragment
- Renders the `FumadocsIcon` with `size-5` class (Tailwind for width/height)
- Displays the app name after the icon

### Step 3: Add Browser Tab Icon (Favicon)

Create a Next.js icon route to generate the favicon dynamically.

**`<dir>/src/app/icon.tsx`:**
```typescript
import { ImageResponse } from 'next/og';

export const size = { width: 32, height: 32 };
export const contentType = 'image/png';

export default function Icon() {
  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
        }}
      >
        <svg width="32" height="32" viewBox="0 0 180 180">
          <circle
            cx="90"
            cy="90"
            r="89"
            fill="url(#iconGradient)"
            stroke="#ffffff"
            strokeWidth="1"
          />
          <defs>
            <linearGradient id="iconGradient" gradientTransform="rotate(45)">
              <stop offset="45%" stopColor="#000000" />
              <stop offset="100%" stopColor="#ffffff" />
            </linearGradient>
          </defs>
        </svg>
      </div>
    ),
    {
      ...size,
    }
  );
}
```

**What it does:**
- Uses Next.js's `ImageResponse` API to generate a favicon dynamically
- Creates a 32×32 PNG image with the circular gradient icon
- Uses a static black-to-white gradient for consistency across all pages
- Next.js automatically serves this as the browser tab icon

**Do not add `export const runtime = 'edge'`.** Next.js 16 deprecated the Edge runtime and warns on build. `ImageResponse` works on the default Node.js runtime, so simply omit the declaration.

**Why static colors?**
- Browser tab icons (favicons) are static images cached by the browser
- They don't have access to CSS variables or dynamic theming
- A neutral black-to-white gradient provides consistent branding across all pages and themes

### Expected Results

After completing this task:

- **Navbar icon displays**: Circular gradient icon appears before the app name
- **Section color adaptation**: Icon gradient changes color when switching sections
  - Section 1: Orange/amber gradient
  - Section 2: Blue gradient
- **Theme adaptation**: Gradient automatically adjusts for light/dark mode
- **Unique IDs**: Multiple instances won't conflict due to `useId()` hook
- **Browser tab icon**: Circular black-to-white gradient icon appears in browser tabs

