# CDN Caching Patterns

Comprehensive guide for CDN caching strategies, cache invalidation, and edge computing patterns.

## When to Apply

- Setting up CDN caching for static and dynamic content
- Configuring cache headers and TTLs
- Implementing cache invalidation strategies
- Optimizing for cache hit ratios
- Building edge computing applications
- Debugging cache behavior

## Cache-Control Headers

### Basic Directives
```http
# Public cache (CDN can cache)
Cache-Control: public, max-age=31536000

# Private cache (browser only, no CDN)
Cache-Control: private, max-age=3600

# No caching at all
Cache-Control: no-store

# Must revalidate before using cached copy
Cache-Control: no-cache

# Immutable content (never changes)
Cache-Control: public, max-age=31536000, immutable
```

### Cache Duration Guidelines
| Content Type | TTL | Cache-Control |
|--------------|-----|---------------|
| HTML pages | 0 or short | `no-cache` or `max-age=60` |
| API responses | 0-300s | `private, max-age=60` |
| JS/CSS (hashed) | 1 year | `public, max-age=31536000, immutable` |
| Images | 1 week - 1 year | `public, max-age=604800` |
| Fonts | 1 year | `public, max-age=31536000` |
| User-specific data | 0 | `private, no-store` |

### Stale Content Directives
```http
# Serve stale while revalidating (background refresh)
Cache-Control: public, max-age=3600, stale-while-revalidate=86400

# Serve stale if origin is down
Cache-Control: public, max-age=3600, stale-if-error=86400

# Combined
Cache-Control: public, max-age=300, stale-while-revalidate=3600, stale-if-error=86400
```

## Caching Strategies

### Cache Everything (Static Sites)
```
Origin → CDN → User

Rules:
- HTML: max-age=60, stale-while-revalidate=3600
- Assets: max-age=31536000, immutable (use content hashing)
- Images: max-age=604800

Invalidation: Purge on deploy
```

### Cache Static, Bypass Dynamic
```
Static requests → CDN cache
Dynamic requests → Origin (bypass cache)

Rules:
- /api/* → bypass cache
- /user/* → bypass cache
- *.html → short TTL or no-cache
- /static/* → long TTL
```

### Edge Cache with Origin Shield
```
User → Edge POP → Shield POP → Origin

Benefits:
- Reduces origin load
- Single point of cache population
- Better cache hit ratio
```

### Tiered Caching
```
L1: Browser cache (private)
L2: CDN edge (public, short TTL)
L3: Origin cache (Redis/Varnish)
L4: Database
```

## Cache Keys

### Default Cache Key Components
- Protocol (http/https)
- Host
- Path
- Query string

### Custom Cache Keys
```javascript
// Cloudflare Worker: Custom cache key
const cacheKey = new Request(url, {
  cf: {
    cacheKey: `${url.pathname}:${country}:${deviceType}`
  }
});

// Vary by:
// - Country for localized content
// - Device type for responsive images
// - User segment for A/B tests
// - Accept header for content negotiation
```

### Cache Key Best Practices
```
# Good: Consistent, predictable
/products/123
/products/123?v=2

# Bad: Random or user-specific
/products/123?timestamp=1234567890
/products/123?session=abc123
```

## Cache Invalidation

### Strategies

#### 1. TTL-Based (Let it Expire)
```http
Cache-Control: max-age=300
# Content refreshes every 5 minutes automatically
```
- Simple, no invalidation needed
- Trade-off: staleness vs freshness

#### 2. Purge on Deploy
```bash
# Cloudflare
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone}/purge_cache" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"purge_everything": true}'

# Fastly
curl -X POST "https://api.fastly.com/service/{service}/purge_all" \
  -H "Fastly-Key: $TOKEN"

# CloudFront
aws cloudfront create-invalidation \
  --distribution-id $DIST_ID \
  --paths "/*"
```

#### 3. Tag-Based Purging (Surrogate Keys)
```http
# Origin response
Surrogate-Key: product-123 category-electronics all-products

# Purge by tag
curl -X POST "https://api.fastly.com/service/{service}/purge/product-123" \
  -H "Fastly-Key: $TOKEN"
```

#### 4. Versioned URLs (Cache Busting)
```html
<!-- Never need to invalidate -->
<script src="/js/app.abc123.js"></script>
<link href="/css/styles.def456.css">

<!-- Or with query string -->
<script src="/js/app.js?v=abc123"></script>
```

### Invalidation Patterns
```python
# Event-driven invalidation
def on_product_update(product_id):
    # Update database
    db.update_product(product_id, data)
    
    # Invalidate CDN
    cdn.purge_by_tag(f"product-{product_id}")
    cdn.purge_url(f"/products/{product_id}")
    cdn.purge_url(f"/api/products/{product_id}")

# Batch invalidation
def on_category_update(category_id):
    product_ids = db.get_products_by_category(category_id)
    
    cdn.purge_by_tag(f"category-{category_id}")
    for product_id in product_ids:
        cdn.purge_by_tag(f"product-{product_id}")
```

## Edge Computing

### Edge Functions (Cloudflare Workers)
```javascript
// A/B Testing at the edge
export default {
  async fetch(request) {
    const url = new URL(request.url);
    
    // Get or create experiment bucket
    let bucket = request.cookies.get('exp_bucket');
    if (!bucket) {
      bucket = Math.random() < 0.5 ? 'A' : 'B';
    }
    
    // Route to variant
    if (bucket === 'B') {
      url.pathname = url.pathname.replace('/page', '/page-v2');
    }
    
    const response = await fetch(url, request);
    
    // Set cookie for consistency
    const newResponse = new Response(response.body, response);
    newResponse.headers.set('Set-Cookie', `exp_bucket=${bucket}; Path=/; Max-Age=86400`);
    
    return newResponse;
  }
};
```

### Edge Caching with Workers
```javascript
// Custom caching logic
export default {
  async fetch(request, env, ctx) {
    const cache = caches.default;
    const cacheKey = new Request(request.url, request);
    
    // Check cache
    let response = await cache.match(cacheKey);
    if (response) {
      return response;
    }
    
    // Fetch from origin
    response = await fetch(request);
    
    // Clone and cache
    const responseToCache = response.clone();
    ctx.waitUntil(cache.put(cacheKey, responseToCache));
    
    return response;
  }
};
```

### Geolocation-Based Content
```javascript
export default {
  async fetch(request) {
    const country = request.cf.country;
    const city = request.cf.city;
    
    // Redirect to localized site
    if (country === 'DE') {
      return Response.redirect('https://de.example.com', 302);
    }
    
    // Serve localized content
    const localizedUrl = new URL(request.url);
    localizedUrl.searchParams.set('locale', country.toLowerCase());
    
    return fetch(localizedUrl);
  }
};
```

## Vary Header

### Correct Usage
```http
# Vary by Accept-Encoding (compression)
Vary: Accept-Encoding

# Vary by multiple headers
Vary: Accept-Encoding, Accept-Language

# Vary by custom header (use sparingly)
Vary: X-Device-Type
```

### Vary Gotchas
```http
# Bad: Vary by Cookie (defeats caching)
Vary: Cookie

# Bad: Vary by User-Agent (too many variants)
Vary: User-Agent

# Solution: Normalize into buckets at edge
# User-Agent: Mobile Safari → X-Device-Type: mobile
# User-Agent: Chrome Windows → X-Device-Type: desktop
```

## Performance Metrics

### Cache Hit Ratio
```
Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)

Target: > 90% for static content
```

### Time to First Byte (TTFB)
```
Cache HIT: ~10-50ms
Cache MISS: ~100-500ms (depends on origin)
```

### Monitoring
```bash
# Check cache status header
curl -I https://example.com/page | grep -i cf-cache-status
# HIT, MISS, EXPIRED, STALE, BYPASS, DYNAMIC

# CloudFront
x-cache: Hit from cloudfront
x-cache: Miss from cloudfront
```

## Common Gotchas

### Query String Caching
```
# Problem: Different cache entries
/page?a=1&b=2
/page?b=2&a=1

# Solution: Sort query params at edge or normalize
```

### Cookie Contamination
```
# Problem: Cookie in cache key prevents sharing
Set-Cookie: session=abc123

# Solution:
# 1. Strip cookies at edge for static content
# 2. Use separate domains (static.example.com)
# 3. Set cookies only where needed
```

### Cache Stampede
```
# Problem: TTL expires, all requests hit origin

# Solution 1: Stale-while-revalidate
Cache-Control: max-age=60, stale-while-revalidate=300

# Solution 2: Jittered TTL
ttl = base_ttl + random(0, jitter)

# Solution 3: Single-flight / request coalescing
# CDN feature: only one request to origin
```

### Private Data Leaking
```
# Problem: User-specific data cached publicly

# Solution: Always set correct Cache-Control
Cache-Control: private, no-store

# And verify at edge
if (request.cookies.has('session')) {
  response.headers.set('Cache-Control', 'private, no-store');
}
```

## CDN Configuration Examples

### Cloudflare Page Rules
```yaml
# Cache everything
URL: example.com/*
Cache Level: Cache Everything
Edge TTL: 1 day

# Bypass cache for API
URL: example.com/api/*
Cache Level: Bypass

# Custom TTL
URL: example.com/static/*
Edge TTL: 1 year
Browser TTL: 1 year
```

### CloudFront Cache Behaviors
```yaml
# Static assets
PathPattern: /static/*
CachePolicyId: Managed-CachingOptimized
TTL: 86400

# API
PathPattern: /api/*
CachePolicyId: Managed-CachingDisabled
OriginRequestPolicyId: AllViewer

# Default (HTML)
PathPattern: Default (*)
CachePolicyId: Managed-CachingOptimizedForUncompressedObjects
TTL: 60
```

## References

- [HTTP Caching (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Cache-Control Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [Cloudflare Caching](https://developers.cloudflare.com/cache/)
- [CloudFront Caching](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ConfiguringCaching.html)
- [Web.dev Caching Guide](https://web.dev/http-cache/)
