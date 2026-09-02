# WordPress performance notes

How I take a WordPress site from **over 8 seconds** to **under 2**, and hold **90+ Google PageSpeed** across every site I ship.

This is method, not client code. Fifteen-plus production sites since 2023, all built from scratch, all still maintained by me — the notes below are what actually moved the numbers, in the order that mattered.

— [Lojan Krishnapillai](https://github.com/Lojan-Krish) · Software Engineer, Jaffna, Sri Lanka

---

## The order is the whole trick

Most performance advice is a list of techniques. The list is not the hard part — the **sequence** is, because each step changes what the next measurement tells you.

| # | Step | Why here |
|---|---|---|
| 1 | **Images to WebP** | Did more than everything else combined. Most "slow sites" are slow because of one 4MB hero image nobody looked at. |
| 2 | **Critical CSS inlined** | The page starts painting instead of waiting on a stylesheet. |
| 3 | **Lazy loading** | Everything below the fold waits its turn. |
| 4 | **Server-side caching** | **Last.** Caching a slow page just serves the slow page faster to the second visitor. |

Do caching first and you get a number that looks better while the page is still badly built. Then you cannot tell what any later change did.

---

## 1 · Images

The single biggest win, every time.

- **Convert to WebP.** Keep a fallback via `<picture>` for anything that must work on ancient browsers; in practice WebP support is now universal enough that the fallback is insurance, not a plan.
- **Resize to the largest size actually rendered.** A 4000px-wide image in a 1200px container is 70% waste. WordPress generates size variants — make the theme request the right one rather than the full size.
- **Set `width` and `height` on every image.** This is a *layout shift* fix, not a bandwidth one, and it is the cheapest CLS improvement available.
- **Compress before upload, not after.** Plugins that optimise on upload still store the original; the disk cost compounds across a multi-site install.

A rough pass over a media directory:

```bash
# convert every jpg/png to webp at quality 82, keeping the original
find ./wp-content/uploads -type f \( -iname '*.jpg' -o -iname '*.png' \) \
  -exec sh -c 'cwebp -q 82 "$1" -o "${1%.*}.webp"' _ {} \;
```

Quality 82 is where I stop being able to tell the difference on photographic content. Flat graphics and screenshots go lower; logos should be SVG and not in this pass at all.

---

## 2 · Critical CSS

The browser will not paint until it has the CSS it needs for what is on screen.

- Extract the rules needed **above the fold**, inline them in `<head>`.
- Load the rest of the stylesheet asynchronously so it never blocks the first paint.
- Keep the inlined block small — past roughly 14KB you are trading one problem for another.

```php
// functions.php — inline critical CSS, defer the rest
add_action( 'wp_head', function () {
    $critical = get_theme_file_path( '/dist/critical.css' );
    if ( file_exists( $critical ) ) {
        echo '<style id="critical-css">' . file_get_contents( $critical ) . '</style>';
    }
}, 1 );

add_filter( 'style_loader_tag', function ( $tag, $handle ) {
    if ( 'theme-main' !== $handle ) {
        return $tag;
    }
    return str_replace(
        "rel='stylesheet'",
        "rel='preload' as='style' onload=\"this.rel='stylesheet'\"",
        $tag
    );
}, 10, 2 );
```

Always keep a `<noscript>` stylesheet link as a fallback. A preload-and-swap trick that leaves no-JS visitors with unstyled HTML is not an optimisation.

---

## 3 · Lazy loading

- WordPress adds `loading="lazy"` to images by default now. **Take it off the hero image.** Lazy-loading the largest element above the fold actively hurts LCP — the browser waits to discover the one image the score is measured on.
- Lazy-load iframes too. One embedded map or video player can pull more weight than the rest of the page.
- `fetchpriority="high"` on the hero tells the browser what matters before it has finished parsing.

```php
// stop WordPress lazy-loading the first in-content image
add_filter( 'wp_get_attachment_image_attributes', function ( $attr, $attachment, $size ) {
    static $first = true;
    if ( $first ) {
        $attr['loading']       = 'eager';
        $attr['fetchpriority'] = 'high';
        $first = false;
    }
    return $attr;
}, 10, 3 );
```

---

## 4 · Caching, last

Once the page is genuinely fast, caching makes it fast for everyone else.

- **Page cache** for anonymous visitors.
- **Object cache** if the host offers Redis or Memcached — this is what helps logged-in and admin pages, which a page cache cannot touch.
- **Browser cache headers** on static assets. Long max-age plus a build hash in the filename, so you can cache for a year and still ship changes.
- **Exclude** cart, checkout, account and any nonce-bearing page. A cached nonce is a broken form.

```apache
# .htaccess — long cache on hashed static assets
<IfModule mod_expires.c>
  ExpiresActive On
  ExpiresByType image/webp             "access plus 1 year"
  ExpiresByType image/svg+xml          "access plus 1 year"
  ExpiresByType text/css               "access plus 1 year"
  ExpiresByType application/javascript "access plus 1 year"
  ExpiresByType text/html              "access plus 0 seconds"
</IfModule>
```

---

## Core Web Vitals — what I aim at

| Metric | Target | What usually breaks it |
|---|---|---|
| **LCP** | under 2.5s | The hero image — too large, or lazy-loaded by mistake |
| **CLS** | under 0.1 | Images without dimensions; web fonts swapping; injected banners |
| **INP** | under 200ms | Too much JavaScript on the main thread, usually from plugins |
| **TTFB** | under 600ms | Host, no object cache, or an unindexed query |

Fonts deserve their own line: `font-display: swap`, preload the one weight used above the fold, and self-host rather than hitting a third-party origin. A font request to another domain costs a DNS lookup, a TLS handshake and a round trip before a single character appears.

---

## Serving one install across several domains

A site can answer on several domains without any of the usual redirect cost. I run four domains on one install this way.

**Parked domain aliases, not 301s.** A 301 sends the visitor somewhere else — second request, extra round trip, every alias a hop away from the real site. A parked alias serves the same install directly.

**The catch nobody mentions:** several domains serving identical content is a duplicate-content problem *unless* the canonical tag points every variant at one preferred URL. Aliases without canonicals are worse than redirects.

```php
// force one canonical host regardless of which alias was requested
add_action( 'wp_head', function () {
    $canonical_host = 'example.com';
    $path = wp_parse_url( home_url( add_query_arg( array() ) ), PHP_URL_PATH );
    if ( ! $path ) { $path = '/'; }
    printf( '<link rel="canonical" href="https://%s%s" />', esc_attr( $canonical_host ), esc_attr( $path ) );
}, 2 );
```

So: **aliases for speed, canonical tags for search, one place where the content actually lives.**

---

## Measuring, honestly

- Test the **same page** before and after. A homepage and a blog post are not comparable.
- Test **cold**. A warmed cache flatters you.
- Lab tools (Lighthouse, PageSpeed Insights) find causes. **Field data** (Search Console Core Web Vitals) tells you whether real users felt it. They disagree often, and the field data is the one that counts.
- Record the number before you touch anything. Without it you cannot tell improvement from noise.

---

## Results this produced

| | |
|---|---|
| **90+** | Google PageSpeed on 100% of deployments |
| **8s to under 2s** | load time on a rebuild, following the order above |
| **60% faster** | page loads on a client site rebuild |
| **20% lower** | bounce rate on the same rebuild |
| **150%** | organic search visibility lift for a client |

---

## What I would tell you if you only had an hour

1. Open the site, find the largest image, convert it and size it properly. Measure again.
2. That is usually most of the win. Everything after it is refinement.

The lesson I keep relearning: performance is not a phase at the end. I settle it while I am choosing the images, not after the build ships.

---

MIT licensed. The techniques are standard; the sequencing and the measurements are from my own builds.
