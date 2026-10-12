**Caution: All the paths/directories mentioned in this file is for reference only because paths may change depending on different versions of fumadocs.**

## Path Convention

Throughout this guide, `<dir>` stands for the top-level directory of the Fumadocs app you are working in — the folder containing `package.json`, `src/`, and `content/`.

- If the app is at the repository root, `<dir>` is `.` and `<dir>/src/app/global.css` simply means `src/app/global.css`.
- If the app lives in a subfolder (e.g. `template/`, `apps/docs/`), substitute that name: `<dir>/src/app/global.css` → `template/src/app/global.css`.

Resolve `<dir>` once at the start, then apply it consistently to every path below.

**Note:** `<dir>` applies to filesystem paths only. Import specifiers beginning with `@/` (such as `@/lib/layout-config`) are TypeScript path aliases resolved relative to the app's own `src/` directory — leave them exactly as written, with no `<dir>` prefix.

# Task: Multiple Layout Support with Easy Switching

This task enables multiple layout presets in Fumadocs (Docs, Spacious, Glass, Notebook) and creates a centralized configuration system that allows you to easily switch between layouts by changing a single constant.

**Objective**: 
1. Customize the Fumadocs layout components locally
2. Enable the Spacious layout with its panel-based design
3. Create a unified configuration system for easy layout switching
4. Maintain all existing functionality (section colors, tab transforms, etc.)

## Prerequisites

- A working Fumadocs project with the default Docs layout
- Existing section color system in place (see Docs.md Task 2)
- Tab transform logic configured (see Docs.md Task 2, Step 5)

## Required Dependencies

No additional dependencies are required — this task uses existing packages.

## Task Steps

### Step 1: Customize Layout Components

Run the Fumadocs CLI customize command to create local copies of layout components:

**IMPORTANT FOR AI AGENTS**: Do NOT run these commands directly. You MUST prompt the human user to run them manually and wait for confirmation that they have been completed.

```bash
cd "<dir>"
pnpm dlx @fumadocs/cli@latest customize
```

When prompted, select all layout presets you want to support:
- **Layout presets**: Docs, Spacious, Glass, Notebook (select all)

This command creates a `<dir>/src/layouts/` directory with customizable layout components for each preset that you can modify independently of the fumadocs-ui package.

### Step 2: Create Layout Configuration File

Create a centralized configuration file for managing which layout is active.

**`<dir>/src/lib/layout-config.ts`:**
```typescript
/**
 * Centralized layout configuration
 * Change ACTIVE_LAYOUT to switch between different layout styles
 */

export type LayoutType = 'docs' | 'spacious' | 'glass' | 'notebook';

/**
 * CHANGE THIS VALUE TO SWITCH LAYOUTS
 * - 'docs': Default Docs Layout (compact, traditional)
 * - 'spacious': Spacious Layout (panel-based, less compact)
 * - 'glass': Glass Layout (glassmorphism design)
 * - 'notebook': Notebook Layout (notebook-style design)
 */
export const ACTIVE_LAYOUT: LayoutType = 'spacious';

export const LAYOUT_CONFIG = {
  docs: {
    name: 'Default Docs Layout',
    description: 'Compact traditional documentation layout',
    requiresCustomCSS: false,
  },
  spacious: {
    name: 'Spacious Layout',
    description: 'Panel-based layout with page in inset beside sidebar',
    requiresCustomCSS: true,
  },
  glass: {
    name: 'Glass Layout',
    description: 'Modern glassmorphism design',
    requiresCustomCSS: true,
  },
  notebook: {
    name: 'Notebook Layout',
    description: 'Notebook-style documentation layout',
    requiresCustomCSS: true,
  },
} as const;
```

### Step 3: Extract Shared Transform Logic

Create a shared utility for the tab transform logic so it can be used by all layout types, with special handling for the Glass layout's icon sizing.

**`<dir>/src/lib/layout-utils.tsx`:**
```typescript
import { source } from '@/lib/source';
import { getSection } from '@/lib/navigation';
import type { CSSProperties } from 'react';
import type * as PageTree from 'fumadocs-core/page-tree';
import type { LayoutType } from './layout-config';

/**
 * Shared tab transform logic used by all layouts
 * Applies section-based coloring to tab icons
 */
export function createTabTransform(layoutType?: LayoutType) {
  return (option: any, node: PageTree.Node) => {
    const meta = source.getNodeMeta(node);
    if (!meta || !node.icon) return option;

    const color = `var(--${getSection(meta.path)}-color, var(--color-fd-foreground))`;

    // Glass layout needs constrained icon size
    if (layoutType === 'glass') {
      return {
        ...option,
        icon: (
          <div
            className="[&_svg]:size-4 rounded-lg text-(--tab-color) max-md:bg-(--tab-color)/10 max-md:border max-md:p-1.5"
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
    }

    // Default for docs, spacious, notebook
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
  };
}
```

### Step 4: Create Unified Layout Wrapper

Create a client component wrapper that dynamically selects the correct layout based on configuration.

**`<dir>/src/app/docs/layout.client.tsx`:**
```typescript
'use client';

import { DocsLayout as DocsLayoutDefault } from 'fumadocs-ui/layouts/docs';
import { DocsLayout as DocsLayoutSpacious } from '@/layouts/spacious';
import { DocsLayout as DocsLayoutGlass } from '@/layouts/glass';
import { DocsLayout as DocsLayoutNotebook } from '@/layouts/notebook';
import { createTabTransform } from '@/lib/layout-utils';
import type { ReactNode } from 'react';
import type { BaseLayoutProps } from 'fumadocs-ui/layouts/shared';
import type * as PageTree from 'fumadocs-core/page-tree';
import type { LayoutType } from '@/lib/layout-config';

interface DocsLayoutWrapperProps extends BaseLayoutProps {
  layoutType: LayoutType;
  tree: PageTree.Root;
  children: ReactNode;
}

/**
 * Unified layout wrapper that switches between all available layouts
 * based on the layoutType prop
 */
export function DocsLayoutWrapper({ layoutType, children, ...props }: DocsLayoutWrapperProps) {
  let Layout;

  switch (layoutType) {
    case 'spacious':
      Layout = DocsLayoutSpacious;
      break;
    case 'glass':
      Layout = DocsLayoutGlass;
      break;
    case 'notebook':
      Layout = DocsLayoutNotebook;
      break;
    case 'docs':
    default:
      Layout = DocsLayoutDefault;
      break;
  }

  return (
    <Layout
      {...props}
      tabs={{
        transform: createTabTransform(layoutType),
      }}
    >
      {children}
    </Layout>
  );
}
```

**Why this is a client component:**
- The `DocsLayout` components from all sources are client components
- Functions (like `transform`) can only be passed to client components in Next.js 16
- This wrapper handles the dynamic layout selection on the client side

### Step 5: Update Docs Layout

Simplify the docs layout to use the unified wrapper with the active layout configuration.

**IMPORTANT**: The `pnpm dlx @fumadocs/cli@latest customize` command automatically updates `<dir>/src/app/docs/layout.tsx` to import from `@/layouts/docs` instead of `fumadocs-ui/layouts/docs`. This is expected behavior.

**`<dir>/src/app/docs/layout.tsx`:**

Replace the entire file content with:

```typescript
import { source } from '@/lib/source';
import { DocsLayoutWrapper } from './layout.client';
import { baseOptions } from '@/lib/layout.shared';
import { ACTIVE_LAYOUT } from '@/lib/layout-config';

export default function Layout({ children }: LayoutProps<'/docs'>) {
  return (
    <DocsLayoutWrapper layoutType={ACTIVE_LAYOUT} tree={source.getPageTree()} {...baseOptions()}>
      {children}
    </DocsLayoutWrapper>
  );
}
```

**Note**: The customize command will have changed the import from `fumadocs-ui/layouts/docs` to `@/layouts/docs`. We're replacing this with our unified wrapper that supports all layout types.

### Step 6: Create Dynamic Page Components Export

Create a module that automatically exports the correct page components based on the active layout.

**`<dir>/src/lib/page-components.ts`:**
```typescript
import { ACTIVE_LAYOUT } from '@/lib/layout-config';

/**
 * Dynamic page component exports based on active layout
 * Automatically switches between Docs, Spacious, Glass, and Notebook page components
 */

// Layout-specific imports
import * as DocsPageComponents from 'fumadocs-ui/layouts/docs/page';
import * as SpaciousPageComponents from '@/layouts/spacious/page';
import * as GlassPageComponents from '@/layouts/glass/page';
import * as NotebookPageComponents from '@/layouts/notebook/page';

// Select the correct components based on active layout
let components;

switch (ACTIVE_LAYOUT) {
  case 'spacious':
    components = SpaciousPageComponents;
    break;
  case 'glass':
    components = GlassPageComponents;
    break;
  case 'notebook':
    components = NotebookPageComponents;
    break;
  case 'docs':
  default:
    components = DocsPageComponents;
    break;
}

// Re-export all page components
export const DocsPage = components.DocsPage;
export const DocsBody = components.DocsBody;
export const DocsDescription = components.DocsDescription;
export const DocsTitle = components.DocsTitle;
export const MarkdownCopyButton = components.MarkdownCopyButton;
export const ViewOptionsPopover = components.ViewOptionsPopover;

// Type exports for convenience
export type { DocsPageProps } from 'fumadocs-ui/layouts/docs/page';
```

### Step 7: Update Page Component Imports

Update the docs page file to use the dynamic page component exports.

**IMPORTANT**: Similar to Step 5, the customize command will have changed the import path in `<dir>/src/app/docs/[[...slug]]/page.tsx` from `fumadocs-ui/layouts/docs/page` to `@/layouts/docs/page`.

**`<dir>/src/app/docs/[[...slug]]/page.tsx`:**

Change the import from:
```typescript
import {
  DocsBody,
  DocsDescription,
  DocsPage,
  DocsTitle,
  MarkdownCopyButton,
  ViewOptionsPopover,
} from '@/layouts/docs/page';
```

To:
```typescript
import {
  DocsBody,
  DocsDescription,
  DocsPage,
  DocsTitle,
  MarkdownCopyButton,
  ViewOptionsPopover,
} from '@/lib/page-components';
```

**Note**: The original import may have been `fumadocs-ui/layouts/docs/page` before running the customize command, but it will be `@/layouts/docs/page` after. Either way, change it to use `@/lib/page-components`.

Leave the rest of the file unchanged.

### Step 8: Add Layout CSS Imports

Add the required CSS imports for all custom layouts to your global styles.

**`<dir>/src/app/global.css`:**

Add the layout CSS imports after the `@import 'tailwindcss';` line:

```css
@import 'tailwindcss';
@import 'fumadocs-ui/css/generated/spacious.css';
@import 'fumadocs-ui/css/generated/glass.css';
@import 'fumadocs-ui/css/generated/notebook.css';
@import 'fumadocs-ui/css/neutral.css';
@import 'fumadocs-ui/css/preset.css';

/* Rest of your CSS remains unchanged */
```

**Note**: The Docs layout doesn't require a custom CSS import. If you want to optimize bundle size, you can comment out the CSS imports for layouts you're not actively using:
```css
/* @import 'fumadocs-ui/css/generated/spacious.css'; */
/* @import 'fumadocs-ui/css/generated/glass.css'; */
/* @import 'fumadocs-ui/css/generated/notebook.css'; */
```

### Step 9: Fix Search Bar Order in Spacious Layout

By default, the spacious layout places the section switching bar above the search bar in the sidebar. To match the docs layout order (search bar first, then section switching bar), make the following adjustment:

**`<dir>/src/layouts/spacious/slots/sidebar.tsx`:**

Find the section around line 84 that looks like this:

```typescript
        <div className="flex flex-col gap-2 px-3 empty:hidden">
          <TabsMenu />
          {slots.searchTrigger && (
            <slots.searchTrigger.full
              hideIfDisabled
              className="h-10 gap-3 rounded-xl ps-[calc(--spacing(2.5)-1px)] pe-2 [&_svg]:shrink-0"
            />
          )}
        </div>
```

Replace it with (swap the order of `searchTrigger` and `TabsMenu`):

```typescript
        <div className="flex flex-col gap-2 px-3 empty:hidden">
          {slots.searchTrigger && (
            <slots.searchTrigger.full
              hideIfDisabled
              className="h-10 gap-3 rounded-xl ps-[calc(--spacing(2.5)-1px)] pe-2 [&_svg]:shrink-0"
            />
          )}
          <TabsMenu />
        </div>
```

This ensures the search bar appears above the section switching bar, matching the visual order of the docs layout.

## How The System Works

### Layout Switching Flow

1. **Single configuration point**: `ACTIVE_LAYOUT` in `<dir>/src/lib/layout-config.ts` controls the entire layout system
2. **Layout wrapper**: `DocsLayoutWrapper` reads the layout type and selects the appropriate layout component
3. **Page components**: `page-components.ts` automatically exports the correct components for the active layout
4. **Transform logic**: Shared tab transform logic works with all layouts, with special handling for Glass layout icon sizing

### Request Flow

1. **User visits** `/docs/sec-2/test`
2. **Docs layout** renders `DocsLayoutWrapper` with `layoutType={ACTIVE_LAYOUT}`
3. **Wrapper selects** the correct layout component based on the layout type
4. **Transform applied**: Shared `createTabTransform()` colors the tab icons
5. **Page renders** using components from `page-components.ts` matching the active layout
6. **Section colors** work as before through the existing `Body` component

### Why This Architecture

- **Single toggle point**: Change one constant to switch entire layout system
- **No code duplication**: Transform logic shared between layouts
- **Layout-specific optimizations**: Glass layout gets proper icon sizing automatically
- **Type-safe**: TypeScript ensures props match between layouts
- **Easy maintenance**: Layout-specific code isolated in one wrapper
- **Future-proof**: Easy to add more layout types

## Expected Results

After completing this task:

### With Spacious Layout Active (`ACTIVE_LAYOUT = 'spacious'`)
- **Panel-based design**: Page content appears in an inset panel beside the sidebar
- **Page scrolling**: Content scrolls inside the panel on desktop (not the full window)
- **Header actions**: Page-level actions appear at the top of the panel
- **Less compact**: More breathing room compared to default layout
- **Section colors**: Still work correctly through the existing system

### With Docs Layout Active (`ACTIVE_LAYOUT = 'docs'`)
- **Traditional design**: Compact, full-width content layout
- **Window scrolling**: Page scrolls the entire window
- **Section colors**: Work as before
- **All features**: Maintain existing functionality

### Both Layouts
- **Tab transforms**: Icons colored by section
- **Theme support**: Light/dark mode adaptation
- **All existing features**: Navigation, search, sidebar, etc.

## How to Switch Layouts

To switch between layouts, simply change one line in `<dir>/src/lib/layout-config.ts`:

```typescript
// For Spacious layout
export const ACTIVE_LAYOUT: LayoutType = 'spacious';

// For default Docs layout
export const ACTIVE_LAYOUT: LayoutType = 'docs';
```

**CSS Note**: When switching to 'docs', remember to comment out the Spacious CSS import in `global.css`:
```css
/* @import 'fumadocs-ui/css/generated/spacious.css'; */
```

## Files Created

1. `<dir>/src/layouts/` — Directory created by customize command containing local layout components
2. `<dir>/src/lib/layout-config.ts` — Central layout configuration
3. `<dir>/src/lib/layout-utils.tsx` — Shared transform logic
4. `<dir>/src/lib/page-components.ts` — Dynamic page component exports
5. `<dir>/src/app/docs/layout.client.tsx` — Unified layout wrapper

## Files Modified

1. `<dir>/src/app/docs/layout.tsx` — Updated to use layout wrapper with configuration
2. `<dir>/src/app/docs/[[...slug]]/page.tsx` — Updated page component imports
3. `<dir>/src/app/global.css` — Added Spacious CSS import

## Customization Guide

**Note**: This section is for human readers to understand how to customize the implementation. AI agents should NOT attempt to implement these customizations unless explicitly asked by the user.

### Modifying the Spacious Layout

Since the customize command created local copies of the layout components in `<dir>/src/layouts/`, you can now:

- Customize the Spacious layout structure by editing `<dir>/src/layouts/spacious/index.tsx`
- Modify page components in `<dir>/src/layouts/spacious/page/index.tsx`
- Adjust header, sidebar, or other slots in `<dir>/src/layouts/spacious/slots/`

These changes only affect the Spacious layout — the default Docs layout remains unchanged.

### Adding More Layout Types

To add a third layout option (e.g., 'glass'):

1. **Run customize command** again and select the Glass preset
2. **Update `layout-config.ts`**:
   ```typescript
   export type LayoutType = 'docs' | 'spacious' | 'glass';
   ```
3. **Update the layout wrapper** in `layout.client.tsx` to handle the new type
4. **Update page components** in `page-components.ts` if needed
5. **Add CSS imports** for the new layout in `global.css`

### Customizing Transform Logic

The shared transform logic in `layout-utils.tsx` can be customized to:
- Change icon wrapper styles
- Modify color application
- Add hover effects
- Adjust mobile/desktop differences

Changes apply to both layouts automatically.

## Troubleshooting

### Layout Not Switching
- Verify `ACTIVE_LAYOUT` is set correctly in `layout-config.ts`
- Clear Next.js cache: `rm -rf .next` and restart dev server
- Check that imports in `layout.client.tsx` are correct

### Page Components Error
- Ensure `page-components.ts` imports match the actual exports from both layout packages
- Verify the page file is importing from `@/lib/page-components`

### Styles Not Applied
- Check that Spacious CSS is imported in `global.css` when using Spacious layout
- Verify import order (Spacious CSS should come after `tailwindcss`)

### Transform Function Not Working
- Functions can only be passed to client components in Next.js 16
- Ensure `layout.client.tsx` has `'use client';` directive
- Verify `createTabTransform()` is being called correctly

## Pattern Reference

### Official Fumadocs Approach

The official Fumadocs documentation uses multiple layout files for different sections. This implementation improves on that by:

- **Centralizing configuration**: One constant controls the entire system
- **Reducing duplication**: Shared logic extracted to utilities
- **Type safety**: TypeScript ensures consistency
- **Easy switching**: No need to change multiple files

### Key Design Decisions

1. **Client component wrapper**: Required because Next.js 16 doesn't allow passing functions to client components from server components
2. **Dynamic exports**: Page components module exports the right components automatically
3. **Shared transform**: Tab coloring logic works identically in both layouts
4. **CSS commenting**: Manual step required because conditional CSS imports aren't supported

### Why Not Use Dynamic Imports?

Dynamic imports (`import()`) are async, but layout components need to be available synchronously. The conditional selection approach used here is:
- Synchronous
- Type-safe
- Tree-shakeable (unused layout code can be eliminated by the bundler)

## Notes

- The customize command creates copies that you own — fumadocs-ui updates won't affect them
- Both layouts can coexist in your project, switched via configuration
- Section colors and all existing customizations work with both layouts
- The wrapper pattern can extend to support more layouts in the future
- Consider the bundle size impact of including multiple layout types
