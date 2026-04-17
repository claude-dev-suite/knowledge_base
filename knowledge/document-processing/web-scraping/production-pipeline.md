# Web Scraping -- Production Pipeline

## Overview / TL;DR

A production web scraping pipeline for RAG must discover pages, render JavaScript content, extract clean text, handle failures and rate limits, deduplicate content, chunk for embedding, and support incremental re-crawling. This guide provides a complete async pipeline implementation with configurable crawling, content extraction, cleaning, chunking, and integration with vector stores.

---

## Architecture

```
Seed URLs / Sitemap
    |
    v
[1] URL Discovery
    |-- sitemap.xml parsing
    |-- link following (BFS with depth limit)
    |-- URL normalization and dedup
    |
    v
[2] Page Fetching
    |-- static pages: HTTP + trafilatura
    |-- JS pages: Playwright/Crawl4AI
    |-- rate limiting + retry
    |
    v
[3] Content Extraction
    |-- main content detection
    |-- boilerplate removal
    |-- markdown conversion
    |
    v
[4] Cleaning & Dedup
    |-- navigation/footer removal
    |-- cross-page dedup (simhash)
    |-- content quality check
    |
    v
[5] Chunking
    |-- markdown-aware chunking
    |-- heading hierarchy metadata
    |
    v
[6] Output (vector store / JSON)
```

---

## Pipeline Configuration

```python
from dataclasses import dataclass, field


@dataclass
class CrawlConfig:
    seed_urls: list[str] = field(default_factory=list)
    max_pages: int = 500
    max_depth: int = 5
    allowed_domains: list[str] = field(default_factory=list)
    include_paths: list[str] = field(default_factory=list)
    exclude_paths: list[str] = field(default_factory=list)
    respect_robots_txt: bool = True
    delay_seconds: float = 1.0
    max_concurrent: int = 5
    timeout_seconds: int = 30
    js_rendering: bool = True
    retry_count: int = 2
    user_agent: str = "RAGBot/1.0 (compatible; +https://example.com/bot)"


@dataclass
class ExtractionConfig:
    remove_navigation: bool = True
    remove_footer: bool = True
    remove_sidebar: bool = True
    extract_tables: bool = True
    extract_code_blocks: bool = True
    min_content_length: int = 100
    max_content_length: int = 100000
    output_format: str = "markdown"


@dataclass
class ChunkConfig:
    chunk_size: int = 1000
    chunk_overlap: int = 200
    respect_headings: bool = True
    min_chunk_size: int = 100
```

---

## URL Discovery

```python
import re
import logging
from urllib.parse import urlparse, urljoin, urldefrag
from collections import deque
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class URLQueue:
    seen: set = field(default_factory=set)
    queue: deque = field(default_factory=deque)
    domain_filter: set = field(default_factory=set)


def discover_from_sitemap(base_url: str) -> list[str]:
    """Extract URLs from sitemap.xml."""
    import requests

    sitemap_urls = [
        f"{base_url}/sitemap.xml",
        f"{base_url}/sitemap_index.xml",
    ]

    urls = []
    for sitemap_url in sitemap_urls:
        try:
            resp = requests.get(sitemap_url, timeout=10)
            if resp.status_code == 200:
                found = re.findall(r'<loc>(.*?)</loc>', resp.text)
                urls.extend(found)
        except Exception:
            continue

    return urls


def normalize_url(url: str) -> str:
    """Normalize a URL for deduplication."""
    url, _ = urldefrag(url)
    parsed = urlparse(url)
    # Remove trailing slash
    path = parsed.path.rstrip("/") or "/"
    # Remove common tracking parameters
    return f"{parsed.scheme}://{parsed.netloc}{path}"


def should_crawl(url: str, config: CrawlConfig) -> bool:
    """Check if a URL should be crawled based on config."""
    parsed = urlparse(url)

    # Domain check
    if config.allowed_domains:
        if parsed.netloc not in config.allowed_domains:
            return False

    # Path include/exclude
    path = parsed.path
    if config.include_paths:
        if not any(path.startswith(p) for p in config.include_paths):
            return False
    if config.exclude_paths:
        if any(path.startswith(p) for p in config.exclude_paths):
            return False

    # Skip non-HTML resources
    skip_extensions = {".pdf", ".png", ".jpg", ".gif", ".css", ".js", ".svg", ".zip"}
    if any(path.lower().endswith(ext) for ext in skip_extensions):
        return False

    return True
```

---

## Async Crawling Engine

```python
import asyncio
import logging
import time
from dataclasses import dataclass, field
from pathlib import Path

logger = logging.getLogger(__name__)


@dataclass
class ScrapedPage:
    url: str
    title: str
    markdown: str
    html: str = ""
    status_code: int = 200
    content_length: int = 0
    fetch_seconds: float = 0.0
    error: str | None = None


class WebScrapingPipeline:
    """Async web scraping pipeline for RAG."""

    def __init__(
        self,
        crawl_config: CrawlConfig,
        extraction_config: ExtractionConfig,
        chunk_config: ChunkConfig,
    ):
        self.crawl_config = crawl_config
        self.extraction_config = extraction_config
        self.chunk_config = chunk_config

    async def crawl(self) -> list[ScrapedPage]:
        """Crawl all pages starting from seed URLs."""
        url_queue = URLQueue(
            domain_filter=set(self.crawl_config.allowed_domains),
        )

        # Discover initial URLs
        for seed in self.crawl_config.seed_urls:
            sitemap_urls = discover_from_sitemap(seed)
            for url in sitemap_urls:
                norm = normalize_url(url)
                if should_crawl(norm, self.crawl_config) and norm not in url_queue.seen:
                    url_queue.seen.add(norm)
                    url_queue.queue.append((norm, 0))

            norm = normalize_url(seed)
            if norm not in url_queue.seen:
                url_queue.seen.add(norm)
                url_queue.queue.append((norm, 0))

        logger.info(f"Starting crawl with {len(url_queue.queue)} URLs")

        results = []
        semaphore = asyncio.Semaphore(self.crawl_config.max_concurrent)

        while url_queue.queue and len(results) < self.crawl_config.max_pages:
            url, depth = url_queue.queue.popleft()

            if depth > self.crawl_config.max_depth:
                continue

            async with semaphore:
                page = await self._fetch_page(url)

                if page.error:
                    logger.debug(f"Failed: {url} - {page.error}")
                    continue

                if page.content_length < self.extraction_config.min_content_length:
                    continue

                results.append(page)
                logger.info(f"[{len(results)}/{self.crawl_config.max_pages}] {url}")

                # Extract and queue new links
                if depth < self.crawl_config.max_depth:
                    links = self._extract_links(page.html, url)
                    for link in links:
                        norm = normalize_url(link)
                        if norm not in url_queue.seen and should_crawl(norm, self.crawl_config):
                            url_queue.seen.add(norm)
                            url_queue.queue.append((norm, depth + 1))

                # Rate limiting
                await asyncio.sleep(self.crawl_config.delay_seconds)

        logger.info(f"Crawl complete: {len(results)} pages")
        return results

    async def _fetch_page(self, url: str) -> ScrapedPage:
        """Fetch and extract content from a single page."""
        start = time.perf_counter()

        for attempt in range(self.crawl_config.retry_count + 1):
            try:
                if self.crawl_config.js_rendering:
                    return await self._fetch_with_browser(url, start)
                else:
                    return await self._fetch_static(url, start)
            except Exception as e:
                if attempt < self.crawl_config.retry_count:
                    await asyncio.sleep(2 ** attempt)
                    continue
                return ScrapedPage(
                    url=url,
                    title="",
                    markdown="",
                    error=str(e),
                    fetch_seconds=time.perf_counter() - start,
                )

        return ScrapedPage(url=url, title="", markdown="", error="Max retries exceeded")

    async def _fetch_static(self, url: str, start: float) -> ScrapedPage:
        """Fetch static page using HTTP + trafilatura."""
        import trafilatura

        loop = asyncio.get_event_loop()
        downloaded = await loop.run_in_executor(
            None, trafilatura.fetch_url, url,
        )

        if not downloaded:
            return ScrapedPage(url=url, title="", markdown="", error="Fetch failed")

        text = await loop.run_in_executor(
            None,
            lambda: trafilatura.extract(
                downloaded,
                output_format="txt",
                include_tables=True,
                include_links=False,
                favor_precision=True,
            ),
        )

        return ScrapedPage(
            url=url,
            title="",
            markdown=text or "",
            html=downloaded,
            content_length=len(text or ""),
            fetch_seconds=time.perf_counter() - start,
        )

    async def _fetch_with_browser(self, url: str, start: float) -> ScrapedPage:
        """Fetch JS-rendered page using Crawl4AI."""
        from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig

        browser_config = BrowserConfig(headless=True)
        run_config = CrawlerRunConfig(
            excluded_tags=["nav", "footer", "aside", "header"],
            remove_overlay_elements=True,
        )

        async with AsyncWebCrawler(config=browser_config) as crawler:
            result = await crawler.arun(url=url, config=run_config)

            if not result.success:
                return ScrapedPage(
                    url=url, title="", markdown="",
                    error=f"Status {result.status_code}",
                )

            markdown = ""
            if result.markdown_v2:
                markdown = result.markdown_v2.raw_markdown

            return ScrapedPage(
                url=url,
                title=result.metadata.get("title", "") if result.metadata else "",
                markdown=markdown,
                html=result.cleaned_html or "",
                status_code=result.status_code,
                content_length=len(markdown),
                fetch_seconds=time.perf_counter() - start,
            )

    def _extract_links(self, html: str, base_url: str) -> list[str]:
        """Extract links from HTML content."""
        links = []
        for match in re.finditer(r'href=["\']([^"\']+)["\']', html):
            href = match.group(1)
            if href.startswith(("#", "mailto:", "javascript:", "tel:")):
                continue
            absolute = urljoin(base_url, href)
            links.append(absolute)
        return links
```

---

## Content Deduplication

```python
import hashlib


def simhash(text: str, num_bits: int = 64) -> int:
    """Compute simhash for near-duplicate detection."""
    words = text.lower().split()
    v = [0] * num_bits

    for word in words:
        h = int(hashlib.md5(word.encode()).hexdigest(), 16)
        for i in range(num_bits):
            if h & (1 << i):
                v[i] += 1
            else:
                v[i] -= 1

    fingerprint = 0
    for i in range(num_bits):
        if v[i] > 0:
            fingerprint |= (1 << i)

    return fingerprint


def hamming_distance(a: int, b: int) -> int:
    """Count differing bits between two simhashes."""
    return bin(a ^ b).count("1")


def deduplicate_pages(
    pages: list[ScrapedPage],
    similarity_threshold: int = 5,
) -> list[ScrapedPage]:
    """Remove near-duplicate pages using simhash."""
    unique = []
    hashes = []

    for page in pages:
        if not page.markdown.strip():
            continue

        h = simhash(page.markdown)
        is_dup = False

        for existing_hash in hashes:
            if hamming_distance(h, existing_hash) <= similarity_threshold:
                is_dup = True
                break

        if not is_dup:
            unique.append(page)
            hashes.append(h)

    removed = len(pages) - len(unique)
    if removed > 0:
        logger.info(f"Deduplicated: removed {removed} near-duplicate pages")

    return unique
```

---

## Chunking and Output

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter


def chunk_pages(
    pages: list[ScrapedPage],
    config: ChunkConfig,
) -> list[dict]:
    """Chunk scraped pages for vector store indexing."""
    all_chunks = []

    # Markdown header splitter
    headers_to_split = [
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ]
    md_splitter = MarkdownHeaderTextSplitter(
        headers_to_split_on=headers_to_split,
        strip_headers=False,
    )

    # Size splitter for long sections
    size_splitter = RecursiveCharacterTextSplitter(
        chunk_size=config.chunk_size,
        chunk_overlap=config.chunk_overlap,
    )

    for page in pages:
        if not page.markdown.strip():
            continue

        # First split by headers
        header_chunks = md_splitter.split_text(page.markdown)

        # Then split long sections by size
        final_chunks = size_splitter.split_documents(header_chunks)

        for i, chunk in enumerate(final_chunks):
            # Skip tiny chunks
            if len(chunk.page_content.strip()) < config.min_chunk_size:
                continue

            all_chunks.append({
                "text": chunk.page_content,
                "metadata": {
                    "source_url": page.url,
                    "title": page.title,
                    "chunk_index": i,
                    "total_chunks": len(final_chunks),
                    **chunk.metadata,
                },
            })

    return all_chunks
```

---

## Running the Pipeline

```python
import asyncio
import json
from pathlib import Path


async def main():
    crawl_config = CrawlConfig(
        seed_urls=["https://docs.example.com/"],
        allowed_domains=["docs.example.com"],
        max_pages=200,
        max_depth=4,
        delay_seconds=1.0,
        js_rendering=True,
    )

    extraction_config = ExtractionConfig(
        min_content_length=100,
        output_format="markdown",
    )

    chunk_config = ChunkConfig(
        chunk_size=1000,
        chunk_overlap=200,
        min_chunk_size=100,
    )

    pipeline = WebScrapingPipeline(crawl_config, extraction_config, chunk_config)

    # Crawl
    pages = await pipeline.crawl()

    # Deduplicate
    pages = deduplicate_pages(pages)

    # Chunk
    chunks = chunk_pages(pages, chunk_config)

    # Save
    output_dir = Path("./output")
    output_dir.mkdir(exist_ok=True)

    with open(output_dir / "chunks.json", "w") as f:
        json.dump(chunks, f, indent=2, default=str)

    print(f"Scraped {len(pages)} pages, produced {len(chunks)} chunks")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Common Pitfalls

1. **Not deduplicating before chunking.** Shared boilerplate across pages creates thousands of near-identical chunks.
2. **Crawling too aggressively.** Fast crawling without rate limits can get your IP blocked or trigger DDoS protections.
3. **Not handling pagination and infinite scroll.** Some sites load content dynamically. Set hard page count limits.
4. **Missing error recovery.** Transient network errors are common. Implement retries with exponential backoff.
5. **Not tracking source URLs.** Every chunk needs its source URL for citations. Losing this metadata makes the RAG system uncitable.
6. **Using browser rendering for static sites.** Playwright/Crawl4AI are 10x slower than HTTP fetching. Only use them when JS rendering is needed.

---

## Production Checklist

- [ ] Seed URLs and allowed domains are configured
- [ ] Sitemap.xml is parsed for initial URL discovery
- [ ] Rate limiting and robots.txt compliance are enforced
- [ ] JS rendering is used only for sites that need it
- [ ] Content extraction strips navigation, footer, and sidebar
- [ ] Near-duplicate pages are removed before chunking
- [ ] Chunking respects heading structure
- [ ] Source URL is preserved in chunk metadata
- [ ] Retry logic handles transient failures
- [ ] Crawl depth and page count limits are set

---

## References

- Firecrawl documentation -- https://docs.firecrawl.dev/
- Crawl4AI documentation -- https://docs.crawl4ai.com/
- trafilatura documentation -- https://trafilatura.readthedocs.io/
- LangChain text splitters -- https://python.langchain.com/docs/how_to/#text-splitters
