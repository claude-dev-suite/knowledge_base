# Web Scraping for RAG -- Comprehensive Guide

## Overview / TL;DR

Web scraping for RAG pipelines converts live web pages into clean, structured text suitable for chunking, embedding, and retrieval. Unlike traditional scraping (which extracts specific data points), RAG-oriented scraping aims to capture the full semantic content of a page -- prose, headings, tables, code blocks, and lists -- while stripping navigation, ads, boilerplate, and script tags. This guide covers every major scraping tool (Firecrawl, Crawl4AI, trafilatura, Scrapy, Playwright, Beautiful Soup), explains when each excels, and provides production-ready Python code for building crawl-to-index pipelines.

---

## Why Web Scraping for RAG Is Different

Traditional scraping extracts structured fields (price, title, date). RAG scraping extracts the full readable content of a page for LLM consumption. Key differences:

1. **Content extraction, not data extraction.** You want the article body, not just the title field.
2. **Markdown output preferred.** Clean markdown preserves headings, lists, and code blocks better than raw HTML for embedding.
3. **JavaScript rendering often required.** Modern SPAs and documentation sites render content client-side.
4. **Crawl depth matters.** Documentation sites may have hundreds of pages linked from a root.
5. **Deduplication is critical.** Navigation menus, sidebars, and footers repeat across pages.
6. **Rate limiting and politeness.** Aggressive crawling can get you blocked or cause outages.

---

## Tool Landscape

### Firecrawl

Managed API for scraping websites and converting to clean markdown. Handles JavaScript rendering, anti-bot protections, and rate limiting.

```python
from firecrawl import FirecrawlApp


def scrape_single_page(url: str, api_key: str) -> dict:
    """Scrape a single URL using Firecrawl."""
    app = FirecrawlApp(api_key=api_key)

    result = app.scrape_url(
        url,
        params={
            "formats": ["markdown", "html"],
            "onlyMainContent": True,     # Strip nav, footer, sidebar
            "waitFor": 3000,             # Wait for JS rendering (ms)
            "timeout": 30000,
        },
    )

    return {
        "url": url,
        "markdown": result.get("markdown", ""),
        "html": result.get("html", ""),
        "metadata": result.get("metadata", {}),
        "title": result.get("metadata", {}).get("title", ""),
    }


def crawl_site(
    start_url: str,
    api_key: str,
    max_pages: int = 100,
    include_paths: list[str] = None,
    exclude_paths: list[str] = None,
) -> list[dict]:
    """Crawl an entire site starting from a URL."""
    app = FirecrawlApp(api_key=api_key)

    params = {
        "limit": max_pages,
        "scrapeOptions": {
            "formats": ["markdown"],
            "onlyMainContent": True,
        },
    }

    if include_paths:
        params["includePaths"] = include_paths
    if exclude_paths:
        params["excludePaths"] = exclude_paths

    result = app.crawl_url(start_url, params=params, poll_interval=5)

    pages = []
    for page in result.get("data", []):
        pages.append({
            "url": page.get("metadata", {}).get("sourceURL", ""),
            "markdown": page.get("markdown", ""),
            "title": page.get("metadata", {}).get("title", ""),
        })

    return pages
```

**Strengths**: Managed infrastructure, JS rendering, anti-bot handling, clean markdown output, site crawling. **Weaknesses**: Cloud-only, per-page pricing ($1/1000 pages), rate limits.

### Crawl4AI

Open-source async crawling library optimized for LLM data preparation.

```python
from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig
from crawl4ai.extraction_strategy import LLMExtractionStrategy
import asyncio


async def scrape_with_crawl4ai(url: str) -> dict:
    """Scrape a URL using Crawl4AI."""
    browser_config = BrowserConfig(
        headless=True,
        java_script_enabled=True,
    )

    run_config = CrawlerRunConfig(
        wait_for="css:main",     # Wait for main content to load
        word_count_threshold=10, # Skip pages with < 10 words
        excluded_tags=["nav", "footer", "aside", "header"],
        remove_overlay_elements=True,
    )

    async with AsyncWebCrawler(config=browser_config) as crawler:
        result = await crawler.arun(url=url, config=run_config)

        return {
            "url": url,
            "markdown": result.markdown_v2.raw_markdown if result.markdown_v2 else "",
            "cleaned_html": result.cleaned_html or "",
            "success": result.success,
            "status_code": result.status_code,
            "title": result.metadata.get("title", "") if result.metadata else "",
        }


async def crawl_site_crawl4ai(
    start_url: str,
    max_pages: int = 100,
) -> list[dict]:
    """Crawl a site with link following."""
    visited = set()
    results = []
    queue = [start_url]

    browser_config = BrowserConfig(headless=True)
    run_config = CrawlerRunConfig(
        word_count_threshold=10,
        excluded_tags=["nav", "footer", "aside"],
    )

    async with AsyncWebCrawler(config=browser_config) as crawler:
        while queue and len(visited) < max_pages:
            url = queue.pop(0)
            if url in visited:
                continue
            visited.add(url)

            try:
                result = await crawler.arun(url=url, config=run_config)
                if result.success:
                    results.append({
                        "url": url,
                        "markdown": result.markdown_v2.raw_markdown if result.markdown_v2 else "",
                        "title": result.metadata.get("title", "") if result.metadata else "",
                    })

                    # Extract links from same domain
                    if result.links:
                        from urllib.parse import urlparse
                        base_domain = urlparse(start_url).netloc
                        for link in result.links.get("internal", []):
                            href = link.get("href", "")
                            if href and urlparse(href).netloc == base_domain:
                                if href not in visited:
                                    queue.append(href)
            except Exception as e:
                continue

    return results


# Run async function
# pages = asyncio.run(crawl_site_crawl4ai("https://docs.example.com"))
```

**Strengths**: Open-source, async, JS rendering via Playwright, built-in content extraction, LLM-optimized markdown. **Weaknesses**: Requires Playwright/Chromium, heavier than HTTP-only tools.

### trafilatura

Python library focused on extracting the main text content from web pages. Rule-based, no browser needed.

```python
import trafilatura
from trafilatura import fetch_url, extract
from trafilatura.settings import use_config


def scrape_with_trafilatura(url: str) -> dict:
    """Extract main content from a URL using trafilatura."""
    config = use_config()
    config.set("DEFAULT", "EXTRACTION_TIMEOUT", "30")

    downloaded = fetch_url(url)
    if downloaded is None:
        return {"url": url, "error": "Failed to fetch URL"}

    # Extract as markdown
    text = extract(
        downloaded,
        output_format="txt",
        include_comments=False,
        include_tables=True,
        include_links=True,
        include_images=False,
        favor_precision=True,     # Prefer precision over recall
        deduplicate=True,
    )

    # Also get metadata
    metadata = trafilatura.extract(
        downloaded,
        output_format="json",
        with_metadata=True,
    )

    return {
        "url": url,
        "text": text or "",
        "metadata_json": metadata,
    }


def batch_scrape_trafilatura(urls: list[str]) -> list[dict]:
    """Batch scrape multiple URLs."""
    results = []
    for url in urls:
        result = scrape_with_trafilatura(url)
        results.append(result)
    return results
```

**Strengths**: Lightweight (no browser), fast, excellent boilerplate removal, good for articles and blog posts. **Weaknesses**: No JavaScript rendering, misses dynamically loaded content, no crawling support.

### Scrapy + Playwright

Industrial-strength crawling framework with optional Playwright integration for JS rendering.

```python
# scrapy_spider.py
import scrapy
from scrapy import Request


class DocumentationSpider(scrapy.Spider):
    """Scrapy spider for crawling documentation sites."""
    name = "docs_spider"
    allowed_domains = ["docs.example.com"]
    start_urls = ["https://docs.example.com/"]

    custom_settings = {
        "CONCURRENT_REQUESTS": 5,
        "DOWNLOAD_DELAY": 1.0,
        "ROBOTSTXT_OBEY": True,
        "DEPTH_LIMIT": 5,
        "CLOSESPIDER_PAGECOUNT": 500,
    }

    def parse(self, response):
        # Extract main content
        title = response.css("title::text").get("")
        # Remove nav, footer, sidebar
        content_selectors = [
            "article", "main", ".content", ".documentation",
            "#content", ".markdown-body",
        ]

        content_html = ""
        for selector in content_selectors:
            elements = response.css(selector)
            if elements:
                content_html = elements.get()
                break

        if not content_html:
            content_html = response.css("body").get("")

        # Convert to text
        from trafilatura import extract
        text = extract(content_html, output_format="txt") or ""

        yield {
            "url": response.url,
            "title": title,
            "text": text,
            "html": content_html,
        }

        # Follow links within the documentation
        for link in response.css("a::attr(href)").getall():
            if link and not link.startswith(("#", "mailto:", "javascript:")):
                yield response.follow(link, callback=self.parse)
```

### Playwright (Direct)

```python
from playwright.async_api import async_playwright
import asyncio


async def scrape_with_playwright(url: str) -> dict:
    """Scrape a JavaScript-rendered page using Playwright."""
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()

        try:
            await page.goto(url, wait_until="networkidle", timeout=30000)

            # Wait for content to load
            await page.wait_for_selector("main, article, .content", timeout=10000)

            # Remove unwanted elements
            await page.evaluate("""
                document.querySelectorAll('nav, footer, aside, .sidebar, .cookie-banner')
                    .forEach(el => el.remove());
            """)

            # Get text content
            text = await page.inner_text("body")
            html = await page.inner_html("body")
            title = await page.title()

            return {
                "url": url,
                "text": text,
                "html": html,
                "title": title,
            }
        finally:
            await browser.close()
```

---

## Quick Comparison

| Feature | Firecrawl | Crawl4AI | trafilatura | Scrapy | Playwright |
|---------|-----------|----------|------------|--------|-----------|
| JS rendering | Yes | Yes | No | Plugin | Yes |
| Markdown output | Built-in | Built-in | Basic | Manual | Manual |
| Boilerplate removal | Excellent | Good | Excellent | Manual | Manual |
| Crawling | Built-in | Built-in | No | Excellent | Manual |
| Anti-bot handling | Yes | Basic | No | Middleware | Basic |
| Async | Yes | Yes | No | Yes | Yes |
| Rate limiting | Built-in | Manual | Manual | Built-in | Manual |
| Local/Cloud | Cloud | Local | Local | Local | Local |
| Cost | $1/1K pages | Free | Free | Free | Free |
| Setup effort | Minimal | Medium | Minimal | High | Medium |

---

## Content Cleaning and Normalization

```python
import re


def clean_scraped_markdown(markdown: str) -> str:
    """Clean scraped markdown for RAG ingestion."""
    # Remove excessive blank lines
    markdown = re.sub(r'\n{4,}', '\n\n\n', markdown)

    # Remove navigation remnants
    nav_patterns = [
        r'^\s*Skip to .*$',
        r'^\s*Previous\s*Next\s*$',
        r'^\s*On this page.*$',
        r'^\s*Table of contents.*$',
        r'^\s*\[edit\].*$',
        r'^\s*Was this page helpful\?.*$',
    ]
    for pattern in nav_patterns:
        markdown = re.sub(pattern, '', markdown, flags=re.MULTILINE | re.IGNORECASE)

    # Clean up link-heavy sections (usually nav menus)
    lines = markdown.split('\n')
    cleaned = []
    consecutive_links = 0
    for line in lines:
        link_count = len(re.findall(r'\[.*?\]\(.*?\)', line))
        if link_count > 3 and len(line.strip()) < link_count * 50:
            consecutive_links += 1
            if consecutive_links > 3:
                continue
        else:
            consecutive_links = 0
        cleaned.append(line)

    markdown = '\n'.join(cleaned)

    # Remove empty headers
    markdown = re.sub(r'^#{1,6}\s*$', '', markdown, flags=re.MULTILINE)

    # Normalize whitespace
    markdown = re.sub(r' {3,}', ' ', markdown)

    return markdown.strip()
```

---

## Common Pitfalls

1. **Not rendering JavaScript.** Many documentation sites (Next.js, Docusaurus, GitBook) render content client-side. HTTP-only tools (trafilatura, requests) get empty pages.
2. **Scraping navigation and boilerplate.** Without content extraction, chunks include menus, footers, and cookie banners that pollute embeddings.
3. **Ignoring robots.txt and rate limits.** Aggressive crawling gets your IP blocked and can disrupt small sites. Always respect robots.txt and add delays.
4. **Not deduplicating across pages.** Shared sidebars, headers, and footers appear in every page's content. Remove them before chunking.
5. **Scraping too deep.** Without depth limits, crawlers can follow infinite pagination, comment threads, or external links.
6. **Not handling encoding correctly.** Pages may use various encodings. Detect charset from HTTP headers or meta tags.
7. **Missing dynamic content behind authentication.** Gated docs require authentication cookies or API keys.
8. **Not tracking source URLs in metadata.** Every chunk needs its source URL for citations and re-crawling.

---

## Production Checklist

- [ ] JavaScript rendering is enabled for SPA/dynamic sites
- [ ] Content extraction strips navigation, sidebars, footers, and ads
- [ ] Crawl depth and page count limits are configured
- [ ] Rate limiting respects target site's robots.txt and capacity
- [ ] Output is clean markdown with preserved heading structure
- [ ] Source URL is tracked in chunk metadata for citation
- [ ] Deduplication removes repeated boilerplate across pages
- [ ] Authentication is handled for gated content
- [ ] Error handling retries transient failures
- [ ] Incremental crawling processes only new/changed pages

---

## References

- Firecrawl documentation -- https://docs.firecrawl.dev/
- Crawl4AI documentation -- https://docs.crawl4ai.com/
- trafilatura documentation -- https://trafilatura.readthedocs.io/
- Scrapy documentation -- https://docs.scrapy.org/
- Playwright for Python -- https://playwright.dev/python/
