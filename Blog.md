**Caution: All the paths/directories mentioned in this file is for reference only because paths may change depending on different versions of fumadocs.**

## Path Convention

Throughout this guide, `<dir>` stands for the top-level directory of the Fumadocs app you are working in — the folder containing `package.json`, `src/`, and `content/`.

- If the app is at the repository root, `<dir>` is `.` and `<dir>/src/lib/source.ts` simply means `src/lib/source.ts`.
- If the app lives in a subfolder (e.g. `template/`, `apps/docs/`), substitute that name: `<dir>/src/lib/source.ts` → `template/src/lib/source.ts`.

Resolve `<dir>` once at the start, then apply it consistently to every path below.

**Note:** `<dir>` applies to filesystem paths only. Import specifiers beginning with `@/` (such as `@/lib/source`) are TypeScript path aliases resolved relative to the app's own `src/` directory — leave them exactly as written, with no `<dir>` prefix.

Separately, `<official>` stands for a local checkout of the official Fumadocs monorepo, used only as a source to copy reference assets from. It is a different location than `<dir>`.

# Task 6: Adding a Blog Alongside Docs

This task adds a blog to an existing Fumadocs app as a **second content collection**, independent of the docs tree. Complete Task 1 first — Step 5 depends on the section-color system being in place.

**Objective**: Serve MDX posts from `content/blog` at `/blog`, with an index page listing posts newest-first and per-post pages that inherit the app's section color.

## Required Dependencies

Install from inside `<dir>`:

```bash
cd "<dir>" && pnpm add zod
```

### Dependencies Explained:
- **zod**: Needed to extend the built-in `pageSchema` with the blog-specific `author` and `date` frontmatter fields. It is usually already present as a transitive dependency of `fumadocs-core`, but extending a schema in your own code requires it as a **direct** dependency — otherwise the import fails to resolve.

## Task 6 Steps

### Step 1: Add the Blog Route Constant

Keep route strings in one place so the loader and the navbar cannot drift apart.

**`<dir>/src/lib/shared.ts`:**
```ts
export const appName = 'Future Studio';
export const docsRoute = '/docs';
export const blogRoute = '/blog';
export const docsImageRoute = '/og/docs';
export const docsContentRoute = '/llms.mdx/docs';
```

### Step 2: Define the Blog Collection and Loader

A blog is a separate collection with its own loader and its own `baseUrl` — it is *not* a folder inside `content/docs`. Keeping it separate means blog posts never appear in the docs sidebar or page tree.

**`<dir>/src/lib/source.ts`:**
```typescript
import { loader } from 'fumadocs-core/source';
import { lucideIconsPlugin } from 'fumadocs-core/source/lucide-icons';
import { blogRoute, docsContentRoute, docsImageRoute, docsRoute } from './shared';
import { defineCollections, defineDocs } from 'fumadocs-mdx/macro';
import { metaSchema, pageSchema } from 'fumadocs-core/source/schema';
import { z } from 'zod';

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
  schema: pageSchema.extend({
    author: z.string(),
    date: z.iso.date().or(z.date()),
  }),
});

// See https://fumadocs.dev/docs/headless/source-api for more info
export const source = loader({
  baseUrl: docsRoute,
  source: docs.toFumadocsSource(),
  plugins: [lucideIconsPlugin()],
});

export const blogLoader = loader({
  baseUrl: blogRoute,
  source: blog.toFumadocsSource(),
});
```

**Key points:**
- `defineCollections` with `type: 'doc'` declares a doc-only collection (no `meta.json` handling, since a blog has no sidebar to order)
- `pageSchema.extend({ ... })` keeps the standard `title`/`description` fields and adds blog-specific ones
- `z.iso.date().or(z.date())` accepts both `2026-09-11` (parsed by the YAML loader as a string) and a real `Date`
- `baseUrl: blogRoute` is what makes `page.url` resolve to `/blog/<slug>`
- No `lucideIconsPlugin()` — posts do not use icons

**Note:** This collection is **synchronous**, matching the template's docs setup — `page.data.body` is the MDX component and `page.data.toc` is available directly. The official Fumadocs repo uses `async: true` on its blog collection, which instead requires `const { body: Mdx, toc } = await page.data.load()`. Do not mix the two styles.

### Step 3: Create Example Posts

Create `<dir>/content/blog/` and add posts. Each file needs all four frontmatter fields defined by the schema:

**`<dir>/content/blog/introducing-the-blog.mdx`:**
```mdx
---
title: Introducing the Blog
description: How this blog is wired up, and how to add your own posts.
author: Future Studio
date: 2026-09-11
---

## Where Posts Live

Posts live in `content/blog` as MDX files. The file name becomes the URL slug,
so `introducing-the-blog.mdx` is served at `/blog/introducing-the-blog`.
```

**`<dir>/content/blog/writing-your-second-post.mdx`:**
```mdx
---
title: Writing Your Second Post
description: A shorter example, mostly here to show how the index sorts posts.
author: Future Studio
date: 2026-08-20
---

## How Sorting Works

This post has an earlier `date`, so it appears second on the index.
```

**Frontmatter reference:**

| Field | Required | Notes |
|-------|----------|-------|
| `title` | yes | Post heading, also the browser tab title |
| `description` | no | Shown on the index card |
| `author` | yes | Displayed above the title |
| `date` | yes | Controls sort order on the index |

Sort order comes from `date`, not the file name, so files need no numeric prefixes.

### Step 4: Copy the Banner Image

The index page uses a full-width banner behind its heading.

```bash
cp "<official>/apps/docs/app/(home)/blog/banner.png" "<dir>/src/app/(home)/blog/banner.png"
```

If you do not have the official repo checked out locally, substitute any wide image — the container is `aspect-[3.2]`, so roughly 3.2:1 works best.

### Step 5: Add Banner Text and Blog Colors

This is the only step that touches `global.css`, so it makes all three additions at once — the banner's own foreground colors, plus the blog's section color and the `.blog` rule that Step 9 activates.

The banner renders light text over a dark image, so it needs foreground colors that do not depend on the active theme's text color. The blog color works exactly like `--sec-1-color` and `--sec-2-color` from Task 1: a variable pair plus a class that maps it to `--color-fd-primary`.

**`<dir>/src/app/global.css`:**
```css
@import 'tailwindcss';
@import 'fumadocs-ui/css/neutral.css';
@import 'fumadocs-ui/css/preset.css';

:root {
  --sec-1-color: hsl(26, 73%, 51%);
  --sec-2-color: hsl(217, 100%, 58%);
  --banner-text-color-up: #59592a;
  --banner-text-color-down: #a8a866;
  --blog-color: hsl(25, 55%, 35%);
}

.dark {
  --sec-1-color: #fff383;
  --sec-2-color: #a9ceff;
  --banner-text-color-up: #e4e2d0;
  --banner-text-color-down: #b7af7e;
  --blog-color: #d9b38c;
}

.sec-1 {
  --color-fd-primary: var(--sec-1-color);
}

.sec-2 {
  --color-fd-primary: var(--sec-2-color);
}

.blog {
  --color-fd-primary: var(--blog-color);
}

html {
  scrollbar-gutter: stable;
}

html > body[data-scroll-locked] {
  margin-right: 0px !important;
  --removed-body-scroll-bar-size: 0px !important;
}
```

**Why `:root` and not `@theme`?** Tailwind v4 only generates named utility classes for colors declared inside `@theme`, and only for the `--color-*` namespace — a variable named `--banner-text-color-up` would produce no class even there. Plain custom properties in `:root` sidestep the namespace entirely: the arbitrary-value syntax `text-(--banner-text-color-up)` compiles straight to `color: var(--banner-text-color-up)`, referencing whatever value is live. The `.dark` block then overrides the values at runtime.

Naming the pair `up` and `down` reflects their position in the banner — `up` is the heading, `down` is the subtitle below it.

**The `.blog` rule does nothing on its own.** It stays inert until Step 9 applies the `blog` class to an element — the two halves must both be in place. Since `.blog` is hand-written CSS rather than a Tailwind utility, it is emitted verbatim and needs no safelisting.

### Step 6: Create the Blog Index Page

Placing blog routes inside the `(home)` route group means they inherit the navbar from `<dir>/src/app/(home)/layout.tsx` without the docs sidebar.

**`<dir>/src/app/(home)/blog/page.tsx`:**
```typescript
import Link from 'next/link';
import Image from 'next/image';
import { blogLoader } from '@/lib/source';
import { appName } from '@/lib/shared';
import BannerImage from './banner.png';
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Blog',
  description: `Latest announcements of ${appName}.`,
};

export default function Page() {
  const posts = [...blogLoader.getPages()].sort(
    (a, b) => new Date(b.data.date).getTime() - new Date(a.data.date).getTime()
  );

  return (
    <main className="mx-auto w-full max-w-[1400px] px-4 pb-12 md:py-12">
      <div className="dark relative z-2 mb-4 aspect-[3.2] p-8 md:p-12">
        <Image
          src={BannerImage}
          priority
          alt="banner"
          className="absolute inset-0 -z-1 size-full object-cover"
        />
        <h1 className="mb-4 font-mono text-3xl font-medium text-(--banner-text-color-up)">
          {appName} Blog
        </h1>
        <p className="font-mono text-sm text-(--banner-text-color-down)">
          Latest announcements of {appName}.
        </p>
      </div>
      <div className="grid grid-cols-1 gap-2 md:grid-cols-3 xl:grid-cols-4">
        {posts.map((post) => (
          <Link
            key={post.url}
            href={post.url}
            className="flex flex-col rounded-2xl border bg-fd-card p-4 shadow-sm transition-colors hover:bg-fd-accent hover:text-fd-accent-foreground"
          >
            <p className="font-medium">{post.data.title}</p>
            <p className="text-sm text-fd-muted-foreground">{post.data.description}</p>
            <p className="mt-auto pt-4 text-xs" style={{ color: 'var(--color-fd-primary)' }}>
              {new Date(post.data.date).toDateString()}
            </p>
          </Link>
        ))}
      </div>
    </main>
  );
}
```

**Notes on the layout:**
- `[...blogLoader.getPages()]` copies before sorting — `getPages()` returns the loader's own array, which should not be mutated in place
- The `dark` class on the banner container forces dark-mode tokens for its subtree, so text stays readable over the image in both themes
- `-z-1` on the image with `z-2` on the container puts the artwork behind the text
- `mt-auto` on the date pins it to the card bottom, so cards in a row align regardless of description length
- Grid columns are `1 / 3 / 4` across mobile / `md` / `xl` with `gap-2`, matching the official compact layout

### Step 7: Create the Share Button

A small client component — clipboard access and the copied-state toggle both require the browser.

**`<dir>/src/app/(home)/blog/[slug]/page.client.tsx`:**
```typescript
'use client';

import { Check, Share } from 'lucide-react';
import { useCopyButton } from 'fumadocs-ui/utils/use-copy-button';

export function ShareButton({ url }: { url: string }) {
  const [isChecked, onCopy] = useCopyButton(() => {
    void navigator.clipboard.writeText(`${window.location.origin}${url}`);
  });

  return (
    <button
      type="button"
      className="inline-flex items-center gap-2 rounded-full px-4 py-2 text-sm font-medium text-fd-primary-foreground transition-opacity hover:opacity-90"
      style={{ backgroundColor: 'var(--color-fd-primary)' }}
      onClick={onCopy}
    >
      {isChecked ? <Check className="size-4" /> : <Share className="size-4" />}
      {isChecked ? 'Copied URL' : 'Share Post'}
    </button>
  );
}
```

**What it does:**
- `useCopyButton` from Fumadocs handles the "Copied" state and its automatic reset, so no local `useState` or timer is needed
- `page.url` is a root-relative path, so `window.location.origin` is prepended to copy an absolute URL
- `backgroundColor: var(--color-fd-primary)` makes the button follow the active section color

### Step 8: Create the Post Page

**`<dir>/src/app/(home)/blog/[slug]/page.tsx`:**
```typescript
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';
import Link from 'next/link';
import { Undo2 } from 'lucide-react';
import { InlineTOC } from 'fumadocs-ui/components/inline-toc';
import { blogLoader } from '@/lib/source';
import { getMDXComponents } from '@/components/mdx';
import { ShareButton } from './page.client';

export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const params = await props.params;
  const page = blogLoader.getPage([params.slug]);
  if (!page) notFound();

  const MDX = page.data.body;

  return (
    <article className="mx-auto flex w-full max-w-[800px] flex-col px-4 py-8">
      <div className="mb-8 flex flex-row gap-6 text-sm">
        <div>
          <p className="mb-1 text-fd-muted-foreground">Written by</p>
          <p className="font-medium">{page.data.author}</p>
        </div>
        <div>
          <p className="mb-1 text-fd-muted-foreground">At</p>
          <p className="font-medium">{new Date(page.data.date).toDateString()}</p>
        </div>
      </div>

      <h1 className="mb-4 text-3xl font-semibold">{page.data.title}</h1>
      <p className="mb-8 text-fd-muted-foreground">{page.data.description}</p>

      <div className="prose min-w-0 flex-1">
        <div className="not-prose mb-8 flex flex-row gap-2">
          <ShareButton url={page.url} />
          <Link
            href="/blog"
            className="inline-flex items-center gap-2 rounded-full border bg-fd-secondary px-4 py-2 text-sm font-medium text-fd-secondary-foreground transition-colors hover:bg-fd-accent"
          >
            <Undo2 className="size-4" />
            Back
          </Link>
        </div>

        <InlineTOC items={page.data.toc} />
        <MDX components={getMDXComponents()} />
      </div>
    </article>
  );
}

export function generateStaticParams(): { slug: string }[] {
  return blogLoader.getPages().map((page) => ({
    slug: page.slugs[0],
  }));
}

export async function generateMetadata(props: PageProps<'/blog/[slug]'>): Promise<Metadata> {
  const params = await props.params;
  const page = blogLoader.getPage([params.slug]);
  if (!page) notFound();

  return {
    title: page.data.title,
    description: page.data.description,
  };
}
```

**Key points:**
- `getPage()` takes an **array** of slugs, so a single `[slug]` param is wrapped: `getPage([params.slug])`
- `prose` comes from the typography plugin that `fumadocs-ui/css/preset.css` loads via `@plugin`, so it is available without installing `@tailwindcss/typography`
- `not-prose` on the button row stops typography styles from restyling the buttons
- `generateStaticParams` pre-renders every post at build time
- `generateMetadata` gives each post its own tab title — with `metadata.title` in the root layout being a plain string rather than a template, no site-name suffix is appended

### Step 9: Fix Section Detection for Non-Docs Routes

The `Body` component from Task 2 must be corrected, or blog posts render with the wrong section color.

**`<dir>/src/app/layout.client.tsx`:**
```typescript
'use client';

import { useParams, usePathname } from 'next/navigation';
import { type ReactNode, useId } from 'react';
import { getSection } from '@/lib/navigation';
import { blogRoute } from '@/lib/shared';

export function Body({ children }: { children: ReactNode }) {
  const pathname = usePathname();
  const { slug } = useParams();
  const section = pathname.startsWith(blogRoute)
    ? 'blog'
    : getSection(Array.isArray(slug) ? slug[0] : slug);

  return <body className={`flex flex-col min-h-screen ${section}`}>{children}</body>;
}
```

**Why this change is required:**

`useParams()` returns a different shape depending on the route's segment type, and it never exposes the route prefix:

| Route | Segment | `slug` value |
|-------|---------|--------------|
| `/docs/sec-2/test` | `[[...slug]]` — catch-all | `['sec-2', 'test']` — an array |
| `/blog/my-post` | `[slug]` — single dynamic | `'my-post'` — a **string** |

Two separate bugs come out of this:

1. **The original `Array.isArray(slug) ? getSection(slug[0]) : undefined`** left `section` as `undefined` on a blog post, so the template literal emitted an empty class, no `.sec-*` rule matched, and `--color-fd-primary` kept its unthemed default — the navbar gradient icon rendered black-to-white.

2. **Normalizing the shape alone is still not enough.** `getSection('my-post')` finds no match in its lookup table and falls back to `'sec-1'`, so blog routes silently inherit Section 1's color. Because `useParams()` cannot see the `/blog` prefix, distinguishing the blog requires `usePathname()`.

Checking `pathname.startsWith(blogRoute)` first resolves both. Docs routes still fall through to `getSection`, which keeps its `'sec-1'` default for `/docs` itself.

**This is the half that activates `.blog`.** The rule added in Step 5 is inert until this class lands on the `<body>` — neither half does anything alone. To make the blog inherit Section 1's color instead, drop the `pathname.startsWith` branch here and remove the `.blog` rule and its two `--blog-color` declarations from Step 5.

### Step 10: Add Navbar Links

Add both links so the blog and docs are reachable from the home and blog pages, with Docs first.

`baseOptions()` is shared by both the home and docs layouts, so putting `links` there is a mistake: `DocsLayout` passes `links` through `useLinkItems`, and its sidebar slot renders the result as `menuItems`. The links would appear **in the docs sidebar**, not just the navbar. Keep `baseOptions()` free of `links` and add a second function for the home layout.

**`<dir>/src/lib/layout.shared.tsx`:**
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

Then point the home layout at the new function. The docs layout keeps calling `baseOptions()` and needs no change.

**`<dir>/src/app/(home)/layout.tsx`:**
```typescript
import { HomeLayout } from 'fumadocs-ui/layouts/home';
import { homeOptions } from '@/lib/layout.shared';

export default function Layout({ children }: LayoutProps<'/'>) {
  return <HomeLayout {...homeOptions()}>{children}</HomeLayout>;
}
```

`links` array order is render order. Because blog routes live in the `(home)` route group, they inherit this navbar too — so Docs and Blog are reachable from every page except the docs pages, which have their own section tabs for navigation.

---

### Step 11 (Optional): Add a Blog Icon Button to the Docs Sidebar

**Note**: This is an optional step. If you are an AI, you must ask the user whether this should be executed, meaning AI should ask and obtain clear permission to execute this step.

After Step 10 the docs pages have no route to the blog — the sidebar's bottom-left corner holds only the GitHub button. This step adds a second icon button beside it, to GitHub's right.

`DocsLayout` splits `links` by type: entries with `type: 'icon'` go to the sidebar's **footer row** next to GitHub, while every other type is rendered in the sidebar **body** above the page tree. So the blog link must be `type: 'icon'` — a plain text link would reintroduce the sidebar clutter Step 10 removed.

Ordering is the subtle part. `resolveLinkItems` **appends** the `githubUrl` shortcut after everything in `links`, so any icon declared in `links` lands to GitHub's *left*. To put the blog icon on the right, drop `githubUrl` from `baseOptions()` and declare GitHub as an explicit `type: 'icon'` entry first. lucide-react ships no brand icons, so the GitHub mark is inlined as an SVG.

**`<dir>/src/lib/layout.shared.tsx`:**
```typescript
import type { BaseLayoutProps } from 'fumadocs-ui/layouts/shared';
import { Signature } from 'lucide-react';
import { appName, blogRoute, docsRoute, gitConfig } from './shared';
import { FumadocsIcon } from '@/app/layout.client';

const githubUrl = `https://github.com/${gitConfig.user}`;

// `githubUrl` is appended after `links`, so GitHub is declared explicitly to control icon order.
const GithubIcon = (
  <svg role="img" viewBox="0 0 24 24">
    <path
      fill="currentColor"
      fillRule="evenodd"
      clipRule="evenodd"
      d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385c.6.105.825-.255.825-.57c0-.285-.015-1.23-.015-2.235c-3.015.555-3.795-.735-4.035-1.41c-.135-.345-.72-1.41-1.23-1.695c-.42-.225-1.02-.78-.015-.795c.945-.015 1.62.87 1.845 1.23c1.08 1.815 2.805 1.305 3.495.99c.105-.78.42-1.305.765-1.605c-2.67-.3-5.46-1.335-5.46-5.925c0-1.305.465-2.385 1.23-3.225c-.12-.3-.54-1.53.12-3.18c0 0 1.005-.315 3.3 1.23c.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23c.66 1.65.24 2.88.12 3.18c.765.84 1.23 1.905 1.23 3.225c0 4.605-2.805 5.625-5.475 5.925c.435.375.81 1.095.81 2.22c0 1.605-.015 2.895-.015 3.3c0 .315.225.69.825.57A12.02 12.02 0 0 0 24 12c0-6.63-5.37-12-12-12"
    />
  </svg>
);

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
        icon: GithubIcon,
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

`homeOptions()` **overrides** `links` rather than extending it, so the home and blog navbars keep the plain Docs and Blog text links from Step 10 and get GitHub back through `githubUrl`. Only the docs sidebar sees the two icon entries.

No size class is needed on `<Signature />`: the sidebar renders icon items with `buttonVariants({ size: 'icon-sm' })`, which is `p-1.5 [&_svg]:size-4.5`.

To use a different lucide icon, swap the import and the JSX — the rest of the entry is unchanged.

---

## How The System Works

### Two Independent Collections

```
content/docs/          →  source      →  baseUrl '/docs'   →  sidebar + tabs + page tree
content/blog/          →  blogLoader  →  baseUrl '/blog'   →  flat list, sorted by date
```

Each collection has its own schema and its own loader. Nothing is shared but the MDX component set, so a post can use any component a doc page can.

### Request Flow for a Post

1. **User visits** `/blog/introducing-the-blog`
2. **Root layout** renders `Body`; `usePathname()` returns `/blog/introducing-the-blog`, which starts with `/blog`, so `section` is `'blog'` → `<body class="... blog">`
3. **CSS activates**: `.blog { --color-fd-primary: var(--blog-color) }`, so the navbar icon, the date text, and the Share Post button all pick up the blog color
4. **`(home)` layout** wraps the page with the navbar via `homeOptions()`, including the Docs and Blog links
5. **Post page** calls `blogLoader.getPage(['introducing-the-blog'])`, renders `page.data.body`, and `notFound()`s on an unknown slug

## Expected Results

- **`/blog`**: banner with "Future Studio Blog" over the background image, then post cards — 1 column on mobile, 3 at `md`, 4 at `xl`
- **Sorting**: newest `date` first, independent of file name
- **`/blog/<slug>`**: author and date header, title, description, Share Post + Back buttons, inline table of contents, then the post body
- **Share Post**: copies the absolute post URL and swaps to "Copied URL" briefly
- **Blog color**: navbar icon, card dates, and the Share button all render in the blog color (brown in light mode, tan in dark) on every `/blog` route, while docs routes keep their own section colors
- **Navbar**: Docs and Blog links, in that order, on the home and blog pages — the docs pages navigate by section tabs instead, and their sidebar shows no `links` entries
- **Docs sidebar** (Step 11 only): a GitHub button and, to its right, a `Signature` button linking to `/blog`

## Files Created

1. `<dir>/content/blog/introducing-the-blog.mdx` — example post
2. `<dir>/content/blog/writing-your-second-post.mdx` — second example, earlier date
3. `<dir>/src/app/(home)/blog/banner.png` — index banner artwork
4. `<dir>/src/app/(home)/blog/page.tsx` — index page
5. `<dir>/src/app/(home)/blog/[slug]/page.tsx` — post page
6. `<dir>/src/app/(home)/blog/[slug]/page.client.tsx` — `ShareButton`

## Files Modified

1. `<dir>/src/lib/shared.ts` — added `blogRoute`
2. `<dir>/src/lib/source.ts` — blog collection and `blogLoader`
3. `<dir>/src/lib/layout.shared.tsx` — `homeOptions()` added with the Docs and Blog navbar links; Step 11 adds the sidebar icon entries
4. `<dir>/src/app/(home)/layout.tsx` — switched to `homeOptions()`
5. `<dir>/src/app/global.css` — `--banner-text-color-up` / `--banner-text-color-down`, plus the `--blog-color` variables and `.blog` rule
6. `<dir>/src/app/layout.client.tsx` — section detection fixed for single dynamic segments, and the `blog` class applied on `/blog` routes
7. `<dir>/package.json` — `zod` promoted to a direct dependency

---

## Customization Guide

**Note**: This section is for human readers to understand how to customize the implementation. AI agents should NOT attempt to implement these customizations unless explicitly asked by the user.

### Changing or Removing the Blog Color

The blog's brown is set up across Steps 5 and 9. To change it, edit `--blog-color` in both the `:root` and `.dark` blocks of `<dir>/src/app/global.css`.

To make the blog inherit Section 1's color instead, remove the `.blog` rule and the two `--blog-color` declarations, then drop the pathname branch from `Body`:

```typescript
export function Body({ children }: { children: ReactNode }) {
  const { slug } = useParams();
  const section = getSection(Array.isArray(slug) ? slug[0] : slug);

  return <body className={`flex flex-col min-h-screen ${section}`}>{children}</body>;
}
```

`getSection` returns `'sec-1'` for any unrecognized path, so blog routes fall back to it automatically.

### Changing Grid Density

The index grid is `grid-cols-1 gap-2 md:grid-cols-3 xl:grid-cols-4`. For roomier cards use `md:grid-cols-2 xl:grid-cols-3` with `gap-4`; for denser, add `2xl:grid-cols-5`.

### Adding an RSS Feed

The official docs app serves a feed from `app/(home)/blog/rss.xml/route.ts`, building the XML from `blogLoader.getPages()`. See `<official>/apps/docs/app/(home)/blog/rss.xml/route.ts` for a working implementation.

### Adding Open Graph Images

Docs pages get OG images via `<dir>/src/app/og/docs/[...slug]/route.tsx`. A parallel route under `og/blog` plus an `openGraph.images` entry in the post's `generateMetadata` would extend that to posts.

### Additional Frontmatter Fields

Extend the schema and the field is typed everywhere `page.data` is used:

```typescript
schema: pageSchema.extend({
  author: z.string(),
  date: z.iso.date().or(z.date()),
  tags: z.array(z.string()).optional(),
}),
```

---

## Pattern Reference

### Official Fumadocs Implementation

This implementation mirrors the official Fumadocs docs app:
- The blog is a separate `defineCollections` call with its own `loader`, exported as `blogLoader`
- Routes live under the `(home)` route group to inherit the navbar without the docs sidebar
- The index sorts descending by date and renders a banner over a background image
- `ShareButton` is an isolated client component using `useCopyButton`

Two deliberate differences:
- **Sync instead of async collection** — matches the template's existing docs setup, so `page.data.body` is used directly rather than `await page.data.load()`
- **No `buttonVariants`** — the template has no `components/ui/button`, so buttons use inline utility classes and follow `--color-fd-primary` directly

### Key Insight

A blog is not a special Fumadocs feature — it is just a second content collection pointed at a different directory with a different `baseUrl`. Anything the docs loader can do, a blog loader can do.

The one cross-cutting gotcha is route-shape coupling: any code reading `useParams()` globally, such as the section-color `Body`, must handle both catch-all arrays and single dynamic strings. Adding a route type the app did not previously have is exactly when that assumption breaks.

A shared config object hides a similar trap. `baseOptions()` looks like navbar-only configuration, but each layout interprets its fields differently — `DocsLayout` renders `links` in its sidebar, while `HomeLayout` renders them in the navbar. Fields that should only affect one layout belong in a layout-specific wrapper like `homeOptions()`, not in the shared base.
