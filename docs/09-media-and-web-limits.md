> 中文标题：媒体与 Web 限制
>
> English title: Media And Web Limits
>
> 中文导读：本章整理图片、PDF、媒体数量、WebFetch、URL、HTTP 内容、超时、缓存和重定向策略的具体默认值。
>
> English guide: This chapter collects concrete defaults for images, PDFs, media counts, WebFetch, URLs, HTTP content, timeouts, caches, and redirect policy.

# 9. Media And Web Limits

## 9.1 Images And PDFs

| Setting | Value |
|---|---:|
| Image base64 max | `5 MB` |
| Image raw target size | `3.75 MB` |
| Image max width | `2000 px` |
| Image max height | `2000 px` |
| PDF raw target size | `20 MB` |
| PDF extract threshold | `3 MB` |
| PDF extraction max size | `100 MB` |
| PDF API max pages | `100` |
| PDF pages per Read call | `20` |
| Inline PDF @ mention threshold | `10 pages` |
| Max media items per request | `100` |

If a request has more than 100 media items, drop oldest media or fail with a clear error before API call.

## 9.2 Web Fetch

| Setting | Value |
|---|---:|
| URL max length | `2_000 chars` |
| HTTP content max | `10 MB` |
| Fetch timeout | `60_000 ms` |
| Domain check timeout | `10_000 ms` |
| Max redirects | `10` |
| Markdown max length | `100_000 chars` |
| URL cache TTL | `15 minutes` |
| URL cache size | `50 MB` |
| Domain check cache TTL | `5 minutes` |
| Domain check cache entries | `128` |

Redirect policy:

- allow same origin,
- allow adding/removing `www.`,
- block protocol change,
- block port change,
- block username/password,
- return redirect info instead of silently following unsafe redirect.
