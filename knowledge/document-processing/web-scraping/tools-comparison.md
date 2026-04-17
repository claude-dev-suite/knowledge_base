# Web Scraping -- Tools Comparison

## Overview / TL;DR

Choosing the right web scraping tool for RAG depends on whether target sites use JavaScript rendering, the volume of pages, budget constraints, and content extraction quality requirements. This guide provides systematic comparisons of Firecrawl, Crawl4AI, trafilatura, Scrapy, Playwright, and Beautiful Soup across JS rendering, speed, content quality, anti-bot handling, and cost.

---

## Feature Matrix

| Feature | Firecrawl | Crawl4AI | trafilatura | Scrapy | Playwright | Beautiful Soup |
|---------|-----------|----------|------------|--------|-----------|---------------|
| **JS rendering** | Yes | Yes (Playwright) | No | Via plugin | Yes | No |
| **Content extraction** | Excellent | Good | Excellent | Manual | Manual | Manual |
| **Markdown output** | Built-in | Built-in | Plain text | Manual | Manual | Manual |
| **Site crawling** | Built-in | Built-in | No | Excellent | Manual | No |
| **Anti-bot** | Managed | Basic | None | Middleware | Stealth plugin | None |
| **Async** | Yes | Yes | No | Yes | Yes | No |
| **Rate limiting** | Automatic | Manual | Manual | Built-in | Manual | Manual |
| **Proxy support** | Built-in | Manual | No | Built-in | Yes | Via requests |
| **robots.txt** | Respected | Manual | Manual | Built-in | Manual | Manual |
| **Max throughput** | ~50 pg/min | ~100 pg/min | ~200 pg/min | ~500 pg/min | ~30 pg/min | ~300 pg/min |
| **Setup effort** | API key | pip + Playwright | pip | pip + config | pip + browser | pip |
| **Local/Cloud** | Cloud | Local | Local | Local | Local | Local |
| **Cost** | $1/1K pages | Free | Free | Free | Free | Free |
| **License** | Commercial | Apache-2.0 | Apache-2.0 | BSD | Apache-2.0 | MIT |

---

## Content Extraction Quality

### Benchmark Approach

```python
from dataclasses import dataclass
from difflib import SequenceMatcher


@dataclass
class ExtractionQuality:
    tool: str
    url: str
    content_similarity: float   # vs ground truth
    boilerplate_ratio: float    # 0 = no boilerplate, 1 = all boilerplate
    heading_preservation: float # 0-1
    code_block_preservation: float
    table_preservation: float


def measure_extraction_quality(
    extracted: str,
    ground_truth: str,
    original_html: str,
) -> dict:
    """Measure content extraction quality."""
    # Content similarity to ground truth
    matcher = SequenceMatcher(
        None,
        " ".join(extracted.split()),
        " ".join(ground_truth.split()),
    )
    content_sim = matcher.ratio()

    # Boilerplate detection
    gt_words = set(ground_truth.lower().split())
    ext_words = extracted.lower().split()
    noise_words = sum(1 for w in ext_words if w not in gt_words)
    boilerplate = noise_words / max(len(ext_words), 1)

    # Heading preservation
    import re
    gt_headings = set(re.findall(r'^#{1,6}\s+(.+)$', ground_truth, re.MULTILINE))
    ext_headings = set(re.findall(r'^#{1,6}\s+(.+)$', extracted, re.MULTILINE))
    heading_score = (
        len(gt_headings & ext_headings) / max(len(gt_headings), 1)
        if gt_headings else 1.0
    )

    # Code block preservation
    gt_code = len(re.findall(r'```', ground_truth))
    ext_code = len(re.findall(r'```', extracted))
    code_score = min(ext_code, gt_code) / max(gt_code, 1) if gt_code else 1.0

    return {
        "content_similarity": round(content_sim, 3),
        "boilerplate_ratio": round(boilerplate, 3),
        "heading_preservation": round(heading_score, 3),
        "code_block_preservation": round(code_score, 3),
    }
```

### Typical Results by Site Type

**Static documentation (Read the Docs, Sphinx)**:

| Tool | Content Quality | Boilerplate | Speed |
|------|---------------|------------|-------|
| trafilatura | 95% | 5% | Very Fast |
| Firecrawl | 97% | 3% | Medium |
| Crawl4AI | 90% | 8% | Medium |
| Beautiful Soup | 70% (manual) | 15% | Fast |

**JavaScript-rendered docs (Docusaurus, Next.js)**:

| Tool | Content Quality | Boilerplate | Speed |
|------|---------------|------------|-------|
| trafilatura | 20-40% (misses JS content) | N/A | Fast |
| Firecrawl | 95% | 3% | Medium |
| Crawl4AI | 92% | 5% | Medium |
| Playwright + extraction | 90% | 10% | Slow |

**Complex pages (blogs with sidebars, comments)**:

| Tool | Content Quality | Boilerplate | Speed |
|------|---------------|------------|-------|
| trafilatura | 92% | 5% | Very Fast |
| Firecrawl | 94% | 4% | Medium |
| Crawl4AI | 88% | 8% | Medium |
| Beautiful Soup | 60% | 25% | Fast |

---

## Speed Benchmarks

```python
import time
import asyncio
from dataclasses import dataclass


@dataclass
class SpeedResult:
    tool: str
    urls_tested: int
    total_seconds: float
    avg_seconds_per_page: float
    pages_per_minute: float
    successful: int
    failed: int


def benchmark_scraping_speed(urls: list[str]) -> list[SpeedResult]:
    """Benchmark scraping speed across tools."""
    results = []

    # trafilatura
    try:
        import trafilatura
        start = time.perf_counter()
        success = 0
        for url in urls:
            downloaded = trafilatura.fetch_url(url)
            if downloaded:
                text = trafilatura.extract(downloaded)
                if text:
                    success += 1
        elapsed = time.perf_counter() - start
        results.append(SpeedResult(
            tool="trafilatura",
            urls_tested=len(urls),
            total_seconds=round(elapsed, 2),
            avg_seconds_per_page=round(elapsed / len(urls), 2),
            pages_per_minute=round(len(urls) / elapsed * 60, 1),
            successful=success,
            failed=len(urls) - success,
        ))
    except ImportError:
        pass

    # Crawl4AI (async)
    try:
        from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig

        async def _benchmark_crawl4ai():
            config = BrowserConfig(headless=True)
            run_config = CrawlerRunConfig()

            start = time.perf_counter()
            success = 0

            async with AsyncWebCrawler(config=config) as crawler:
                for url in urls:
                    try:
                        result = await crawler.arun(url=url, config=run_config)
                        if result.success:
                            success += 1
                    except Exception:
                        pass

            elapsed = time.perf_counter() - start
            return elapsed, success

        elapsed, success = asyncio.run(_benchmark_crawl4ai())
        results.append(SpeedResult(
            tool="crawl4ai",
            urls_tested=len(urls),
            total_seconds=round(elapsed, 2),
            avg_seconds_per_page=round(elapsed / len(urls), 2),
            pages_per_minute=round(len(urls) / elapsed * 60, 1),
            successful=success,
            failed=len(urls) - success,
        ))
    except ImportError:
        pass

    return results
```

---

## Anti-Bot and JS Rendering Comparison

| Challenge | Firecrawl | Crawl4AI | Scrapy | Playwright |
|-----------|-----------|----------|--------|-----------|
| Basic JS rendering | Yes | Yes | Via scrapy-playwright | Yes |
| SPA (React/Vue/Next) | Yes | Yes | Via plugin | Yes, with wait |
| Cloudflare protection | Yes | Partial | No | Partial |
| CAPTCHA | Managed | No | No | Manual |
| Rate limit detection | Automatic | Manual | Middleware | Manual |
| Cookie consent dialogs | Handled | Manual | No | Manual |
| Infinite scroll | Yes | Manual | No | Manual |
| Login-protected content | API support | Manual cookies | Middleware | Full control |

---

## Cost Analysis

### Self-Hosted (Local Processing)

| Tool | Infrastructure Cost | Per 10K Pages | Notes |
|------|-------------------|---------------|-------|
| trafilatura | ~$0.50 (CPU only) | ~$0.50 | No browser needed |
| Beautiful Soup | ~$0.50 (CPU only) | ~$0.50 | HTTP library needed |
| Scrapy | ~$1 (CPU) | ~$1 | Needs proper server |
| Crawl4AI | ~$5 (browser memory) | ~$5 | Chromium overhead |
| Playwright | ~$5 (browser memory) | ~$5 | Chromium overhead |

### Managed Services

| Service | Per 10K Pages | Free Tier | Notes |
|---------|-------------|-----------|-------|
| Firecrawl | $10 | 500 pages | Full managed |
| ScrapingBee | $49/10K credits | 1000 credits | Proxy + rendering |
| Apify | ~$5-25 | $5/mo free | Per compute unit |

---

## Decision Framework

```
What type of site are you scraping?
  |
  +---> Static HTML (blogs, wikis, old docs)
  |     --> trafilatura (fastest, best content extraction, free)
  |
  +---> JavaScript-rendered (Docusaurus, Next.js, GitBook)
  |     |
  |     +---> Budget available
  |     |     --> Firecrawl (easiest, best quality)
  |     |
  |     +---> Self-hosted required
  |           --> Crawl4AI (open-source, async)
  |
  +---> Large-scale crawl (1000+ pages)
  |     |
  |     +---> Static site
  |     |     --> Scrapy (best crawl control)
  |     |
  |     +---> JS-rendered site
  |           --> Scrapy + scrapy-playwright OR Firecrawl crawl API
  |
  +---> Anti-bot protected
  |     --> Firecrawl (managed anti-bot)
  |
  +---> Single pages / ad-hoc
        --> trafilatura for static, Playwright for JS
```

---

## Common Pitfalls

1. **Using HTTP-only tools on JS-rendered sites.** trafilatura and Beautiful Soup get empty or partial content from React/Next.js sites.
2. **Not removing boilerplate before chunking.** Navigation menus, footers, and sidebars repeated on every page create thousands of near-identical chunks.
3. **Crawling without depth limits.** Documentation sites can have thousands of pages. Set explicit page count and depth limits.
4. **Ignoring robots.txt.** Besides being unethical, violating robots.txt can result in IP bans and legal issues.
5. **Not deduplicating content.** Many sites have the same content at multiple URLs (with/without trailing slash, query parameters, etc.).
6. **Choosing by speed alone.** The fastest tool (trafilatura) is useless if the site requires JS rendering.

---

## References

- Firecrawl documentation -- https://docs.firecrawl.dev/
- Crawl4AI documentation -- https://docs.crawl4ai.com/
- trafilatura -- https://trafilatura.readthedocs.io/
- Scrapy documentation -- https://docs.scrapy.org/
- Playwright documentation -- https://playwright.dev/python/
