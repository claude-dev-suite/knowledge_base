# Email Ingestion -- Production Pipeline

## Overview / TL;DR

A production email ingestion pipeline must parse multiple formats, clean email bodies, reconstruct threads, handle attachments, deduplicate content, and produce chunks with rich metadata for RAG retrieval. This guide provides a complete pipeline from mailbox to indexed chunks with thread-aware chunking, attachment processing, and deduplication.

---

## Architecture

```
Email Source (EML/MSG/MBOX/API)
    |
    v
[1] Parsing (format-specific)
    |-- EML: Python email module
    |-- MSG: extract-msg
    |-- MBOX: mailbox module
    |
    v
[2] Body Cleaning
    |-- HTML to text
    |-- Remove quoted replies
    |-- Remove signatures
    |-- Normalize whitespace
    |
    v
[3] Thread Reconstruction
    |-- Build thread trees
    |-- Order by date
    |-- Merge context
    |
    v
[4] Attachment Processing
    |-- Extract to temp files
    |-- Route to format-specific processor
    |-- Link back to parent email
    |
    v
[5] Deduplication
    |-- Content hash dedup (exact)
    |-- Simhash dedup (near-duplicate)
    |-- Forwarded content detection
    |
    v
[6] Chunking & Output
    |-- Per-email chunks with metadata
    |-- Thread-aware chunks (conversation context)
    |-- Attachment chunks linked to email
```

---

## Pipeline Implementation

```python
import logging
import hashlib
import time
from dataclasses import dataclass, field
from pathlib import Path
from concurrent.futures import ProcessPoolExecutor, as_completed

logger = logging.getLogger(__name__)


@dataclass
class EmailChunk:
    text: str
    metadata: dict


@dataclass
class PipelineResult:
    total_emails: int = 0
    parsed: int = 0
    failed: int = 0
    threads: int = 0
    chunks: list[EmailChunk] = field(default_factory=list)
    errors: list[dict] = field(default_factory=list)
    processing_seconds: float = 0.0


class EmailIngestionPipeline:
    """Production pipeline for email ingestion into RAG."""

    def __init__(
        self,
        chunk_size: int = 1500,
        process_attachments: bool = True,
        dedup: bool = True,
        max_workers: int = 4,
    ):
        self.chunk_size = chunk_size
        self.process_attachments = process_attachments
        self.dedup = dedup
        self.max_workers = max_workers

    def process_directory(self, dir_path: str) -> PipelineResult:
        """Process all email files in a directory."""
        start = time.perf_counter()
        path = Path(dir_path)

        # Discover email files
        eml_files = list(path.rglob("*.eml"))
        msg_files = list(path.rglob("*.msg"))
        mbox_files = list(path.rglob("*.mbox"))

        all_emails = []

        # Parse EML files
        for f in eml_files:
            try:
                parsed = parse_eml(str(f))
                parsed.headers["_source_file"] = str(f)
                all_emails.append(parsed)
            except Exception as e:
                logger.warning(f"Failed to parse {f}: {e}")

        # Parse MSG files
        for f in msg_files:
            try:
                parsed = parse_msg(str(f))
                parsed.headers["_source_file"] = str(f)
                all_emails.append(parsed)
            except Exception as e:
                logger.warning(f"Failed to parse {f}: {e}")

        # Parse MBOX files
        for f in mbox_files:
            parsed_list = parse_mbox(str(f))
            for p in parsed_list:
                p.headers["_source_file"] = str(f)
            all_emails.extend(parsed_list)

        logger.info(f"Parsed {len(all_emails)} emails from {dir_path}")

        # Process
        return self._process_emails(all_emails, start)

    def _process_emails(
        self,
        emails: list[ParsedEmail],
        start_time: float,
    ) -> PipelineResult:
        """Process parsed emails through the pipeline."""
        result = PipelineResult(total_emails=len(emails))

        # Step 1: Clean bodies
        for email_msg in emails:
            email_msg.body_text = clean_email_body(email_msg)
            result.parsed += 1

        # Step 2: Build threads
        threads = build_threads(emails)
        result.threads = len(threads)

        # Step 3: Deduplication
        if self.dedup:
            emails = self._deduplicate(emails)

        # Step 4: Chunk
        for email_msg in emails:
            chunks = self._chunk_email(email_msg)
            result.chunks.extend(chunks)

        # Step 5: Thread-level chunks
        for thread in threads:
            if len(thread.messages) > 1:
                thread_chunk = self._chunk_thread(thread)
                if thread_chunk:
                    result.chunks.append(thread_chunk)

        result.processing_seconds = time.perf_counter() - start_time
        return result

    def _deduplicate(self, emails: list[ParsedEmail]) -> list[ParsedEmail]:
        """Remove duplicate emails by content hash."""
        seen = set()
        unique = []

        for email_msg in emails:
            content = f"{email_msg.subject}|{email_msg.from_addr}|{email_msg.body_text[:500]}"
            content_hash = hashlib.sha256(content.encode()).hexdigest()

            if content_hash not in seen:
                seen.add(content_hash)
                unique.append(email_msg)

        removed = len(emails) - len(unique)
        if removed > 0:
            logger.info(f"Dedup removed {removed} duplicate emails")

        return unique

    def _chunk_email(self, email_msg: ParsedEmail) -> list[EmailChunk]:
        """Chunk a single email with metadata."""
        body = email_msg.body_text
        if not body or len(body.strip()) < 20:
            return []

        # Build header context
        header = (
            f"Subject: {email_msg.subject}\n"
            f"From: {email_msg.from_addr}\n"
            f"To: {', '.join(email_msg.to_addrs)}\n"
            f"Date: {email_msg.date}\n\n"
        )

        full_text = header + body

        # If small enough, keep as single chunk
        if len(full_text) <= self.chunk_size:
            return [EmailChunk(
                text=full_text,
                metadata=self._email_metadata(email_msg),
            )]

        # Split body into chunks, prepending header to each
        from langchain_text_splitters import RecursiveCharacterTextSplitter
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.chunk_size - len(header),
            chunk_overlap=100,
        )
        body_chunks = splitter.split_text(body)

        return [
            EmailChunk(
                text=header + chunk,
                metadata={
                    **self._email_metadata(email_msg),
                    "chunk_index": i,
                    "total_chunks": len(body_chunks),
                },
            )
            for i, chunk in enumerate(body_chunks)
        ]

    def _chunk_thread(self, thread: EmailThread) -> EmailChunk | None:
        """Create a thread-level chunk summarizing the conversation."""
        if not thread.messages:
            return None

        parts = [f"Thread: {thread.subject}\n"]
        parts.append(f"Participants: {', '.join(thread.participants)}\n")
        parts.append(f"Messages: {len(thread.messages)}\n\n")

        for msg in thread.messages:
            body = msg.body_text[:300] if msg.body_text else ""
            parts.append(f"[{msg.from_addr}] ({msg.date}): {body}\n")

        text = "\n".join(parts)
        if len(text) > self.chunk_size * 2:
            text = text[:self.chunk_size * 2]

        return EmailChunk(
            text=text,
            metadata={
                "thread_id": thread.thread_id,
                "subject": thread.subject,
                "message_count": len(thread.messages),
                "participants": list(thread.participants),
                "date_range_start": thread.date_range[0],
                "date_range_end": thread.date_range[1],
                "chunk_type": "thread_summary",
            },
        )

    def _email_metadata(self, email_msg: ParsedEmail) -> dict:
        """Build metadata dict for an email chunk."""
        return {
            "message_id": email_msg.message_id,
            "subject": email_msg.subject,
            "from": email_msg.from_addr,
            "to": email_msg.to_addrs,
            "cc": email_msg.cc_addrs,
            "date": email_msg.date,
            "has_attachments": len(email_msg.attachments) > 0,
            "attachment_count": len(email_msg.attachments),
            "in_reply_to": email_msg.in_reply_to,
            "chunk_type": "email",
        }
```

---

## Running the Pipeline

```python
def main():
    pipeline = EmailIngestionPipeline(
        chunk_size=1500,
        process_attachments=True,
        dedup=True,
    )

    result = pipeline.process_directory("/data/emails")

    print(f"Processed {result.parsed}/{result.total_emails} emails")
    print(f"Threads: {result.threads}")
    print(f"Chunks: {len(result.chunks)}")
    print(f"Time: {result.processing_seconds:.1f}s")

    # Output to JSON
    import json
    chunks_data = [
        {"text": c.text, "metadata": c.metadata}
        for c in result.chunks
    ]
    Path("/data/output/email_chunks.json").write_text(
        json.dumps(chunks_data, indent=2, default=str)
    )


if __name__ == "__main__":
    main()
```

---

## Common Pitfalls

1. **Not removing quoted text before chunking.** Reply chains duplicate content. Each reply contains the full history.
2. **Indexing signatures as content.** Signatures appear in every email but carry no information value.
3. **Missing thread context.** Individual email chunks lose thread context. Thread-level summary chunks provide conversation overview.
4. **Not deduplicating forwarded emails.** A forwarded email contains the original in full. Both get indexed as separate chunks.
5. **Ignoring email headers in chunks.** Without subject, sender, and date in the chunk text, retrieved content lacks attribution context.
6. **Processing attachments inline.** Large attachments should be extracted to temp files and processed by format-specific pipelines.

---

## Production Checklist

- [ ] Multiple formats supported (EML, MSG, MBOX)
- [ ] HTML bodies converted to clean text
- [ ] Quoted reply chains removed
- [ ] Signatures detected and stripped
- [ ] Threads reconstructed from headers
- [ ] Content deduplication removes exact and near-duplicate messages
- [ ] Each chunk includes email headers (subject, from, date) as context
- [ ] Thread-level summary chunks provide conversation overview
- [ ] Attachments extracted and processed separately
- [ ] Metadata includes message ID, thread ID, participants, and date

---

## References

- Python email module -- https://docs.python.org/3/library/email.html
- extract-msg -- https://github.com/TeamMsgExtractor/msg-extractor
- Python mailbox module -- https://docs.python.org/3/library/mailbox.html
- Gmail API -- https://developers.google.com/gmail/api
