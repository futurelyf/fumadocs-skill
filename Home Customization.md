**Caution: All the paths/directories mentioned in this file is for reference only because paths may change depending on different versions of fumadocs.**

## Path Convention

Throughout this guide, `<dir>` stands for the top-level directory of the Fumadocs app you are working in — the folder containing `package.json`, `src/`, and `content/`.

- If the app is at the repository root, `<dir>` is `.` and `<dir>/src/app/layout.tsx` simply means `src/app/layout.tsx`.
- If the app lives in a subfolder (e.g. `template/`, `apps/docs/`), substitute that name: `<dir>/src/app/layout.tsx` → `template/src/app/layout.tsx`.

Resolve `<dir>` once at the start, then apply it consistently to every path below.

**Note:** `<dir>` applies to paths inside the app you are modifying. It does NOT apply to import specifiers beginning with `@/` (such as `@/lib/cn`) — those are TypeScript path aliases resolved relative to the app's own `src/` directory, so leave them exactly as written. It also does not apply to `<official>` (see below).

Separately, `<official>` stands for a local checkout of the official Fumadocs monorepo, used only as a source to copy reference assets from. It is a different location than `<dir>`.

# Task 4: Hero Section with Animated Shaders

This task transplants the official Fumadocs homepage hero section design to your app, including animated gradient backgrounds and the screenshot preview.

## Required Dependencies

Install the following packages using pnpm, from inside `<dir>`:

```bash
cd "<dir>" && pnpm add @paper-design/shaders-react next-themes
```

### Dependencies Explained:
- **@paper-design/shaders-react**: Provides WebGL-based shader components (GrainGradient, Dithering) for animated visual effects
- **next-themes**: Handles theme detection (light/dark mode) to adapt shader colors dynamically

## Files to Create/Modify

### 1. Create `<dir>/src/app/(home)/page.client.tsx`

This file contains the Hero component with shader effects and intersection observer logic.

```tsx
'use client';

import Image from 'next/image';
import { useTheme } from 'next-themes';
import { useEffect, useRef, useState, type RefObject } from 'react';
import { cn } from '@/lib/cn';
import HeroImage from './hero-preview.jpeg';
import dynamic from 'next/dynamic';

const GrainGradient = dynamic(
  () => import('@paper-design/shaders-react').then((mod) => mod.GrainGradient),
  {
    ssr: false,
  },
);

const Dithering = dynamic(
  () => import('@paper-design/shaders-react').then((mod) => mod.Dithering),
  {
    ssr: false,
  },
);

export function Hero() {
  const { resolvedTheme } = useTheme();
  const ref = useRef<HTMLImageElement | null>(null);
  const visible = useIsVisible(ref);
  const [showGradient, setShowGradient] = useState(false);
  const [showDithering, setShowDithering] = useState(false);
  const [imageReady, setImageReady] = useState(false);

  useEffect(() => {
    // Stagger the shader appearances for gradual effect
    const gradientTimer = setTimeout(() => {
      setShowGradient(true);
    }, 200);

    const ditheringTimer = setTimeout(() => {
      setShowDithering(true);
    }, 600);

    return () => {
      clearTimeout(gradientTimer);
      clearTimeout(ditheringTimer);
    };
  }, []);

  return (
    <>
      {showGradient && (
        <GrainGradient
          className="absolute inset-0 animate-fd-fade-in duration-800"
          colors={
            resolvedTheme === 'dark'
              ? ['#39BE1C', '#9c2f05', '#7A2A0000']
              : ['#fcfc51', '#ffa057', '#7A2A0020']
          }
          colorBack="#00000000"
          softness={1}
          intensity={0.9}
          noise={0.5}
          speed={visible ? 1 : 0}
          shape="corners"
          minPixelRatio={1}
          maxPixelCount={1920 * 1080}
        />
      )}
      {showDithering && (
        <Dithering
          width={720}
          height={720}
          colorBack="#00000000"
          colorFront={resolvedTheme === 'dark' ? '#DF3F00' : '#fa8023'}
          shape="sphere"
          type="4x4"
          scale={0.5}
          size={3}
          speed={0}
          frame={5000 * 120}
          className="absolute animate-fd-fade-in duration-400 max-lg:bottom-[-50%] max-lg:left-[-200px] lg:top-[-5%] lg:right-0"
          minPixelRatio={1}
        />
      )}
      <Image
        ref={ref}
        src={HeroImage}
        alt="hero-image"
        className={cn(
          'absolute top-[460px] left-[20%] max-w-[1200px] rounded-xl border-2 lg:top-[400px]',
          imageReady ? 'animate-fd-fade-in duration-400' : 'invisible',
        )}
        onLoad={() => setImageReady(true)}
        priority
      />
    </>
  );
}

let observer: IntersectionObserver;
const observerTargets = new WeakMap<Element, (entry: IntersectionObserverEntry) => void>();

function useIsVisible(ref: RefObject<HTMLElement | null>) {
  const [visible, setVisible] = useState(false);

  useEffect(() => {
    observer ??= new IntersectionObserver((entries) => {
      for (const entry of entries) {
        observerTargets.get(entry.target)?.(entry);
      }
    });

    const element = ref.current;
    if (!element) return;
    observerTargets.set(element, (entry) => {
      setVisible(entry.isIntersecting);
    });
    observer.observe(element);

    return () => {
      observer.unobserve(element);
      observerTargets.delete(element);
    };
  }, [ref]);

  return visible;
}
```

### 2. Update `<dir>/src/app/(home)/page.tsx`

Replace the simple "Hello World" page with the hero section design:

```tsx
import Link from 'next/link';
import { cn } from '@/lib/cn';
import { Hero } from './page.client';

export default function HomePage() {
  return (
    <div className="pt-4 pb-6 md:pb-12">
      <div className="relative flex min-h-[600px] h-[70vh] max-h-[900px] border rounded-2xl overflow-hidden mx-auto w-full max-w-[1400px] bg-origin-border">
        <Hero />
        <div className="flex flex-col z-2 px-4 size-full md:p-12 max-md:items-center max-md:text-center">
          <p className="mt-12 text-xs font-medium rounded-full p-2 border w-fit" style={{ color: 'var(--color-fd-primary)', borderColor: 'color-mix(in srgb, var(--color-fd-primary) 50%, transparent)' }}>
            the React.js docs framework you love.
          </p>
          <h1 className="text-4xl my-8 leading-tighter font-medium xl:text-5xl xl:mb-12">
            Build excellent
            <br className="md:hidden" /> documentation,
            <br />
            your <span style={{ color: 'var(--color-fd-primary)' }}>style</span>.
          </h1>
          <div className="flex flex-row items-center justify-center gap-4 flex-wrap w-fit">
            <Link
              href="/docs"
              className={cn(
                'inline-flex justify-center px-5 py-3 rounded-full font-medium tracking-tight transition-colors max-sm:text-sm',
                'text-fd-primary-foreground hover:opacity-90'
              )}
              style={{ backgroundColor: 'var(--color-fd-primary)' }}
            >
              Getting Started
            </Link>
            <a
              href="https://github.com/fuma-nama/fumadocs"
              target="_blank"
              rel="noreferrer noopener"
              className={cn(
                'inline-flex justify-center px-5 py-3 rounded-full font-medium tracking-tight transition-colors max-sm:text-sm',
                'border bg-fd-secondary text-fd-secondary-foreground hover:bg-fd-accent'
              )}
            >
              Open GitHub
            </a>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### 3. Copy Screenshot Image

Copy the hero preview image from a local checkout of the official repo (`<official>`) into your app (`<dir>`):

```bash
cp "<official>/apps/docs/app/(home)/hero-preview.jpeg" "<dir>/src/app/(home)/hero-preview.jpeg"
```

If you do not have the official repo checked out locally, substitute any image with a similar aspect ratio instead.

## Implementation Details

### Shader Effects

1. **GrainGradient**: 
   - Creates an animated gradient background that adapts to theme
   - Light mode: Yellow-orange gradient (#fcfc51, #ffa057)
   - Dark mode: Green-orange-red gradient (#39BE1C, #9c2f05)
   - Speed controlled by intersection observer (animates only when visible)
   - Appears at 200ms with 800ms fade-in duration

2. **Dithering**:
   - Sphere-shaped dithering effect positioned in top-right corner
   - Light mode: Orange (#fa8023)
   - Dark mode: Red-orange (#DF3F00)
   - Responsive positioning (bottom-left on mobile, top-right on desktop)
   - Appears at 600ms with 400ms fade-in duration

**Gradual Appearance**: The background elements appear in staggered sequence (gradient → dithering → screenshot) creating a natural, layered reveal effect matching the official design.

### Performance Optimizations

- **Dynamic imports**: Shaders loaded only on client-side
- **Intersection Observer**: Animation speed set to 0 when hero not visible
- **Staggered shader initialization**: GrainGradient appears at 200ms, Dithering at 600ms for gradual, natural reveal effect
- **Image loading**: Screenshot fades in only after fully loaded

### Layout Structure

- Full-height hero container (600px min, 70vh default, 900px max)
- Responsive text positioning (centered on mobile, left-aligned on desktop)
- Rounded corners with border
- Z-index layering: shaders → screenshot → text content

## Testing

After implementation:

1. Start dev server: `pnpm dev` (from inside `<dir>`)
2. Navigate to homepage
3. Verify animated gradient background appears
4. Verify dithering sphere effect in top-right
5. Verify screenshot appears in bottom-right
6. Test theme switching (light/dark mode)
7. Test responsive behavior on mobile viewport

## Notes

- Uses `next-themes` for theme detection, ensure RootProvider is set up in layout
- Shaders are WebGL-based and may not work on very old browsers
- The `useIsVisible` hook optimizes performance by pausing animations when off-screen
- Screenshot image can be replaced with your own (maintain similar aspect ratio)

---

# Task 5: Customization Updates

This task customizes the branding and links in your app to match your personal/company identity.

## Changes Required

### 1. Change Navbar Title

**File**: `<dir>/src/lib/shared.ts`

Change the app name from "My App" to "Future Studio":

```ts
export const appName = 'Future Studio';
```

### 2. Update GitHub Links

**Files**: 
- `<dir>/src/lib/shared.ts` - Update the user config
- `<dir>/src/lib/layout.shared.tsx` - Remove repo from URL

First, update the GitHub user in `shared.ts`:

```ts
export const gitConfig = {
  user: 'futurelyf',
  repo: 'fumadocs',
  branch: 'main',
};
```

Then, modify the GitHub URL in `layout.shared.tsx` to point to your profile (not a specific repo):

```tsx
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
```

This will update the GitHub link in the navbar to `https://github.com/futurelyf` (profile page, not a specific repository).

### 3. Update Hero Section Text & Adjust Heading Line Height

**File**: `<dir>/src/app/(home)/page.tsx`

Change the tagline from "the React.js docs framework you love." to "The docs template you love!" and adjust the heading line height:

```tsx
<p className="mt-12 text-xs font-medium rounded-full p-2 border w-fit" style={{ color: 'var(--color-fd-primary)', borderColor: 'color-mix(in srgb, var(--color-fd-primary) 50%, transparent)' }}>
  The docs template you love!
</p>
<h1 className="text-4xl my-8 leading-tight font-medium xl:text-5xl xl:mb-12">
  Build excellent
  <br className="md:hidden" /> documentation,
  <br />
  your <span style={{ color: 'var(--color-fd-primary)' }}>style</span>.
</h1>
```

### 4. Update "Getting Started" Button

**File**: `<dir>/src/app/(home)/page.tsx`

Change button text from "Getting Started" to "Get Started":

```tsx
<Link
  href="/docs"
  className={cn(
    'inline-flex justify-center px-5 py-3 rounded-full font-medium tracking-tight transition-colors max-sm:text-sm',
    'text-fd-primary-foreground hover:opacity-90'
  )}
  style={{ backgroundColor: 'var(--color-fd-primary)' }}
>
  Get Started
</Link>
```

### 5. Change Second Button to Contact

**File**: `<dir>/src/app/(home)/page.tsx`

Change button text from "Open GitHub" to "Contact" and update the link to your website:

```tsx
<a
  href="https://futurestudio.dev"
  target="_blank"
  rel="noreferrer noopener"
  className={cn(
    'inline-flex justify-center px-5 py-3 rounded-full font-medium tracking-tight transition-colors max-sm:text-sm',
    'border bg-fd-secondary text-fd-secondary-foreground hover:bg-fd-accent'
  )}
>
  Contact
</a>
```

### 6. Set Browser Tab Title

**File**: `<dir>/src/app/layout.tsx`

Add metadata export to set the browser tab title:

```tsx
import { RootProvider } from 'fumadocs-ui/provider/next';
import './global.css';
import { Inter } from 'next/font/google';
import { Body } from './layout.client';
import type { Metadata } from 'next';

const inter = Inter({
  subsets: ['latin'],
});

export const metadata: Metadata = {
  title: 'Future Studio',
};

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

**What it does:**
- Sets the homepage browser tab title to "Future Studio"
- Leaves other pages' titles untouched — docs pages set their own title via `generateMetadata`, so they display just the page name with no suffix appended
- Follows Next.js App Router metadata convention

**Why a plain string instead of `title.template`?** A `template` such as `'%s | Future Studio'` appends the site name to every other page's tab title. Use a plain string when you want only the home page branded.

## Summary of Changes

| Item | From | To |
|------|------|-----|
| Navbar Title | "My App" | "Future Studio" |
| GitHub User | `fuma-nama` | `futurelyf` |
| Hero Tagline | "the React.js docs framework you love." | "The docs template you love!" |
| Primary Button | "Getting Started" | "Get Started" |
| Secondary Button | "Open GitHub" → `github.com/fuma-nama/fumadocs` | "Contact" → `futurestudio.dev` |
| Page Title | (none) | "Future Studio" on the home page only (no suffix on other pages) |

## Files Modified

1. `<dir>/src/lib/shared.ts` - App name and GitHub config
2. `<dir>/src/app/(home)/page.tsx` - Hero section text and buttons
3. `<dir>/src/app/layout.tsx` - Page title metadata
