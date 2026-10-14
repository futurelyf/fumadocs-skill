**Caution: All the paths/directories mentioned in this file is for reference only because paths may change depending on different versions of fumadocs.**

## Path Convention

Throughout this guide, `<dir>` stands for the top-level directory of the Fumadocs app you are working in — the folder containing `package.json`, `src/`, and `content/`.

- If the app is at the repository root, `<dir>` is `.` and `<dir>/src/app/global.css` simply means `src/app/global.css`.
- If the app lives in a subfolder (e.g. `template/`, `apps/docs/`), substitute that name: `<dir>/src/app/global.css` → `template/src/app/global.css`.

Resolve `<dir>` once at the start, then apply it consistently to every path below.

**Note:** `<dir>` applies to filesystem paths only. Import specifiers beginning with `@/` (such as `@/lib/source`) are TypeScript path aliases resolved relative to the app's own `src/` directory — leave them exactly as written, with no `<dir>` prefix.

# Task: Adding Blog Functionality to Fumadocs

This task adds a complete blog system to your Fumadocs site with post listings, individual post pages, tag filtering, author profiles, and an interactive table of contents.

**Objective**: 
1. Install blog components and utilities using the Fumadocs CLI
2. Configure blog collection and author profiles
3. Create blog route pages with banner design
4. Add sample blog posts with proper frontmatter

## Prerequisites

- A working Fumadocs project with Next.js App Router
- Existing HomeLayout configured (typically in `<dir>/src/layouts/home`)
- MDX components set up (`<dir>/src/components/mdx`)

## Required Dependencies

No additional dependencies are required — the CLI adds necessary utilities and components.

## Task Steps

### Step 1: Install Blog Components

Run the Fumadocs CLI to add blog functionality:

**IMPORTANT FOR AI AGENTS**: Do NOT run this command directly. You MUST prompt the human user to run it manually and wait for confirmation that it has been completed.

```bash
cd "<dir>"
pnpm dlx @fumadocs/cli@latest add blog
```

This command creates:
- `<dir>/src/components/blog/` - Blog UI components
- `<dir>/src/components/blog/panel.tsx` - Floating TOC panel
- `<dir>/src/components/toc/` - TOC context and rendering
- `<dir>/src/lib/blog.ts` - Blog utilities and helpers

### Step 2: Configure Blog Collection

Add the blog collection to your source configuration file.

**`<dir>/src/lib/source.ts`:**

Before:
```typescript
import { llms, loader } from 'fumadocs-core/source';
import { lucideIconsPluginWrapper } from './lucide-plugin-wrapper';
import { iconifyPlugin } from './iconify-plugin';
import { docsContentRoute, docsImageRoute, docsRoute } from './shared';
import { defineDocs } from 'fumadocs-mdx/macro';
import { metaSchema, pageSchema } from 'fumadocs-core/source/schema';

const docs = defineDocs({
  dir: 'content/docs',
  docs: {
    schema: pageSchema,
    postprocess: {
      includeProcessedMarkdown: true,
    },
  },
  meta: {
    schema: metaSchema,
  },
});

// See https://fumadocs.dev/docs/headless/source-api for more info
export const source = loader({
  baseUrl: docsRoute,
  source: docs.toFumadocsSource(),
  plugins: [lucideIconsPluginWrapper(), iconifyPlugin()],
});

export const docsLlms = llms(source, {
  renderPage: async (page) => `# ${page.data.title} (${page.url})

${await page.data.getText('processed')}`,
});
```

After:
```typescript
import { llms, loader } from 'fumadocs-core/source';
import { lucideIconsPluginWrapper } from './lucide-plugin-wrapper';
import { iconifyPlugin } from './iconify-plugin';
import { docsContentRoute, docsImageRoute, docsRoute } from './shared';
import { defineDocs, defineCollections } from 'fumadocs-mdx/macro';
import { metaSchema, pageSchema } from 'fumadocs-core/source/schema';
import { type BlogAuthor, blogPageSchema } from './blog';

const docs = defineDocs({
  dir: 'content/docs',
  docs: {
    schema: pageSchema,
    postprocess: {
      includeProcessedMarkdown: true,
    },
  },
  meta: {
    schema: metaSchema,
  },
});

const blog = defineCollections({
  type: 'doc',
  dir: 'content/blog',
  schema: blogPageSchema,
});

// See https://fumadocs.dev/docs/headless/source-api for more info
export const source = loader({
  baseUrl: docsRoute,
  source: docs.toFumadocsSource(),
  plugins: [lucideIconsPluginWrapper(), iconifyPlugin()],
});

export const blogSource = loader({
  baseUrl: '/blog',
  source: blog.toFumadocsSource(),
});

export const blogAuthors: Record<string, BlogAuthor> = {
  futurelyf: {
    name: 'Yifan Liang',
    title: 'Developer',
    url: 'https://github.com/futurelyf',
    image: 'https://avatars.githubusercontent.com/futurelyf',
  },
};

export const docsLlms = llms(source, {
  renderPage: async (page) => `# ${page.data.title} (${page.url})

${await page.data.getText('processed')}`,
});
```

**Key changes:**
1. Add `defineCollections` to the import from `'fumadocs-mdx/macro'`
2. Import blog types: `type BlogAuthor` and `blogPageSchema` from `'./blog'`
3. Define the `blog` collection with schema after the `docs` definition
4. Export `blogSource` loader after the main `source` export
5. Export `blogAuthors` with author profile definitions

### Step 3: Configure Remote Image Patterns

If using remote author avatars (like GitHub avatars), configure Next.js to allow them.

**`<dir>/next.config.mjs`:**

Add the images configuration:

Before:
```javascript
const config = {
  reactStrictMode: true,
};
```

After:
```javascript
const config = {
  reactStrictMode: true,
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'avatars.githubusercontent.com',
        port: '',
      },
    ],
    dangerouslyAllowSVG: true,
    dangerouslyAllowLocalIP: true,
    contentSecurityPolicy: "default-src 'self'; script-src 'none'; sandbox;",
  },
};
```

**Important configuration notes:**
- `dangerouslyAllowSVG: true` - Required if GitHub returns SVG avatars
- `dangerouslyAllowLocalIP: true` - Required if GitHub avatar URLs resolve to private IP addresses in your network environment (common in some corporate/development setups)
- `contentSecurityPolicy` - Security policy for handling SVG images

### Step 4: Create Blog Layout

Create a layout file for the blog routes that uses the HomeLayout.

**`<dir>/src/app/blog/layout.tsx`:**

```typescript
import { HomeLayout } from '@/layouts/home';
import { baseOptions } from '@/lib/layout.shared';

export default function Layout({ children }: LayoutProps<'/blog'>) {
  return <HomeLayout {...baseOptions()}>{children}</HomeLayout>;
}
```

**Note**: The blog uses `HomeLayout` (the marketing/landing page layout) rather than `DocsLayout`. This is because blog pages should share the same header, footer, and styling as your homepage.

### Step 5: Create Main Blog Page

Create the main blog listing page with a banner.

**`<dir>/src/app/blog/page.tsx`:**

```typescript
import Image from 'next/image';
import Link from 'fumadocs-core/link';
import { TagsIcon } from 'lucide-react';
import { blogAuthors, blogSource } from '@/lib/source';
import { getPosts } from '@/lib/blog';
import { BlogCards } from '@/components/blog';
import BannerImage from './banner.png';

export default function Page() {
  const posts = getPosts(blogSource.getPages(), blogAuthors);

  return (
    <div className="mx-auto w-full max-w-page px-4 pb-12 md:py-12">
      <div className="relative dark mb-4 aspect-[3.2] p-8 z-2 md:p-12">
        <Image
          src={BannerImage}
          priority
          alt="banner"
          className="absolute inset-0 size-full -z-1 object-cover"
        />
        <h1 className="mb-4 text-3xl text-landing-foreground font-mono font-medium">
          Future Studio Blog
        </h1>
        <p className="text-sm font-mono text-landing-foreground-200">
          Latest announcements of Future Studio
        </p>
        <Link
          href="/blog/tags"
          className="mt-6 inline-flex items-center gap-2 rounded-lg border bg-fd-secondary/50 px-3 py-1.5 text-sm font-mono text-landing-foreground transition-colors hover:bg-fd-accent"
        >
          <TagsIcon className="size-4" />
          All Tags
        </Link>
      </div>
      <BlogCards posts={posts} />
    </div>
  );
}
```

**Key features:**
- Banner image with dark overlay for text visibility
- Grid of blog post cards with images
- Link to tags page
- Responsive design with proper spacing

### Step 6: Add Banner Image

Copy a banner image for the blog index page. The banner should be approximately 3.2:1 aspect ratio (e.g., 1600x500px).

**IMPORTANT - For AI agents:** The `skill/` folder is located at the **REPOSITORY ROOT**, parallel to `<dir>`, NOT inside it. Check your current working directory first with `pwd`:

**Directory structure:**
```
fumadocs/                    ← Repository root
├── skill/                   ← Skill files are HERE
│   └── Image/
│       └── banner.png
└── <dir>/                   ← Your target directory
    └── src/app/blog/
```

**Command based on your current location:**

```bash
# Step 1: Check where you are
pwd

# Step 2a: If at repository root (path ends with 'fumadocs'):
cp "skill/Image/banner.png" "<dir>/src/app/blog/banner.png"

# Step 2b: If already inside <dir> (path ends with 'fumadocs/<dir-name>' or similar):
cp "../skill/Image/banner.png" "src/app/blog/banner.png"
```

**Rule:** If you're inside `<dir>`, use `../skill/` to go up one level to access the skill folder.

If you do not have the image in `skill/Image/`, substitute any wide image — the container is `aspect-[3.2]`, so roughly 3.2:1 works best.

Place it at: `<dir>/src/app/blog/banner.png`

### Step 7: Create Individual Blog Post Page

Create the page for individual blog posts with TOC support.

**`<dir>/src/app/blog/[slug]/page.tsx`:**

```typescript
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { blogAuthors, blogSource } from '@/lib/source';
import { findPost, getPosts } from '@/lib/blog';
import { getMDXComponents } from '@/components/mdx';
import {
  BlogAuthors,
  BlogBackLink,
  BlogBody,
  BlogDate,
  BlogDescription,
  BlogHeader,
  BlogPage,
  BlogPostNav,
  BlogTags,
  BlogTitle,
} from '@/components/blog';

export default async function Page(props: { params: Promise<{ slug: string }> }) {
  const params = await props.params;
  const page = blogSource.getPage([params.slug]);
  const found = page && findPost(getPosts(blogSource.getPages(), blogAuthors), page.url);
  if (!page || !found) notFound();

  const { post, newer, older } = found;
  const MDX = page.data.body;

  return (
    <BlogPage toc={page.data.toc}>
      <BlogHeader>
        <BlogBackLink href="/blog" />
        <BlogTitle>{post.title}</BlogTitle>
        <BlogDescription>{post.description}</BlogDescription>
        <BlogAuthors authors={post.authors} />
        <div className="flex w-full flex-row flex-wrap items-center gap-2">
          <BlogTags tags={post.tags} />
          {post.date && (
            <BlogDate date={post.date} className="ms-auto text-sm text-fd-muted-foreground" />
          )}
        </div>
      </BlogHeader>
      <BlogBody>
        <MDX components={getMDXComponents()} />
      </BlogBody>
      <BlogPostNav newer={newer} older={older} />
    </BlogPage>
  );
}

export async function generateMetadata(props: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const params = await props.params;
  const page = blogSource.getPage([params.slug]);

  if (!page) notFound();

  return {
    title: page.data.title,
    description: page.data.description ?? 'The library for building documentation sites',
  };
}

export function generateStaticParams(): { slug: string }[] {
  return blogSource.getPages().map((page) => ({
    slug: page.slugs[0],
  }));
}
```

**Key features:**
- Back link to blog index
- Full post metadata (title, description, authors, tags, date)
- TOC integration with floating panel
- Newer/older post navigation
- SEO metadata generation
- Static generation support

### Step 8: Create Tags Index Page

Create the page listing all blog tags.

**`<dir>/src/app/blog/tags/page.tsx`:**

```typescript
import { blogSource } from '@/lib/source';
import { getPosts, getTags } from '@/lib/blog';
import {
  BlogBackLink,
  BlogDescription,
  BlogHeader,
  BlogTagCards,
  BlogTitle,
} from '@/components/blog';

export const metadata = {
  title: 'Blog Tags',
  description: 'All tags of Future Studio Blog',
};

export default function Page() {
  const tags = getTags(getPosts(blogSource.getPages()));

  return (
    <div className="mx-auto flex w-full max-w-page flex-col gap-4 px-4 py-12">
      <BlogHeader>
        <BlogBackLink href="/blog" />
        <BlogTitle>All Tags</BlogTitle>
        <BlogDescription>{tags.length} tags in total.</BlogDescription>
      </BlogHeader>
      <BlogTagCards tags={tags} />
    </div>
  );
}
```

**Key features:**
- Back link to blog index
- Tag count display
- Grid of tag cards with post counts

### Step 9: Create Individual Tag Page

Create the page showing posts filtered by tag.

**`<dir>/src/app/blog/tags/[tag]/page.tsx`:**

```typescript
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { blogAuthors, blogSource } from '@/lib/source';
import { getPosts, getTagPosts, getTags } from '@/lib/blog';
import { BlogBackLink, BlogCards, BlogDescription, BlogHeader, BlogTitle } from '@/components/blog';

export default async function Page(props: { params: Promise<{ tag: string }> }) {
  const params = await props.params;
  const result = getTagPosts(getPosts(blogSource.getPages(), blogAuthors), params.tag);
  if (!result) notFound();

  return (
    <div className="mx-auto flex w-full max-w-page flex-col gap-4 px-4 py-12">
      <BlogHeader>
        <BlogBackLink href="/blog/tags">All Tags</BlogBackLink>
        <BlogTitle>{result.tag.name}</BlogTitle>
        <BlogDescription>{result.posts.length} matching posts.</BlogDescription>
      </BlogHeader>
      <BlogCards posts={result.posts} />
    </div>
  );
}

export async function generateMetadata(props: { params: Promise<{ tag: string }> }): Promise<Metadata> {
  const params = await props.params;
  const result = getTagPosts(getPosts(blogSource.getPages()), params.tag);
  if (!result) notFound();

  return {
    title: `${result.tag.name} - Blog`,
    description: `Posts tagged ${result.tag.name} on Future Studio Blog`,
  };
}

export function generateStaticParams(): { tag: string }[] {
  return getTags(getPosts(blogSource.getPages())).map((tag) => ({ tag: tag.slug }));
}
```

**Key features:**
- Back link to tags index
- Post count for the tag
- Filtered post cards
- SEO metadata generation
- Static generation for all tags

### Step 10: Create Sample Blog Posts

Create sample blog posts to demonstrate the system.

#### File: `<dir>/content/blog/welcome.mdx`

```mdx
---
title: Welcome to Our Blog
description: Getting started with our documentation blog
authors: [futurelyf]
tags: [Announcement, Getting Started]
date: 2026-10-13
# image: /blog/welcome.png
---

## Welcome!

Welcome to our documentation blog! This is where we'll share updates, tutorials, and insights.

## What to Expect

In this blog, you'll find:

- **Release announcements** - Latest features and improvements
- **Tutorials** - How to make the most of our tools
- **Best practices** - Tips and tricks from our team

Thank you for being part of our journey!
```

#### File: `<dir>/content/blog/best-practices.mdx`

```mdx
---
title: Building Better Documentation
description: Tips and best practices for creating great docs
authors: [futurelyf]
tags: [Announcement, Tutorial]
date: 2026-10-12
# image: /blog/best-practices.png
---

## The Art of Documentation

Great documentation is more than just writing what your code does. It's about creating an experience that helps users understand and succeed.

## Essential Principles

### Start with the User

Always consider who will be reading your documentation:

- What's their skill level?
- What problem are they trying to solve?
- What context do they need?

### Show, Don't Just Tell

Code examples are crucial. Every concept should have working examples that demonstrate real usage.

## Conclusion

Documentation is a product feature. Invest in it, and your users will thank you.
```

**Frontmatter fields explained:**
- `title` - Post title (required)
- `description` - Post description/summary (required)
- `date` - Publication date in YYYY-MM-DD format (posts are sorted by this)
- `tags` - Array of tag names (each gets its own page under `/blog/tags`)
- `authors` - Array of author keys from `blogAuthors` (unknown keys become plain names)
- `image` - Cover image URL shown on post cards (can be local or remote)

### Step 11: Add Blog Section Colors

This step adds custom color theming for the blog section and proper text colors for the banner overlay.

#### Add Blog Route Constant

**`<dir>/src/lib/shared.ts`:**

Add the blog route constant to your existing exports:

```typescript
export const blogRoute = '/blog';
```

#### Add Blog Color Variables and Page Width

**`<dir>/src/app/global.css`:**

Add the `@theme` block with `--spacing-page` to set proper blog page width, and add blog-specific color variables:

```css
@theme {
  --spacing-page: 1436px;
}

:root {
  --banner-text-color-up: #59592a;
  --banner-text-color-down: #a8a866;
  --blog-color: hsl(25, 55%, 35%);
}

.dark {
  --banner-text-color-up: #e4e2d0;
  --banner-text-color-down: #b7af7e;
  --blog-color: #d9b38c;
}

.blog {
  --color-fd-primary: var(--blog-color);
}
```

**Key additions:**
- `@theme { --spacing-page: 1436px; }` - Sets the maximum width for `max-w-page` used in all blog pages, providing generous left and right margins
- Banner text color variables for light/dark modes
- Blog section color theming

#### Update Section Detection

**`<dir>/src/app/layout.client.tsx`:**

Update the `Body` component to detect blog routes and apply the `.blog` class:

```typescript
'use client';

import { usePathname } from 'next/navigation';
import { RootProvider } from 'fumadocs-ui/provider';
import { FC, ReactNode } from 'react';
import { docsRoute } from '@/lib/shared';

export const Body: FC<{ children: ReactNode }> = ({ children }) => {
  const pathname = usePathname();
  let section = '';
  
  if (pathname.startsWith(docsRoute)) {
    const urlAfterDocs = pathname.slice(docsRoute.length);
    if (urlAfterDocs.startsWith('/sec-1')) section = 'sec-1';
    else if (urlAfterDocs.startsWith('/sec-2')) section = 'sec-2';
  } else if (pathname.startsWith('/blog')) {
    section = 'blog';
  }

  return (
    <body className={section}>
      <RootProvider>{children}</RootProvider>
    </body>
  );
};
```

**Key change:** Added the `else if (pathname.startsWith('/blog'))` condition to detect blog routes and apply the `'blog'` section class.

#### Update Banner Text Colors

**`<dir>/src/app/blog/page.tsx`:**

Update the banner section in the blog index page to use the new color variables:

```typescript
<div className="relative dark mb-4 aspect-[3.2] p-8 z-2 md:p-12">
  <Image
    src={BannerImage}
    priority
    alt="banner"
    className="absolute inset-0 size-full -z-1 object-cover"
  />
  <h1 className="mb-4 text-3xl font-mono font-medium text-(--banner-text-color-up)">
    Future Studio Blog
  </h1>
  <p className="font-mono text-sm text-(--banner-text-color-down)">
    Latest announcements of Future Studio
  </p>
  <Link
    href="/blog/tags"
    className="mt-6 inline-flex items-center gap-2 rounded-lg border bg-fd-secondary/50 px-3 py-1.5 text-sm font-mono transition-colors hover:bg-fd-accent"
    style={{ color: 'var(--banner-text-color-up)' }}
  >
    <TagsIcon className="size-4" />
    All Tags
  </Link>
</div>
```

**Key changes:**
- Heading uses `text-(--banner-text-color-up)` for primary banner text
- Subtitle uses `text-(--banner-text-color-down)` for secondary banner text
- Button text uses inline style `style={{ color: 'var(--banner-text-color-up)' }}` for proper color

### Step 12: Add Navigation Links

This step adds navigation links between docs and blog sections in the navbar.

#### Create Navigation Helper Function

**`<dir>/src/lib/layout.shared.tsx`:**

Add a `homeOptions()` function that extends `baseOptions()` with navigation links:

```typescript
import type { BaseLayoutProps } from 'fumadocs-ui/layouts/shared';
import { appName, blogRoute, docsRoute, gitConfig } from './shared';
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
    githubUrl: `https://github.com/${gitConfig.user}`,
  };
}

export function homeOptions(): BaseLayoutProps {
  return {
    ...baseOptions(),
    links: [
      {
        text: 'Docs',
        url: docsRoute,
      },
      {
        text: 'Blog',
        url: blogRoute,
      },
    ],
  };
}
```

#### Update Home and Blog Layouts

**`<dir>/src/app/(home)/layout.tsx`:**

```typescript
import { HomeLayout } from '@/layouts/home';
import { homeOptions } from '@/lib/layout.shared';

export default function Layout({ children }: LayoutProps<'/'>) {
  return <HomeLayout {...homeOptions()}>{children}</HomeLayout>;
}
```

**`<dir>/src/app/blog/layout.tsx`:**

```typescript
import { HomeLayout } from '@/layouts/home';
import { homeOptions } from '@/lib/layout.shared';

export default function Layout({ children }: { children: React.ReactNode }) {
  return <HomeLayout {...homeOptions()}>{children}</HomeLayout>;
}
```

**Key change:** Both layouts now use `homeOptions()` instead of `baseOptions()` to get the Docs and Blog navigation links in the navbar.

### Step 13: Add Blog Icon to Docs Sidebar

This step adds icon buttons in the docs sidebar footer for easy navigation to GitHub and the blog.

**Prerequisites:** Install `@iconify/react` dependency:

```bash
cd "<dir>" && pnpm add @iconify/react
```

#### Update Layout Shared Configuration

**`<dir>/src/lib/layout.shared.tsx`:**

Replace the implementation with icon-based links:

```typescript
import type { BaseLayoutProps } from 'fumadocs-ui/layouts/shared';
import { Icon } from '@iconify/react';
import { Signature } from 'lucide-react';
import { appName, blogRoute, docsRoute, gitConfig } from './shared';
import { FumadocsIcon } from '@/app/layout.client';

const githubUrl = `https://github.com/${gitConfig.user}`;

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
    links: [
      {
        type: 'icon',
        url: githubUrl,
        text: 'Github',
        label: 'GitHub',
        icon: <Icon icon="fa7-brands:github" />,
        external: true,
      },
      {
        type: 'icon',
        url: blogRoute,
        text: 'Blog',
        label: 'Blog',
        icon: <Signature />,
      },
    ],
  };
}

export function homeOptions(): BaseLayoutProps {
  return {
    ...baseOptions(),
    links: [
      {
        text: 'Docs',
        url: docsRoute,
      },
      {
        text: 'Blog',
        url: blogRoute,
      },
    ],
    githubUrl,
  };
}
```

**Key changes:**
1. Removed `githubUrl` from `baseOptions()` return value
2. Added `links` array with two `type: 'icon'` entries:
   - GitHub icon using `@iconify/react` with `fa7-brands:github`
   - Blog icon using `Signature` from `lucide-react`
3. `homeOptions()` overrides `links` with text links and adds `githubUrl` back

**How it works:**
- `DocsLayout` renders `type: 'icon'` links in the sidebar footer (docs pages only)
- `HomeLayout` renders regular text links in the navbar (home and blog pages)
- The docs sidebar shows GitHub and Blog as icon buttons side-by-side
- Home and blog pages show "Docs" and "Blog" as text links in the navbar with GitHub as a standalone button

**Icon customization:**
- To use a different lucide icon for blog, replace `Signature` import and JSX
- Icon size is automatically applied by fumadocs-ui (`size-4.5`)

## How The System Works

### Blog Architecture

1. **Content layer**: MDX files in `<dir>/content/blog/` with frontmatter
2. **Source layer**: `blogSource` loader transforms MDX into structured data
3. **Utility layer**: `blog.ts` provides helper functions for querying posts
4. **Component layer**: Reusable blog UI components
5. **Route layer**: Next.js App Router pages consume utilities and components

### Post Sorting and Navigation

- Posts are sorted by date (newest first) via `getPosts()`
- `findPost()` returns a post with `newer` and `older` neighbors for navigation
- Tags are aggregated with post counts via `getTags()`
- Tag pages filter posts with `getTagPosts()`

### TOC Integration

- `BlogPage` component wraps post content
- `toc` prop receives table of contents from page data
- Floating panel shows TOC and share button
- Panel uses `TOCProvider` context for item management
- Advanced TOC with animated SVG path tracking

### Author Resolution

- Author keys in frontmatter (e.g., `[fuma]`) are resolved against `blogAuthors`
- Unknown keys become plain name strings
- Resolved authors include profile data (name, title, URL, image)

## Expected Results

After completing this task:

### Blog Routes

- **`/blog`** - Main blog listing with banner and post cards
- **`/blog/[slug]`** - Individual post pages with TOC
- **`/blog/tags`** - All tags index with post counts
- **`/blog/tags/[tag]`** - Posts filtered by tag

### Features

- Post listings with cover images
- Individual post pages with metadata
- Tag filtering system with counts
- Author profiles with avatars
- Date formatting
- Post navigation (newer/older)
- Floating TOC panel with share button
- Banner on main blog page
- Responsive design
- SEO metadata generation
- Static site generation support

### Visual Design

- Dark banner with overlay on main blog page
- Grid layout for post cards (responsive)
- Post cards show cover images, title, description, date, and tags
- Clean typography with proper spacing
- Tag badges with hover effects
- Author avatars and profile links
- Floating TOC panel at bottom of posts
- Animated TOC with active section tracking

## Files Created

1. `<dir>/src/components/blog/` — Blog UI components (created by CLI)
2. `<dir>/src/components/blog/panel.tsx` — Floating TOC panel (created by CLI)
3. `<dir>/src/components/toc/` — TOC context and rendering (created by CLI)
4. `<dir>/src/lib/blog.ts` — Blog utilities and helpers (created by CLI)
5. `<dir>/src/app/blog/layout.tsx` — Blog layout wrapper
6. `<dir>/src/app/blog/page.tsx` — Main blog listing
7. `<dir>/src/app/blog/[slug]/page.tsx` — Individual post page
8. `<dir>/src/app/blog/tags/page.tsx` — Tags index
9. `<dir>/src/app/blog/tags/[tag]/page.tsx` — Tag filter page
10. `<dir>/src/app/blog/banner.png` — Banner image
11. `<dir>/content/blog/welcome.mdx` — Sample welcome post
12. `<dir>/content/blog/best-practices.mdx` — Sample tutorial post

## Files Modified

1. `<dir>/src/lib/source.ts` — Added blog collection, loader, and author definitions
2. `<dir>/next.config.mjs` — Added remote image patterns for avatars
3. `<dir>/src/lib/shared.ts` — Added blog route constant
4. `<dir>/src/app/global.css` — Added `@theme` block with `--spacing-page: 1436px` for proper page width, blog section color variables, and banner text colors
5. `<dir>/src/app/layout.client.tsx` — Updated section detection for blog routes
6. `<dir>/src/lib/layout.shared.tsx` — Added homeOptions() with navigation links
7. `<dir>/src/app/(home)/layout.tsx` — Updated to use homeOptions()
8. `<dir>/src/app/blog/layout.tsx` — Updated to use homeOptions()
9. `<dir>/src/app/blog/page.tsx` — Updated banner with color variables

## Customization Guide

**Note**: This section is for human readers to understand how to customize the implementation. AI agents should NOT attempt to implement these customizations unless explicitly asked by the user.

### Adding More Authors

Update `blogAuthors` in `<dir>/src/lib/source.ts`:

```typescript
export const blogAuthors: Record<string, BlogAuthor> = {
  fuma: {
    name: 'Fuma Nama',
    title: 'Maintainer',
    url: 'https://fuma-nama.dev',
    image: 'https://avatars.githubusercontent.com/u/76240755',
  },
  jane: {
    name: 'Jane Doe',
    title: 'Contributor',
    url: 'https://janedoe.dev',
    image: 'https://avatars.githubusercontent.com/u/123456',
  },
};
```

### Customizing Banner

Replace `banner.png` or modify the banner section in `page.tsx`:

```typescript
<div className="relative dark mb-4 aspect-[3.2] p-8 z-2 md:p-12">
  <Image
    src={BannerImage}
    priority
    alt="banner"
    className="absolute inset-0 size-full -z-1 object-cover"
  />
  <h1 className="mb-4 text-3xl text-landing-foreground font-mono font-medium">
    Your Custom Title
  </h1>
  <p className="text-sm font-mono text-landing-foreground-200">
    Your custom description
  </p>
</div>
```

### Customizing Post Card Layout

Modify `BlogCards` component in `<dir>/src/components/blog/index.tsx`:

- Change grid columns
- Adjust card styling
- Customize image aspect ratio
- Modify metadata display

### Customizing TOC Behavior

Modify `<dir>/src/components/blog/panel.tsx`:

- Change panel position (bottom/side)
- Adjust collapse behavior
- Customize share button behavior
- Modify floating panel styling

### Adding RSS Feed

Create an RSS route at `<dir>/src/app/blog/rss.xml/route.ts`:

```typescript
import { blogSource } from '@/lib/source';
import { getPosts } from '@/lib/blog';
import RSS from 'rss';

export async function GET() {
  const feed = new RSS({
    title: 'Your Blog',
    feed_url: 'https://yoursite.com/blog/rss.xml',
    site_url: 'https://yoursite.com',
  });

  const posts = getPosts(blogSource.getPages());

  posts.forEach((post) => {
    feed.item({
      title: post.title,
      description: post.description,
      url: `https://yoursite.com${post.url}`,
      date: post.date!,
    });
  });

  return new Response(feed.xml(), {
    headers: {
      'Content-Type': 'application/xml',
    },
  });
}
```

## Troubleshooting

### Posts Not Showing

- Verify frontmatter has required fields: `title`, `description`, `date`
- Check date format is YYYY-MM-DD
- Ensure `content/blog/` directory exists with `.mdx` files
- Verify blog collection is defined in `source.ts`

### Images Not Loading

- Check `next.config.mjs` has correct `remotePatterns` for remote images
- For local images, ensure they're in `public/` directory
- Verify image paths in frontmatter are correct
- Check image file extensions match references
- **Avatar images not loading**: If you see an error like "upstream image hostname resolved to private IP", add `dangerouslyAllowLocalIP: true` to the Next.js images configuration. This can happen when GitHub avatar URLs resolve to private IP addresses in certain network environments.

### TOC Not Appearing

- Verify post MDX has headings (`##`, `###`)
- Check `BlogPage` receives `toc={page.data.toc}` prop
- Ensure TOC components are imported from CLI-generated files

### Authors Not Resolving

- Verify author keys in frontmatter match keys in `blogAuthors`
- Check `blogAuthors` export in `source.ts`
- Unknown keys become plain names (expected behavior)

### Tags Not Working

- Tags are case-sensitive
- Tag slugs are auto-generated (lowercase, hyphenated)
- Verify `getTags()` and `getTagPosts()` imports

### Build Errors

- Run `pnpm build` to check for TypeScript errors
- Verify all imports are correct
- Check Next.js App Router conventions (async params)
- Ensure `getMDXComponents()` function exists

## Pattern Reference

### Official Fumadocs Approach

This implementation follows the official Fumadocs blog integration pattern:

- **CLI-first**: Uses `@fumadocs/cli add blog` to scaffold components
- **Source-based**: Leverages Fumadocs source API for content management
- **Component-driven**: Provides reusable blog UI components
- **Type-safe**: Full TypeScript support with schema validation
- **App Router**: Native Next.js App Router integration

### Key Design Decisions

1. **HomeLayout over DocsLayout**: Blog uses marketing layout, not docs layout
2. **Banner design**: Dark overlay on image for text visibility
3. **Tag slugification**: Automatic URL-safe slug generation
4. **Author resolution**: Keys resolve to profiles, unknowns become names
5. **TOC integration**: Floating panel with share button
6. **Static generation**: Full SSG support for all routes

### Why This Architecture

- **Separation of concerns**: Content (MDX) separate from presentation (components)
- **Reusable utilities**: Helper functions for common blog operations
- **Type safety**: Schema validation catches errors early
- **Performance**: Static generation for fast page loads
- **Extensibility**: Easy to add new features (RSS, pagination, search)

## Notes

- The CLI creates components you own — fumadocs-ui updates won't affect them
- Posts are sorted by date in descending order (newest first)
- Tags are case-sensitive but slugs are normalized
- Author avatars require Next.js image configuration for remote URLs
- Cover images can be local (`/blog/image.png`) or remote URLs
- The floating TOC panel uses native `navigator.share()` when available
- All blog routes support static site generation
- The blog is independent of the docs section — different layouts and styling
- Blog section colors are applied via the `.blog` body class using CSS custom properties
- Banner text colors use separate variables for light/dark mode support
- Navigation links appear as text in home/blog layouts
- Icon links appear in docs sidebar footer (requires `@iconify/react`)
