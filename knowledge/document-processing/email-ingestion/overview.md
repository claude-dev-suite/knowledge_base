# Email Ingestion -- Comprehensive Guide

## Overview / TL;DR

Email is one of the richest and most underutilized data sources for enterprise RAG systems. Emails contain decisions, approvals, discussions, agreements, and institutional knowledge that exists nowhere else. Ingesting emails for RAG requires parsing multiple formats (EML, MSG, MBOX), extracting headers and metadata, handling MIME multipart structures, processing attachments, threading conversations, and deduplicating content. This guide covers Python's email module, extract-msg for Outlook MSG files, Gmail API integration, and production-ready code for building email-to-searchable-chunks pipelines.

---

## Why Email Ingestion Is Valuable

1. **Decision trail.** Emails capture who decided what and when, with full context.
2. **Institutional knowledge.** Long-time employees' email archives contain solutions to problems that recur years later.
3. **Relationship mapping.** Email metadata (from, to, cc) reveals organizational communication patterns.
4. **Attachment context.** Documents shared via email have contextual cover text explaining their purpose.
5. **Thread history.** Email threads capture the evolution of discussions and decisions.

---

## Format Landscape

| Format | Extension | Source | Library |
|--------|-----------|--------|---------|
| EML | .eml | Most email clients | Python email module |
| MSG | .msg | Microsoft Outlook | extract-msg |
| MBOX | .mbox | Unix mailboxes, Gmail export | mailbox module |
| PST | .pst | Outlook data file | libpff, pypff |
| Gmail API | N/A | Google Workspace | google-api-python-client |
| IMAP | N/A | Any IMAP server | imaplib |

---

## Parsing EML Files

```python
import email
from email import policy
from email.utils import parsedate_to_datetime
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class ParsedEmail:
    message_id: str
    subject: str
    from_addr: str
    to_addrs: list[str]
    cc_addrs: list[str]
    date: str
    body_text: str
    body_html: str
    attachments: list[dict] = field(default_factory=list)
    in_reply_to: str = ""
    references: list[str] = field(default_factory=list)
    headers: dict = field(default_factory=dict)


def parse_eml(file_path: str) -> ParsedEmail:
    """Parse an EML file into structured data."""
    with open(file_path, "rb") as f:
        msg = email.message_from_binary_file(f, policy=policy.default)

    # Extract headers
    message_id = msg.get("Message-ID", "")
    subject = msg.get("Subject", "")
    from_addr = msg.get("From", "")
    to_addrs = [addr.strip() for addr in (msg.get("To", "") or "").split(",") if addr.strip()]
    cc_addrs = [addr.strip() for addr in (msg.get("Cc", "") or "").split(",") if addr.strip()]
    in_reply_to = msg.get("In-Reply-To", "")
    references = (msg.get("References", "") or "").split()

    # Parse date
    date_str = msg.get("Date", "")
    try:
        date_parsed = parsedate_to_datetime(date_str)
        date = date_parsed.isoformat()
    except Exception:
        date = date_str

    # Extract body and attachments
    body_text = ""
    body_html = ""
    attachments = []

    if msg.is_multipart():
        for part in msg.walk():
            content_type = part.get_content_type()
            disposition = part.get_content_disposition()

            if disposition == "attachment":
                attachments.append({
                    "filename": part.get_filename() or "unnamed",
                    "content_type": content_type,
                    "size": len(part.get_payload(decode=True) or b""),
                })
            elif content_type == "text/plain" and not body_text:
                payload = part.get_payload(decode=True)
                if payload:
                    charset = part.get_content_charset() or "utf-8"
                    body_text = payload.decode(charset, errors="replace")
            elif content_type == "text/html" and not body_html:
                payload = part.get_payload(decode=True)
                if payload:
                    charset = part.get_content_charset() or "utf-8"
                    body_html = payload.decode(charset, errors="replace")
    else:
        payload = msg.get_payload(decode=True)
        if payload:
            charset = msg.get_content_charset() or "utf-8"
            text = payload.decode(charset, errors="replace")
            if msg.get_content_type() == "text/html":
                body_html = text
            else:
                body_text = text

    return ParsedEmail(
        message_id=message_id,
        subject=subject,
        from_addr=from_addr,
        to_addrs=to_addrs,
        cc_addrs=cc_addrs,
        date=date,
        body_text=body_text,
        body_html=body_html,
        attachments=attachments,
        in_reply_to=in_reply_to,
        references=references,
    )
```

---

## Parsing MSG Files (Outlook)

```python
import extract_msg


def parse_msg(file_path: str) -> ParsedEmail:
    """Parse an Outlook MSG file."""
    msg = extract_msg.Message(file_path)

    attachments = []
    for att in msg.attachments:
        attachments.append({
            "filename": att.longFilename or att.shortFilename or "unnamed",
            "content_type": att.mimetype or "application/octet-stream",
            "size": len(att.data) if att.data else 0,
        })

    return ParsedEmail(
        message_id=msg.messageId or "",
        subject=msg.subject or "",
        from_addr=msg.sender or "",
        to_addrs=[addr.strip() for addr in (msg.to or "").split(";") if addr.strip()],
        cc_addrs=[addr.strip() for addr in (msg.cc or "").split(";") if addr.strip()],
        date=str(msg.date) if msg.date else "",
        body_text=msg.body or "",
        body_html=msg.htmlBody.decode("utf-8", errors="replace") if msg.htmlBody else "",
        attachments=attachments,
        in_reply_to=msg.inReplyTo or "",
    )
```

---

## Parsing MBOX Archives

```python
import mailbox


def parse_mbox(mbox_path: str, max_messages: int = 10000) -> list[ParsedEmail]:
    """Parse an MBOX archive file."""
    mbox = mailbox.mbox(mbox_path)
    emails = []

    for i, msg in enumerate(mbox):
        if i >= max_messages:
            break

        try:
            parsed = _mbox_message_to_parsed(msg)
            emails.append(parsed)
        except Exception as e:
            continue

    return emails


def _mbox_message_to_parsed(msg) -> ParsedEmail:
    """Convert a mailbox message to ParsedEmail."""
    body_text = ""
    body_html = ""
    attachments = []

    if msg.is_multipart():
        for part in msg.walk():
            ctype = part.get_content_type()
            disp = part.get_content_disposition()

            if disp == "attachment":
                attachments.append({
                    "filename": part.get_filename() or "unnamed",
                    "content_type": ctype,
                    "size": len(part.get_payload(decode=True) or b""),
                })
            elif ctype == "text/plain" and not body_text:
                payload = part.get_payload(decode=True)
                if payload:
                    body_text = payload.decode(
                        part.get_content_charset() or "utf-8",
                        errors="replace",
                    )
            elif ctype == "text/html" and not body_html:
                payload = part.get_payload(decode=True)
                if payload:
                    body_html = payload.decode(
                        part.get_content_charset() or "utf-8",
                        errors="replace",
                    )
    else:
        payload = msg.get_payload(decode=True)
        if payload:
            body_text = payload.decode(
                msg.get_content_charset() or "utf-8",
                errors="replace",
            )

    return ParsedEmail(
        message_id=msg.get("Message-ID", ""),
        subject=msg.get("Subject", ""),
        from_addr=msg.get("From", ""),
        to_addrs=[a.strip() for a in (msg.get("To", "") or "").split(",") if a.strip()],
        cc_addrs=[a.strip() for a in (msg.get("Cc", "") or "").split(",") if a.strip()],
        date=msg.get("Date", ""),
        body_text=body_text,
        body_html=body_html,
        attachments=attachments,
        in_reply_to=msg.get("In-Reply-To", ""),
        references=(msg.get("References", "") or "").split(),
    )
```

---

## Gmail API Integration

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
import base64


def fetch_gmail_messages(
    credentials: Credentials,
    query: str = "",
    max_results: int = 100,
) -> list[dict]:
    """Fetch messages from Gmail API."""
    service = build("gmail", "v1", credentials=credentials)

    # List message IDs
    results = service.users().messages().list(
        userId="me",
        q=query,
        maxResults=max_results,
    ).execute()

    messages = results.get("messages", [])
    parsed = []

    for msg_ref in messages:
        msg = service.users().messages().get(
            userId="me",
            id=msg_ref["id"],
            format="full",
        ).execute()

        parsed.append(_parse_gmail_message(msg))

    return parsed


def _parse_gmail_message(msg: dict) -> dict:
    """Parse a Gmail API message response."""
    headers = {h["name"]: h["value"] for h in msg["payload"]["headers"]}

    body_text = ""
    if "parts" in msg["payload"]:
        for part in msg["payload"]["parts"]:
            if part["mimeType"] == "text/plain":
                data = part["body"].get("data", "")
                if data:
                    body_text = base64.urlsafe_b64decode(data).decode("utf-8", errors="replace")
                break
    elif msg["payload"]["body"].get("data"):
        body_text = base64.urlsafe_b64decode(
            msg["payload"]["body"]["data"]
        ).decode("utf-8", errors="replace")

    return {
        "message_id": headers.get("Message-ID", msg["id"]),
        "subject": headers.get("Subject", ""),
        "from": headers.get("From", ""),
        "to": headers.get("To", ""),
        "date": headers.get("Date", ""),
        "body": body_text,
        "thread_id": msg.get("threadId", ""),
        "labels": msg.get("labelIds", []),
    }
```

---

## Email Body Cleaning

```python
import re
from html.parser import HTMLParser


class HTMLTextExtractor(HTMLParser):
    """Extract plain text from HTML email body."""
    def __init__(self):
        super().__init__()
        self.text_parts = []
        self._skip = False

    def handle_starttag(self, tag, attrs):
        if tag in ("script", "style"):
            self._skip = True
        elif tag in ("br", "p", "div", "li", "tr"):
            self.text_parts.append("\n")

    def handle_endtag(self, tag):
        if tag in ("script", "style"):
            self._skip = False

    def handle_data(self, data):
        if not self._skip:
            self.text_parts.append(data)

    def get_text(self) -> str:
        return "".join(self.text_parts)


def clean_email_body(parsed: ParsedEmail) -> str:
    """Extract clean text from email body."""
    # Prefer plain text
    if parsed.body_text:
        text = parsed.body_text
    elif parsed.body_html:
        extractor = HTMLTextExtractor()
        extractor.feed(parsed.body_html)
        text = extractor.get_text()
    else:
        return ""

    # Remove quoted reply chains
    text = _remove_quoted_text(text)

    # Remove email signatures
    text = _remove_signature(text)

    # Clean whitespace
    text = re.sub(r'\n{3,}', '\n\n', text)
    text = re.sub(r' {2,}', ' ', text)

    return text.strip()


def _remove_quoted_text(text: str) -> str:
    """Remove quoted reply text (lines starting with >)."""
    lines = text.split("\n")
    cleaned = []
    in_quote = False

    for line in lines:
        stripped = line.strip()
        # Detect quote markers
        if stripped.startswith(">") or stripped.startswith("&gt;"):
            in_quote = True
            continue
        # Detect "On <date>, <person> wrote:" patterns
        if re.match(r'^On .+ wrote:$', stripped):
            break
        if re.match(r'^-{3,}\s*Original Message\s*-{3,}$', stripped, re.IGNORECASE):
            break
        if re.match(r'^From:\s+', stripped):
            if in_quote:
                break

        in_quote = False
        cleaned.append(line)

    return "\n".join(cleaned)


def _remove_signature(text: str) -> str:
    """Remove email signature block."""
    sig_markers = ["--", "-- ", "___", "---", "Best regards", "Regards,", "Sincerely,", "Thanks,"]

    lines = text.split("\n")
    for i, line in enumerate(lines):
        stripped = line.strip()
        if stripped in sig_markers and i > len(lines) * 0.5:
            return "\n".join(lines[:i])

    return text
```

---

## Thread Reconstruction

```python
from dataclasses import dataclass, field


@dataclass
class EmailThread:
    thread_id: str
    subject: str
    messages: list[ParsedEmail] = field(default_factory=list)
    participants: set = field(default_factory=set)
    date_range: tuple = ("", "")


def build_threads(emails: list[ParsedEmail]) -> list[EmailThread]:
    """Reconstruct email threads from individual messages."""
    # Index by message ID
    by_id = {e.message_id: e for e in emails if e.message_id}

    # Group by thread (using In-Reply-To and References)
    thread_map = {}  # root_id -> list of messages

    for email_msg in emails:
        root = _find_thread_root(email_msg, by_id)
        if root not in thread_map:
            thread_map[root] = []
        thread_map[root].append(email_msg)

    # Build thread objects
    threads = []
    for root_id, messages in thread_map.items():
        messages.sort(key=lambda m: m.date)
        participants = set()
        for m in messages:
            participants.add(m.from_addr)
            participants.update(m.to_addrs)

        threads.append(EmailThread(
            thread_id=root_id,
            subject=messages[0].subject,
            messages=messages,
            participants=participants,
            date_range=(messages[0].date, messages[-1].date),
        ))

    return threads


def _find_thread_root(email_msg: ParsedEmail, by_id: dict) -> str:
    """Find the root message ID of a thread."""
    if email_msg.references:
        return email_msg.references[0]
    if email_msg.in_reply_to and email_msg.in_reply_to in by_id:
        return _find_thread_root(by_id[email_msg.in_reply_to], by_id)
    return email_msg.message_id
```

---

## Common Pitfalls

1. **Parsing HTML emails as plain text.** Many emails are HTML-only. Always fall back to HTML extraction when plain text is empty.
2. **Including quoted reply chains.** Reply chains duplicate content exponentially. Strip quoted text before indexing.
3. **Not removing signatures.** Signatures add noise to every email chunk. Detect and strip them.
4. **Ignoring thread structure.** Emails in a thread share context. Threading enables queries like "show the full discussion about X."
5. **Not handling character encoding.** Emails use various encodings (UTF-8, ISO-8859-1, Windows-1252). Always specify `errors="replace"`.
6. **Processing attachments inline.** Large attachments should be extracted and processed separately with format-specific tools.
7. **Not deduplicating forwarded content.** Forwarded emails contain the original message in full, creating duplicate content.
8. **Missing Gmail thread IDs.** Gmail groups messages by thread. Use `threadId` for thread reconstruction instead of References headers.

---

## Production Checklist

- [ ] Multiple formats supported (EML, MSG, MBOX, Gmail API)
- [ ] HTML email bodies are converted to clean text
- [ ] Quoted reply text is removed to prevent content duplication
- [ ] Email signatures are detected and stripped
- [ ] Thread structure is reconstructed from headers
- [ ] Attachments are extracted and processed separately
- [ ] Character encoding is handled with fallbacks
- [ ] Metadata includes sender, recipients, date, subject, thread ID
- [ ] Deduplication removes forwarded/quoted duplicates
- [ ] Batch processing handles large mailbox archives

---

## References

- Python email module -- https://docs.python.org/3/library/email.html
- extract-msg -- https://github.com/TeamMsgExtractor/msg-extractor
- Python mailbox module -- https://docs.python.org/3/library/mailbox.html
- Gmail API -- https://developers.google.com/gmail/api
