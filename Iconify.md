# Adding Iconify Icon Support to Fumadocs (Offline)

This guide explains how to add Iconify icon support to a Fumadocs project with offline capability, ensuring icons work without internet connection. The implementation maintains compatibility with the built-in Lucide icons plugin.

## Path Convention

Throughout this guide, `<dir>` stands for the top-level directory of the Fumadocs app you are working in — the folder containing `package.json`, `src/`, and `content/`.

- If the app is at the repository root, `<dir>` is `.` and `<dir>/src/lib/source.ts` simply means `src/lib/source.ts`.
- If the app lives in a subfolder (e.g. `template/`, `apps/docs/`), substitute that name: `<dir>/src/lib/source.ts` → `template/src/lib/source.ts`.

Resolve `<dir>` once at the start, then apply it consistently to every path below.

**Note:** `<dir>` applies to filesystem paths only. Import specifiers beginning with `@/` (such as `@/lib/icon-registry`) are TypeScript path aliases resolved relative to the app's own `src/` directory — leave them exactly as written, with no `<dir>` prefix.

Separately, `<collection-name>` is not a path. It is a placeholder for an Iconify icon set name (e.g. `mdi`, `logos`, `fa7-brands`) as published under the `@iconify-json/` npm scope — substitute the set you actually want.

## Overview

Fumadocs comes with built-in support for Lucide icons using the `lucideIconsPlugin()`. This implementation adds Iconify support that co-exists with Lucide without conflicts by:

- Using different naming conventions (Lucide: `Building`, Iconify: `source:name`)
- Creating a wrapper for Lucide that skips Iconify-formatted icons
- Adding a separate Iconify plugin that only processes icons with colons
- Preloading icon data locally for offline use (no CDN dependency)

## Prerequisites

- A working Fumadocs project with `fumadocs-core` and `lucide-react` installed
- Package manager (npm, pnpm, or yarn)

## Implementation Steps

### Step 1: Install Required Packages

Install the `@iconify/react` package and icon set packages you need, from inside `<dir>`:

```bash
cd "<dir>"
pnpm install @iconify/react
pnpm install -D @iconify-json/fa7-brands  # Font Awesome Brands example
pnpm install -D @iconify-json/<collection-name>
```

### Step 2: Create the Icon Registry

Create a new file at `<dir>/src/lib/icon-registry.ts`:

This component preloads icon collections for offline use, ensuring icons work without CDN dependency.

```tsx
'use client';

import { addCollection } from '@iconify/react';
import { useEffect } from 'react';

/**
 * Preload icon collections for offline use.
 * This ensures icons work without CDN dependency.
 *
 * To add more icon sets:
 * 1. Install: pnpm install -D @iconify-json/<collection-name>
 * 2. Import the JSON data in this file
 * 3. Add to the collections array below
 */

// Import your icon collections here
import fa7BrandsIcons from '@iconify-json/fa7-brands/icons.json';

// Add all collections to this array
const collections = [
  fa7BrandsIcons,
  // Add more icon sets here
];

// Register all collections
collections.forEach((collection) => {
  addCollection(collection as any);
});

/**
 * IconRegistry component - must be rendered in your root layout
 * to ensure icons are registered on the client
 */
export function IconRegistry() {
  useEffect(() => {
    // Re-register on client mount to ensure they're available
    collections.forEach((collection) => {
      addCollection(collection as any);
    });
  }, []);

  return null;
}
```

### Step 3: Create the Iconify Plugin

Create a new file at `<dir>/src/lib/iconify-plugin.tsx`:

```tsx
import { Icon } from '@iconify/react';
import { createElement } from 'react';

interface IconifyPluginOptions {
  defaultIcon?: string;
}

/**
 * Convert icon names into Iconify Icons, requires `@iconify/react` to be installed.
 *
 * Icon names should follow the format: "source:name" (e.g., "cbi:claude-clawd", "lucide:home")
 */
export function iconifyPlugin(options: IconifyPluginOptions = {}) {
  const { defaultIcon } = options;

  return {
    name: 'fumadocs:iconify-icons',
    transformPageTree: {
      file: (node: any) => {
        if (node.icon !== undefined && typeof node.icon === 'string' && node.icon.includes(':')) {
          node.icon = createElement(Icon, { icon: node.icon });
        } else if (node.icon === undefined && defaultIcon) {
          node.icon = createElement(Icon, { icon: defaultIcon });
        }
        return node;
      },
      folder: (node: any) => {
        if (node.icon !== undefined && typeof node.icon === 'string' && node.icon.includes(':')) {
          node.icon = createElement(Icon, { icon: node.icon });
        } else if (node.icon === undefined && defaultIcon) {
          node.icon = createElement(Icon, { icon: defaultIcon });
        }
        return node;
      },
      separator: (node: any) => {
        if (node.icon !== undefined && typeof node.icon === 'string' && node.icon.includes(':')) {
          node.icon = createElement(Icon, { icon: node.icon });
        } else if (node.icon === undefined && defaultIcon) {
          node.icon = createElement(Icon, { icon: defaultIcon });
        }
        return node;
      },
    },
  };
}
```

### Step 4: Create the Lucide Plugin Wrapper

Create a new file at `<dir>/src/lib/lucide-plugin-wrapper.tsx`:

This wrapper prevents the Lucide plugin from processing Iconify-formatted icons (those containing `:`).

```tsx
import { createElement } from 'react';
import { icons } from 'lucide-react';

/**
 * Custom Lucide Icons plugin that skips Iconify-formatted icons (those containing ':')
 * This allows Lucide and Iconify to co-exist without conflicts.
 */
export function lucideIconsPluginWrapper(options: { defaultIcon?: keyof typeof icons } = {}) {
  const { defaultIcon } = options;

  return {
    name: 'fumadocs:lucide-icons',
    transformPageTree: {
      file: (node: any) => {
        if (node.icon !== undefined && typeof node.icon === 'string') {
          // Skip icons with colon (Iconify format)
          if (node.icon.includes(':')) {
            return node;
          }
          const Icon = icons[node.icon as keyof typeof icons];
          if (!Icon) {
            console.warn(`[lucide-icons-plugin] Unknown icon detected: ${node.icon}.`);
            return node;
          }
          node.icon = createElement(Icon);
        } else if (node.icon === undefined && defaultIcon) {
          node.icon = createElement(icons[defaultIcon]);
        }
        return node;
      },
      folder: (node: any) => {
        if (node.icon !== undefined && typeof node.icon === 'string') {
          // Skip icons with colon (Iconify format)
          if (node.icon.includes(':')) {
            return node;
          }
          const Icon = icons[node.icon as keyof typeof icons];
          if (!Icon) {
            console.warn(`[lucide-icons-plugin] Unknown icon detected: ${node.icon}.`);
            return node;
          }
          node.icon = createElement(Icon);
        } else if (node.icon === undefined && defaultIcon) {
          node.icon = createElement(icons[defaultIcon]);
        }
        return node;
      },
      separator: (node: any) => {
        if (node.icon !== undefined && typeof node.icon === 'string') {
          // Skip icons with colon (Iconify format)
          if (node.icon.includes(':')) {
            return node;
          }
          const Icon = icons[node.icon as keyof typeof icons];
          if (!Icon) {
            console.warn(`[lucide-icons-plugin] Unknown icon detected: ${node.icon}.`);
            return node;
          }
          node.icon = createElement(Icon);
        } else if (node.icon === undefined && defaultIcon) {
          node.icon = createElement(icons[defaultIcon]);
        }
        return node;
      },
    },
  };
}
```

### Step 5: Update the Source Configuration

Modify `<dir>/src/lib/source.ts` to use both plugins:

**Before:**
```ts
import { loader } from 'fumadocs-core/source';
import { lucideIconsPlugin } from 'fumadocs-core/source/lucide-icons';
import { docsContentRoute, docsImageRoute, docsRoute } from './shared';
import { defineDocs } from 'fumadocs-mdx/macro';
import { metaSchema, pageSchema } from 'fumadocs-core/source/schema';

// ... docs definition ...

export const source = loader({
  baseUrl: docsRoute,
  source: docs.toFumadocsSource(),
  plugins: [lucideIconsPlugin()],
});
```

**After:**
```ts
import { loader } from 'fumadocs-core/source';
import { lucideIconsPluginWrapper } from './lucide-plugin-wrapper';
import { iconifyPlugin } from './iconify-plugin';
import { docsContentRoute, docsImageRoute, docsRoute } from './shared';
import { defineDocs } from 'fumadocs-mdx/macro';
import { metaSchema, pageSchema } from 'fumadocs-core/source/schema';

// ... docs definition ...

export const source = loader({
  baseUrl: docsRoute,
  source: docs.toFumadocsSource(),
  plugins: [lucideIconsPluginWrapper(), iconifyPlugin()],
});
```

### Step 6: Update Root Layout

Modify `<dir>/src/app/layout.tsx` to render the IconRegistry component:

**Before:**
```tsx
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

**After:**
```tsx
import { RootProvider } from 'fumadocs-ui/provider/next';
import './global.css';
import { Inter } from 'next/font/google';
import { Body } from './layout.client';
import { IconRegistry } from '@/lib/icon-registry';

const inter = Inter({
  subsets: ['latin'],
});

export default function Layout({ children }: LayoutProps<'/'>) {
  return (
    <html lang="en" className={inter.className} suppressHydrationWarning>
      <Body>
        <IconRegistry />
        <RootProvider>{children}</RootProvider>
      </Body>
    </html>
  );
}
```

## Files Created/Modified

### Files Created:
1. **`<dir>/src/lib/icon-registry.ts`** - Icon registry component for offline icon loading
2. **`<dir>/src/lib/iconify-plugin.tsx`** - The Iconify plugin implementation
3. **`<dir>/src/lib/lucide-plugin-wrapper.tsx`** - Wrapper for Lucide that skips Iconify icons

### Files Modified:
1. **`<dir>/src/lib/source.ts`** - Updated imports and plugin configuration
2. **`<dir>/src/app/layout.tsx`** - Added IconRegistry component
3. **`<dir>/package.json`** - Updated dependencies (via package manager)

## Usage

Once implemented, you can use both icon systems in your `meta.json` files:

### Lucide Icons (Existing Format)
```json
{
  "title": "Getting Started",
  "icon": "Building"
}
```

### Iconify Icons (New Format)
```json
{
  "title": "Claude Documentation",
  "icon": "cbi:claude-clawd"
}
```

### Mixed Usage
```json
{
  "title": "Documentation",
  "pages": [
    {
      "title": "Overview",
      "icon": "Home"
    },
    {
      "title": "API Reference",
      "icon": "mdi:api"
    },
    {
      "title": "Configuration",
      "icon": "Settings"
    },
    {
      "title": "React Guide",
      "icon": "logos:react"
    }
  ]
}
```

## How It Works

### Offline Icon Loading

Icons are preloaded from locally installed packages rather than fetched from a CDN:

1. **Icon packages** (`@iconify-json/*`) contain the icon data as JSON files
2. **Icon Registry** imports these JSON files and registers them with `@iconify/react`
3. **Client-side registration** ensures icons are available before any component tries to render them
4. **No network requests** - all icon data is bundled with your application

### Icon Name Detection

The system automatically detects which icon library to use based on the icon name format:

- **Contains `:` (colon)** → Iconify icon (e.g., `mdi:home`, `bi:person`)
- **No `:` (colon)** → Lucide icon (e.g., `Building`, `Home`, `Settings`)

### Plugin Execution Order

1. **Lucide Plugin Wrapper** runs first
   - Processes icons without colons
   - Skips icons with colons (leaves them as strings)

2. **Iconify Plugin** runs second
   - Only processes string icons that contain colons
   - Converts them to Iconify React components

This ensures no conflicts between the two systems.

## Adding More Icon Sets

To add additional icon sets for offline use:

### Step 1: Install the Icon Set Package

```bash
cd "<dir>"
pnpm install -D @iconify-json/fa7-brands  # Font Awesome Brands example
pnpm install -D @iconify-json/<collection-name>
```

### Step 2: Import and Register in icon-registry.ts

```tsx
// Add to imports
import fa7BrandsIcons from '@iconify-json/fa7-brands/icons.json';
import anotherIconSet from '@iconify-json/<collection-name>/icons.json';

// Add to collections array
const collections = [
  fa7BrandsIcons,
  anotherIconSet,
  // Add more icon sets here
];
```

That's it! The icons will now work offline.

## Finding Iconify Icons

Browse available Iconify icons at: https://icon-sets.iconify.design/

**Note:** Remember to install the corresponding `@iconify-json/<collection-name>` package for any icon set you want to use offline.

## Troubleshooting

### Icons Not Rendering

If Iconify icons aren't rendering:

1. Verify the icon name format includes a colon (e.g., `fa7-brands:github`, not `fa7-brands-github`)
2. Check that you've installed the icon set package (e.g., `@iconify-json/fa7-brands`)
3. Ensure the icon set is imported and added to the `collections` array in `icon-registry.ts`
4. Verify the `IconRegistry` component is rendered in your root layout
5. Check the browser console for errors

### Lucide Icons Breaking

If Lucide icons stop working:

1. Ensure the `lucideIconsPluginWrapper` is listed before `iconifyPlugin` in the plugins array
2. Verify icon names don't contain colons
3. Check that `lucide-react` is still installed

### Type Errors

Run type checking to catch issues, from inside `<dir>`:

```bash
cd "<dir>"
pnpm run types:check
# or
npm run types:check
```

### Icons Not Loading Offline

If icons require internet connection:

1. Verify you've installed the specific `@iconify-json/*` package (not just `@iconify/react`)
2. Check that the icon set is imported in `icon-registry.ts`
3. Ensure the icon set is added to the `collections` array
4. The `IconRegistry` component must be rendered in the layout

## Notes

- All icon data is bundled with your application - no CDN requests
- Icons work completely offline once the packages are installed
- Bundle size increases with each icon set you add
- Only install icon sets you actually use to keep bundle size down
- Both plugins support a `defaultIcon` option for fallback icons
- The wrapper approach preserves all existing Lucide functionality
