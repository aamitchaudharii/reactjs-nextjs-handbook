# P5 · Real-World Project: CMS & Blog Platform

> **A blog/CMS platform appears simple on the surface but contains a surprisingly deep set of engineering decisions: how to handle MDX rendering at scale, how to implement full-text search without a separate service, how to manage the content pipeline from draft to published, how to implement ISR so a single revalidation doesn't rebuild 10,000 posts, and how to make the reading experience fast on every device. This project covers the engineering decisions that make a content-heavy Next.js application fast, maintainable, and authorable.**

---

## Project Overview

**What you'll build:**

- MDX-based blog with rich component embedding
- Content management via file system (Git-based) or database CMS
- Full-text search with Pagefind (static search index)
- Table of contents generation
- Reading time estimation
- Draft / Published workflow
- Author profiles and tag system
- RSS feed generation
- OpenGraph image generation

**Technology choices:**

- Next.js 15 (App Router)
- MDX with next/mdx (content rendering)
- Contentlayer2 or Velite (content pipeline — type-safe MDX processing)
- Pagefind (static full-text search — zero server needed)
- Shiki (syntax highlighting — server-side, zero client JS)
- @vercel/og (OpenGraph image generation at the Edge)

---

## Architecture Decision Record

### ADR-1: Content Storage Strategy

```
THREE APPROACHES TO CONTENT STORAGE:

OPTION A: File-system MDX (Git-based CMS)
  Content lives in /content/*.mdx files committed to the repository.
  Writers use GitHub/GitLab directly or a Git-backed editor (Prose.io, Decap CMS).
  PRO: version history is Git history; content is code-reviewable; no database needed
  PRO: zero infrastructure cost; works with pure SSG
  CON: non-technical writers need to understand Git and MDX
  CON: new posts require a deployment (unless using on-demand ISR)
  USE WHEN: technical blog, documentation site, personal site

OPTION B: Headless CMS (Contentful, Sanity, Payload)
  Content stored in a cloud CMS with a rich editing UI.
  Next.js fetches from the CMS API at build time (SSG) or request time (SSR).
  PRO: non-technical writers can edit via a GUI
  PRO: content updates without code deployments (webhooks trigger revalidation)
  CON: CMS cost + vendor dependency
  CON: API latency adds to build time for large content sets
  USE WHEN: marketing team manages content, non-technical writers

OPTION C: Database-backed (Prisma + PostgreSQL)
  Posts stored in a PostgreSQL database; custom admin UI for editing.
  PRO: full control over the schema and the editing experience
  PRO: complex queries (related posts, personalization) are easy
  CON: need to build the editing UI from scratch
  CON: no version history without implementing it yourself
  USE WHEN: platform with complex content relationships, user-generated content

OUR APPROACH: File-system MDX (Option A)
  Reasoning: technical blog audience, demonstrates the Next.js + MDX
  integration patterns most developers encounter.
  Content pipeline: MDX files → Contentlayer (type-safe processing) →
  Next.js SSG → Pagefind (search index at build time)
```

---

### ADR-2: MDX Content Pipeline with Contentlayer

```
THE PROBLEM WITH RAW MDX:
  You can import .mdx files directly in Next.js, but then:
  - No type safety on frontmatter (author, date, tags are untyped strings)
  - No build-time validation (missing required fields fail at render, not at build)
  - No derived computed fields (slug from filename, reading time from word count)
  - No relationships between content types (post → author)

CONTENTLAYER SOLVES THIS:
  Defines a schema for each content type (Post, Author, Tag)
  At build time: reads all MDX files, validates frontmatter, computes derived fields,
  generates TypeScript types for everything
  Import posts in your pages with full TypeScript type safety
```

```ts
// contentlayer.config.ts
import { defineDocumentType, makeSource } from "contentlayer2/source-files";
import rehypeShiki from "@shikijs/rehype";
import rehypeSlug from "rehype-slug";
import rehypeAutolinkHeadings from "rehype-autolink-headings";
import remarkGfm from "remark-gfm";

export const Post = defineDocumentType(() => ({
  name: "Post",
  filePathPattern: "posts/**/*.mdx",
  contentType: "mdx",
  fields: {
    title: { type: "string", required: true },
    description: { type: "string", required: true },
    date: { type: "date", required: true },
    author: { type: "string", required: true },
    tags: { type: "list", of: { type: "string" }, default: [] },
    published: { type: "boolean", default: false },
    featuredImage: { type: "string" }, // optional
  },
  computedFields: {
    // Slug derived from file path (content/posts/2024/my-post.mdx → my-post):
    slug: {
      type: "string",
      resolve: (doc) => doc._raw.flattenedPath.split("/").pop()!,
    },
    // Reading time computed from word count:
    readingTime: {
      type: "number",
      resolve: (doc) => {
        const words = doc.body.raw.split(/\s+/g).length;
        return Math.ceil(words / 238); // average reading speed: 238 wpm
      },
    },
    // Structured table of contents extracted from headings:
    headings: {
      type: "json",
      resolve: async (doc) => {
        const headingRegex = /^#{2,3}\s+(.+)$/gm;
        return [...doc.body.raw.matchAll(headingRegex)].map(([, text]) => ({
          text: text.trim(),
          slug: text
            .trim()
            .toLowerCase()
            .replace(/\s+/g, "-")
            .replace(/[^\w-]/g, ""),
          level: /* count #s */ 2,
        }));
      },
    },
  },
}));

export default makeSource({
  contentDirPath: "content",
  documentTypes: [Post],
  mdx: {
    remarkPlugins: [remarkGfm],
    rehypePlugins: [
      rehypeSlug,
      [rehypeAutolinkHeadings, { behavior: "wrap" }],
      [
        rehypeShiki,
        {
          themes: { light: "github-light", dark: "github-dark" },
          // Server-side syntax highlighting: ZERO client JS for code blocks
        },
      ],
    ],
  },
});
```

---

### ADR-3: SSG with On-Demand ISR for Draft Preview

```
CHALLENGE: Writers need to preview drafts before publishing.
But the published posts are fully static (SSG).

SOLUTION: two rendering modes

PUBLISHED POSTS (/blog/[slug]):
  Statically generated at build time via generateStaticParams.
  Only includes published: true posts.
  On-demand revalidation when a new post is published (webhook triggers
  revalidatePath('/blog/[slug]') via a protected API route).

DRAFT PREVIEW (/blog/[slug]?draft=true&token=SECRET):
  Force-dynamic rendering with a secret preview token.
  Bypasses the static content and renders the current MDX file directly.
  Only accessible with the correct secret token (known to writers, not public).
```

```ts
// app/blog/[slug]/page.tsx
import { allPosts } from 'contentlayer/generated';

export async function generateStaticParams() {
  return allPosts
    .filter(post => post.published) // only published posts get static pages
    .map(post => ({ slug: post.slug }));
}

export default async function BlogPostPage({
  params,
  searchParams,
}: {
  params: { slug: string };
  searchParams: { draft?: string; token?: string };
}) {
  // Check for draft preview mode:
  const isDraftPreview =
    searchParams.draft === 'true' &&
    searchParams.token === process.env.PREVIEW_SECRET;

  const post = allPosts.find(p =>
    p.slug === params.slug &&
    (isDraftPreview || p.published) // show unpublished only in preview mode
  );

  if (!post) notFound();

  return <PostLayout post={post} isDraftPreview={isDraftPreview} />;
}
```

---

### ADR-4: Static Full-Text Search with Pagefind

```
THE CHALLENGE: Search on a static site.
Options:
  A. Algolia / Typesense (cloud search) — cost, external dependency
  B. Fuse.js (in-memory client-side search) — loads entire content index in browser
  C. Pagefind (static search index generated at build time) — CHOSEN

PAGEFIND:
  At build time: Pagefind crawls the built HTML files, creates a binary search index
  At runtime: the index is loaded progressively (only the relevant shard)
  NO server needed; NO external service; fast and bandwidth-efficient

BUILD PROCESS WITH PAGEFIND:
  1. next build (generates .next/out or .vercel/output)
  2. pagefind --site out/ (indexes the static HTML)
  3. Deploy both the site AND the pagefind index files

INTEGRATION:
  Pagefind uses a browser API (loads WASM and binary indexes).
  Must be wrapped in dynamic import with ssr: false.
```

```tsx
// features/search/components/SearchDialog.tsx
"use client";
import { useState, useEffect } from "react";

interface PagefindResult {
  url: string;
  meta: { title: string };
  excerpt: string;
}

export function SearchDialog({
  isOpen,
  onClose,
}: {
  isOpen: boolean;
  onClose: () => void;
}) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<PagefindResult[]>([]);
  const [pagefind, setPagefind] = useState<any>(null);

  // Load Pagefind lazily (only when the search dialog opens):
  useEffect(() => {
    if (!isOpen || pagefind) return;
    import("/pagefind/pagefind.js").then((pf) => {
      pf.init();
      setPagefind(pf);
    });
  }, [isOpen]);

  useEffect(() => {
    if (!pagefind || !query) {
      setResults([]);
      return;
    }

    const debounceId = setTimeout(async () => {
      const search = await pagefind.search(query);
      const data = await Promise.all(
        search.results.slice(0, 10).map((r: any) => r.data()),
      );
      setResults(data);
    }, 200);

    return () => clearTimeout(debounceId);
  }, [query, pagefind]);

  return (
    <dialog open={isOpen} aria-label="Search posts">
      <input
        autoFocus
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search posts..."
        aria-label="Search query"
      />
      <ul role="listbox">
        {results.map((result) => (
          <li key={result.url} role="option">
            <a href={result.url} onClick={onClose}>
              <strong>{result.meta.title}</strong>
              {/* Pagefind highlights the matched terms in the excerpt: */}
              <span dangerouslySetInnerHTML={{ __html: result.excerpt }} />
            </a>
          </li>
        ))}
      </ul>
    </dialog>
  );
}
```

---

## OpenGraph Image Generation

```ts
// app/blog/[slug]/opengraph-image.tsx
// Next.js automatically serves this as the og:image for each blog post

import { ImageResponse } from 'next/og';
import { allPosts } from 'contentlayer/generated';

export const runtime = 'edge'; // edge for fast global OG image generation

export async function generateImageMetadata({
  params,
}: {
  params: { slug: string };
}) {
  const post = allPosts.find(p => p.slug === params.slug);
  return [{ id: params.slug, alt: post?.title ?? 'Blog Post' }];
}

export default async function OGImage({ params }: { params: { slug: string } }) {
  const post = allPosts.find(p => p.slug === params.slug);

  // Load custom font for the OG image:
  const fontData = await fetch(
    new URL('../../../../public/fonts/InterBold.ttf', import.meta.url)
  ).then(res => res.arrayBuffer());

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          flexDirection: 'column',
          justifyContent: 'flex-end',
          width: '100%',
          height: '100%',
          padding: '60px',
          background: 'linear-gradient(135deg, #1e293b 0%, #0f172a 100%)',
        }}
      >
        <div style={{ fontSize: 64, fontWeight: 700, color: '#f8fafc', lineHeight: 1.2 }}>
          {post?.title ?? 'Blog Post'}
        </div>
        <div style={{ fontSize: 28, color: '#94a3b8', marginTop: 24 }}>
          {post?.description}
        </div>
        <div style={{ display: 'flex', alignItems: 'center', marginTop: 48 }}>
          <div style={{ fontSize: 22, color: '#64748b' }}>
            {post ? formatDate(post.date) : ''} · {post?.readingTime ?? 0} min read
          </div>
        </div>
      </div>
    ),
    {
      width: 1200,
      height: 630,
      fonts: [{ name: 'Inter', data: fontData, weight: 700 }],
    }
  );
}
```

---

## RSS Feed Generation

```ts
// app/feed.xml/route.ts — server-generated RSS feed
import { allPosts } from "contentlayer/generated";

export async function GET() {
  const posts = allPosts
    .filter((p) => p.published)
    .sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime())
    .slice(0, 20); // latest 20 posts

  const rss = `<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>My Blog</title>
    <link>https://myblog.com</link>
    <description>Articles on React, Next.js, and frontend engineering</description>
    <atom:link href="https://myblog.com/feed.xml" rel="self" type="application/rss+xml"/>
    <language>en</language>
    <lastBuildDate>${new Date().toUTCString()}</lastBuildDate>
    ${posts
      .map(
        (post) => `
    <item>
      <title><![CDATA[${post.title}]]></title>
      <link>https://myblog.com/blog/${post.slug}</link>
      <guid isPermaLink="true">https://myblog.com/blog/${post.slug}</guid>
      <description><![CDATA[${post.description}]]></description>
      <pubDate>${new Date(post.date).toUTCString()}</pubDate>
      <author>contact@myblog.com (${post.author})</author>
      ${post.tags.map((tag) => `<category>${tag}</category>`).join("")}
    </item>`,
      )
      .join("")}
  </channel>
</rss>`;

  return new Response(rss, {
    headers: {
      "Content-Type": "application/xml",
      "Cache-Control": "public, max-age=3600, s-maxage=3600",
    },
  });
}
```

---

## Performance Strategy for Content-Heavy Sites

```
IMAGES IN MDX:
  next/image doesn't work in MDX by default (MDX renders <img>, not <Image>).
  Override the MDX img component to use next/image:

  // app/blog/[slug]/page.tsx
  import { getMDXComponent } from 'contentlayer2/hooks';
  import Image from 'next/image';

  const MDXImage = ({ src, alt }: { src: string; alt: string }) => (
    <Image
      src={src} alt={alt ?? ''}
      width={800} height={450}
      className="post-image"
      style={{ width: '100%', height: 'auto' }}
    />
  );

  const MDX = getMDXComponent(post.body.code);
  return <MDX components={{ img: MDXImage }} />;

SYNTAX HIGHLIGHTING:
  Shiki runs at BUILD TIME (configured in contentlayer.config.ts).
  The output is pre-highlighted HTML — ZERO client JavaScript for code blocks.
  Supports dual themes (light/dark) via CSS custom properties.

FONTS:
  Use next/font with display='swap' for web fonts.
  Self-hosted via next/font (no Google Fonts privacy concerns, no extra DNS lookup).

CODE SPLITTING:
  The search dialog is dynamic import (only loads Pagefind WASM when search opens).
  The table of contents is a Server Component (no client JS needed).
  Comments (if using Giscus/Disqus) are dynamic imported, loaded after scroll.
```

---

## Testing Strategy

```
UNIT TESTS:
  - Reading time calculation (edge cases: empty post, very long post)
  - Slug generation from file paths
  - Table of contents extraction from MDX headings
  - RSS feed XML structure (valid XML, correct post count)

INTEGRATION TESTS:
  - Search returns correct results for known queries
  - OG image generation returns a valid image response
  - Draft preview correctly shows unpublished posts with valid token
  - Draft preview correctly rejects requests without valid token

E2E TESTS:
  - Homepage shows latest published posts
  - Click post → verify full post content is visible
  - Open search → type query → verify results appear
  - Click result → verify navigation to correct post
  - Verify RSS feed URL returns valid XML
  - Verify OG image meta tags are present on post pages
```

---

## Key Learning Outcomes

After building this project, you should be able to articulate:

1. **The content pipeline pattern** — how Contentlayer transforms raw MDX files into type-safe, computed data structures that Next.js components consume with full TypeScript safety

2. **SSG + on-demand ISR for content** — how statically generated posts are invalidated when content is published, without rebuilding the entire site

3. **Static search without a server** — how Pagefind builds a binary search index at build time and loads it progressively in the browser, achieving fast full-text search with zero external services

4. **Server-side syntax highlighting** — why Shiki runs at build time (via rehype plugins) rather than in the browser, eliminating the client-side JavaScript bundle for code highlighting

5. **Edge OG image generation** — how Next.js's `ImageResponse` generates custom OpenGraph images at the Edge runtime, and how to use custom fonts and dynamic content in the image

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
