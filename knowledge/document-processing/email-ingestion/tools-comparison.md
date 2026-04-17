# Email Ingestion -- Tools Comparison

## Overview / TL;DR

Email ingestion tools differ in format support, thread handling, attachment processing, and scalability. This guide compares Python's email module, extract-msg, the Gmail API, and IMAP-based approaches across format coverage, metadata extraction, threading capability, and processing speed, with a decision framework for choosing the right approach.

---

## Feature Matrix

| Feature | Python email | extract-msg | Gmail API | IMAP (imaplib) | Unstructured |
|---------|-------------|-------------|-----------|----------------|-------------|
| **EML support** | Excellent | No | N/A | Via fetch | Yes |
| **MSG support** | No | Excellent | N/A | N/A | Yes |
| **MBOX support** | Via mailbox | No | N/A | N/A | No |
| **PST support** | No | No | N/A | N/A | No |
| **Header parsing** | Full | Full | Full | Full | Partial |
| **Body text** | Full MIME | Direct access | Base64 decode | MIME | Auto |
| **Body HTML** | Full MIME | Direct access | Base64 decode | MIME | Auto |
| **Attachments** | Binary access | Binary access | API download | Binary | Recursive |
| **Thread detection** | Via headers | Via headers | threadId | Via headers | No |
| **Character encoding** | Manual | Auto | UTF-8 | Manual | Auto |
| **Batch processing** | Local files | Local files | API (paginated) | Server-side | Local files |
| **Authentication** | None | None | OAuth 2.0 | Password/OAuth | None |
| **Speed (1K emails)** | ~2s | ~5s | ~30s (API) | ~20s (network) | ~10s |
| **Dependencies** | stdlib | extract-msg | google packages | stdlib | unstructured |
| **License** | stdlib | GPL-3.0 | N/A | stdlib | Apache-2.0 |

---

## Format Support Details

### EML (RFC 5322)

The standard internet email format. Supported natively by Python's email module.

```python
def compare_eml_parsers(eml_path: str) -> dict:
    """Compare EML parsing approaches."""
    import time

    # stdlib email
    start = time.perf_counter()
    with open(eml_path, "rb") as f:
        import email
        from email import policy
        msg = email.message_from_binary_file(f, policy=policy.default)
    stdlib_time = time.perf_counter() - start

    # Unstructured
    try:
        from unstructured.partition.email import partition_email
        start = time.perf_counter()
        elements = partition_email(filename=eml_path)
        unst_time = time.perf_counter() - start
    except ImportError:
        elements = []
        unst_time = 0

    return {
        "stdlib": {
            "subject": msg.get("Subject", ""),
            "body_length": len(msg.get_body(preferencelist=("plain",)).get_content() if msg.get_body(preferencelist=("plain",)) else ""),
            "time_ms": round(stdlib_time * 1000, 2),
        },
        "unstructured": {
            "elements": len(elements),
            "time_ms": round(unst_time * 1000, 2),
        },
    }
```

### MSG (Microsoft Outlook)

Proprietary binary format used by Outlook. Requires extract-msg or pypff.

| Feature | extract-msg | Unstructured | Outlook COM |
|---------|------------|-------------|-------------|
| Body text | Direct | Auto-detected | Full |
| HTML body | Direct | Auto-detected | Full |
| Attachments | Binary | Recursive processing | Full |
| Embedded messages | Yes | Partial | Yes |
| Custom properties | Yes | No | Yes |
| Platform | Cross-platform | Cross-platform | Windows only |

### MBOX

Unix mailbox format used by Gmail Takeout and many mail clients.

```python
def benchmark_mbox_parsing(mbox_path: str) -> dict:
    """Benchmark MBOX parsing speed."""
    import mailbox
    import time

    start = time.perf_counter()
    mbox = mailbox.mbox(mbox_path)
    count = 0
    total_chars = 0

    for msg in mbox:
        count += 1
        body = msg.get_payload(decode=True)
        if body:
            total_chars += len(body)

    elapsed = time.perf_counter() - start

    return {
        "messages": count,
        "total_chars": total_chars,
        "time_seconds": round(elapsed, 2),
        "messages_per_second": round(count / max(elapsed, 0.01), 1),
    }
```

---

## Thread Handling Comparison

| Approach | Thread Detection | Quality | Effort |
|----------|-----------------|---------|--------|
| Message-ID + In-Reply-To | Header-based | Good | Low |
| References header | Full chain | Better | Medium |
| Subject grouping | "Re:" prefix matching | Unreliable | Low |
| Gmail threadId | API-provided | Excellent | Low (Gmail only) |
| JWZ threading algorithm | RFC-compliant | Best | High |
| Content similarity | NLP-based | Good | High |

```python
def compare_threading_methods(emails: list[ParsedEmail]) -> dict:
    """Compare thread detection methods."""
    # Method 1: In-Reply-To only
    irt_threads = {}
    for e in emails:
        root = e.in_reply_to or e.message_id
        if root not in irt_threads:
            irt_threads[root] = []
        irt_threads[root].append(e.message_id)

    # Method 2: References header
    ref_threads = {}
    for e in emails:
        root = e.references[0] if e.references else e.message_id
        if root not in ref_threads:
            ref_threads[root] = []
        ref_threads[root].append(e.message_id)

    # Method 3: Subject grouping
    subj_threads = {}
    for e in emails:
        clean_subj = re.sub(r'^(Re|Fwd|Fw):\s*', '', e.subject, flags=re.IGNORECASE).strip()
        if clean_subj not in subj_threads:
            subj_threads[clean_subj] = []
        subj_threads[clean_subj].append(e.message_id)

    return {
        "in_reply_to": {"threads": len(irt_threads), "method": "header"},
        "references": {"threads": len(ref_threads), "method": "header"},
        "subject": {"threads": len(subj_threads), "method": "heuristic"},
    }
```

---

## Cost and Scalability

| Approach | Cost per 100K Emails | Scalability | Notes |
|----------|---------------------|-------------|-------|
| Local EML parsing | ~$0.01 (compute) | Excellent | I/O bound |
| Local MSG parsing | ~$0.05 (compute) | Good | CPU bound |
| Local MBOX parsing | ~$0.01 (compute) | Excellent | I/O bound |
| Gmail API | Free (quota limits) | API-limited | 250 quota units/user/s |
| IMAP fetch | Free (server-limited) | Network-bound | Depends on server |
| Microsoft Graph API | Free (throttled) | API-limited | 10K requests/10min |

---

## Decision Framework

```
What is your email source?
  |
  +---> Local .eml files
  |     --> Python email module (stdlib, fast, no dependencies)
  |
  +---> Outlook .msg files
  |     --> extract-msg (cross-platform, good coverage)
  |
  +---> MBOX archive (Gmail Takeout)
  |     --> Python mailbox module (stdlib, handles large archives)
  |
  +---> Live Gmail account
  |     --> Gmail API (OAuth, threadId support, real-time)
  |
  +---> Live IMAP server
  |     --> imaplib (stdlib, server-side search, any provider)
  |
  +---> Mixed formats
        --> Unstructured (auto-detect, handles EML + MSG)
```

---

## Common Pitfalls

1. **Using only In-Reply-To for threading.** The References header provides the full chain. In-Reply-To only points to the immediate parent.
2. **Not handling charset encoding.** Emails use dozens of character encodings. Always specify `errors="replace"`.
3. **Parsing MSG files with the email module.** MSG is a proprietary binary format, not RFC 5322. Use extract-msg.
4. **Hitting Gmail API quota limits.** The Gmail API has strict rate limits. Implement exponential backoff.
5. **Not testing with real email data.** Synthetic test emails do not have the encoding issues, malformed headers, and edge cases of real mailboxes.

---

## References

- Python email module -- https://docs.python.org/3/library/email.html
- extract-msg -- https://github.com/TeamMsgExtractor/msg-extractor
- Gmail API -- https://developers.google.com/gmail/api
- IMAP RFC -- https://tools.ietf.org/html/rfc3501
- JWZ threading algorithm -- https://www.jwz.org/doc/threading.html
