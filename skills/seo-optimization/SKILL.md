---
name: seo-optimization
description: Universal master blueprint for complete Search Engine Optimization (Technical SEO, On-Page SEO, Metadata Engine, JSON-LD Structured Data, XML Sitemaps, Robots.txt, and Core Web Vitals) across any modern web application or website (SaaS, Agency, E-Commerce, Landing Pages, or Content platforms).
---

# Universal SEO Optimization Master Skill

A framework-agnostic, production-grade master blueprint for implementing comprehensive Search Engine Optimization across any modern web application.

---

## 1. Universal Architecture Overview

Every high-ranking website relies on 5 foundational SEO pillars:

```
┌─────────────────────────────────────────────────────────────┐
│                    MODERN SEO ARCHITECTURE                  │
├──────────────────────────────┬──────────────────────────────┤
│ 1. Metadata Engine           │ Title, Description, OG, X    │
│ 2. Crawl & Index Directives  │ Robots.txt, Dynamic Sitemap  │
│ 3. Structured Data (JSON-LD) │ Organization, Product, FAQ   │
│ 4. Semantic Hierarchy        │ H1-H6, Landmarks, Alt Tags   │
│ 5. Core Web Vitals           │ SSR/SSG, LCP Priority, 0 CLS │
└──────────────────────────────┴──────────────────────────────┘
```

---

## 2. Centralized Metadata Engine

Maintain a single, reusable metadata helper to enforce consistent meta tags, canonical links, and social cards across all static and dynamic pages.

### Implementation Pattern (`src/utils/metadata.ts`)

```typescript
import { Metadata } from "next";

interface MetadataProps {
    title?: string;
    description?: string;
    image?: string | null;
    icons?: Metadata["icons"];
    noIndex?: boolean;
    canonical?: string;
    type?: "website" | "article";
}

export const generateMetadata = ({
    title = "BrandName - Value Proposition Tagline",
    description = "A concise, keyword-focused description between 140-160 characters describing the site.",
    image = "/thumbnail.png",
    icons = [
        { rel: "apple-touch-icon", sizes: "180x180", url: "/apple-touch-icon.png" },
        { rel: "icon", sizes: "32x32", url: "/favicon-32x32.png" },
        { rel: "icon", sizes: "16x16", url: "/favicon-16x16.png" },
    ],
    noIndex = false,
    canonical,
    type = "website",
}: MetadataProps = {}): Metadata => {
    const siteUrl = "https://yourdomain.com";

    return {
        metadataBase: new URL(siteUrl),
        title: {
            default: title,
            template: `%s | BrandName`,
        },
        description,
        icons,
        alternates: {
            canonical: canonical || "/",
        },
        openGraph: {
            title,
            description,
            url: canonical ? `${siteUrl}${canonical}` : siteUrl,
            siteName: "BrandName",
            type,
            ...(image && {
                images: [
                    {
                        url: image,
                        width: 1200,
                        height: 630,
                        alt: title,
                    },
                ],
            }),
        },
        twitter: {
            card: "summary_large_image",
            title,
            description,
            ...(image && { images: [image] }),
        },
        robots: noIndex
            ? { index: false, follow: false }
            : {
                  index: true,
                  follow: true,
                  googleBot: {
                      index: true,
                      follow: true,
                      "max-video-preview": -1,
                      "max-image-preview": "large",
                      "max-snippet": -1,
                  },
              },
    };
};
```

---

## 3. Crawler Directives & Indexation

### 3.1 Search Crawler Directives (`src/app/robots.ts`)
Control bot access and point search engines directly to your dynamic sitemap.

```typescript
import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
    return {
        rules: {
            userAgent: '*',
            allow: '/',
            disallow: ['/dashboard/', '/admin/', '/api/', '/checkout/'],
        },
        sitemap: 'https://yourdomain.com/sitemap.xml',
    };
}
```

### 3.2 Dynamic XML Sitemap (`src/app/sitemap.ts`)
Generate an updated XML manifest with priority weighting and update frequencies for all static and database-driven pages.

```typescript
import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
    const baseUrl = 'https://yourdomain.com';

    // Static pages with hierarchy weighting
    const staticRoutes = [
        { route: '', priority: 1.0, changeFrequency: 'weekly' as const },
        { route: '/products', priority: 0.9, changeFrequency: 'weekly' as const },
        { route: '/services', priority: 0.8, changeFrequency: 'weekly' as const },
        { route: '/pricing', priority: 0.8, changeFrequency: 'monthly' as const },
        { route: '/about', priority: 0.7, changeFrequency: 'monthly' as const },
        { route: '/contact', priority: 0.7, changeFrequency: 'monthly' as const },
        { route: '/privacy', priority: 0.3, changeFrequency: 'yearly' as const },
        { route: '/terms', priority: 0.3, changeFrequency: 'yearly' as const },
    ].map(({ route, priority, changeFrequency }) => ({
        url: `${baseUrl}${route}`,
        lastModified: new Date(),
        changeFrequency,
        priority,
    }));

    // Dynamic programmatic entities (e.g. Products, Services, Case Studies)
    // const dynamicItems = await fetchDynamicEntities();
    // const dynamicRoutes = dynamicItems.map(item => ({
    //     url: `${baseUrl}/products/${item.slug}`,
    //     lastModified: new Date(item.updatedAt),
    //     changeFrequency: 'weekly' as const,
    //     priority: 0.8,
    // }));

    return [...staticRoutes];
}
```

---

## 4. Structured Data (Schema.org / JSON-LD)

Structured data enables Google Rich Snippets (Knowledge Panels, Search Sitelinks, FAQ accordions, Product cards).

### 4.1 Organization & WebSite Schema (`src/app/layout.tsx`)
Embed global brand schema inside the root layout:

```tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
    const orgSchema = {
        "@context": "https://schema.org",
        "@type": "Organization",
        "name": "BrandName",
        "url": "https://yourdomain.com",
        "logo": "https://yourdomain.com/icons/logo.png",
        "sameAs": [
            "https://twitter.com/yourhandle",
            "https://linkedin.com/company/yourbrand",
            "https://github.com/yourbrand"
        ]
    };

    const websiteSchema = {
        "@context": "https://schema.org",
        "@type": "WebSite",
        "name": "BrandName",
        "url": "https://yourdomain.com"
    };

    return (
        <html lang="en">
            <head>
                <script
                    type="application/ld+json"
                    dangerouslySetInnerHTML={{ __html: JSON.stringify(orgSchema) }}
                />
                <script
                    type="application/ld+json"
                    dangerouslySetInnerHTML={{ __html: JSON.stringify(websiteSchema) }}
                />
            </head>
            <body>{children}</body>
        </html>
    );
}
```

### 4.2 Page-Specific Schema Templates

* **Product / Software**:
  ```json
  {
    "@context": "https://schema.org",
    "@type": "SoftwareApplication",
    "name": "Product Name",
    "operatingSystem": "Web",
    "applicationCategory": "BusinessApplication",
    "offers": {
      "@type": "Offer",
      "price": "0",
      "priceCurrency": "USD"
    }
  }
  ```
* **FAQ Page**:
  ```json
  {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    "mainEntity": [
      {
        "@type": "Question",
        "name": "What services do you provide?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "We build scalable web and mobile software solutions."
        }
      }
    ]
  }
  ```

---

## 5. Semantic HTML & Content Architecture

1. **Heading Tag Rules**:
   * Exactly **one `<h1>` tag** per page containing the primary focus keyword.
   * `<h2>` for major sections and topics.
   * `<h3>` to `<h6>` for cards, subsections, and features. Never skip levels (e.g. `<h1>` directly to `<h4>`).
2. **Semantic Landmarks**:
   * Use `<header>`, `<nav>`, `<main>`, `<section>`, `<article>`, `<aside>`, and `<footer>` rather than generic `<div>` wrappers.
3. **Internal Linking & Anchor Text**:
   * Use crawlable internal links (`<Link href="...">`).
   * Use keyword-rich, descriptive anchor text (e.g., *"Explore our custom web development services"* instead of *"Click here"*).
4. **Image SEO**:
   * Always provide descriptive `alt` text explaining what the image shows.
   * Use standard 16:9 or 1:1 aspect ratios with explicit width/height to avoid layout shifting.

---

## 6. Performance & Core Web Vitals (Technical SEO)

Google uses page experience as a direct ranking factor:

1. **Largest Contentful Paint (LCP < 2.5s)**:
   * Use `priority` on the main above-the-fold hero image (`<Image src="..." priority alt="..." />`).
   * Pre-render pages with Server-Side Rendering (SSR) or Static Site Generation (SSG).
2. **Cumulative Layout Shift (CLS < 0.1)**:
   * Load fonts using `next/font` (zero layout shift font swapping).
   * Specify explicit dimensions on images, videos, and embedded media.
3. **Interaction to Next Paint (INP < 200ms)**:
   * Avoid long JavaScript tasks blocking the main thread during hydration.

---

## 7. Universal Pre-Launch SEO Checklist

Before launching any new website or product:

- [ ] **Exact Domain Anchor**: `metadataBase` configured with the live production URL.
- [ ] **Html Language**: `<html lang="en">` declared on the root layout.
- [ ] **Crawler Permissions**: `src/app/robots.ts` allows indexation and references `sitemap.xml`.
- [ ] **XML Sitemap**: `src/app/sitemap.ts` exports all static and dynamic paths with proper priorities.
- [ ] **Social Sharing**: OpenGraph image exists at `1200x630px` resolution with Twitter Card tags.
- [ ] **Favicon Suite**: `favicon.ico`, `favicon-32x32.png`, `favicon-16x16.png`, and `apple-touch-icon.png` in `/public`.
- [ ] **Structured Data**: JSON-LD `Organization` schema in root layout.
- [ ] **Heading Audit**: Every page has exactly one `<h1>` containing the target keyword.
- [ ] **Image Audit**: Every `<Image>` or `<img>` has a meaningful `alt` attribute.
- [ ] **Canonical URLs**: `canonical` tag assigned to prevent duplicate URL indexing.
