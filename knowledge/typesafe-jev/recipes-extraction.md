# TypeSafe Jev - Extraction and Document Recipes

> Official Documentation: https://docs.typesafe.ai/cookbooks
> Date extraction: https://docs.typesafe.ai/cookbooks/date_extraction_cookbook
> Pre-parsed value extraction: https://docs.typesafe.ai/cookbooks/pre_parsed_value_extraction_cookbook
> Structure recovery: https://docs.typesafe.ai/cookbooks/autoformat
> Double-checking citations: https://docs.typesafe.ai/cookbooks/citation_check
> Function calling: https://docs.typesafe.ai/cookbooks/function_calling
> Line-by-line search: https://docs.typesafe.ai/cookbooks/semantic_find
> Confidence formulas: https://docs.typesafe.ai/confidence
> Last verified: 2026-10-03
> Verified against: `typesafe-sdk` 0.7.2 (Python, on `httpx2` 2.13.1), `@typesafe-ai/sdk` 0.6.0 (TypeScript, `tsc` 7.0.2, Node 22.22), `phonenumbers` 9.0.40. Current model per https://docs.typesafe.ai/models on 2026-10-03: `jev-1.13.0` (`jev-latest` and `jev-preview` both point to it). **All six cookbooks pin `jev-1.12`, and every published number below comes from `jev-1.12`**, not from the current model.

## Overview

Six of TypeSafe's official cookbooks turn Jev - a model that answers typed questions (Noul, Choice, Score) with calibrated probabilities but does not generate text - into an extraction or document-processing tool. They share one design move: **the model never produces the value**. Code finds or enumerates the candidates, Jev picks among them and reports how sure it is, and code assembles, normalizes, validates and routes the result.

| Recipe | What Jev picks | What code owns | Requests per item | Gate |
|---|---|---|---|---|
| Date extraction | date *parts*: mode, month, day, year, anchor, weekday, week (7 Choices) | calendar math, year inference, validity | 1 | min part confidence < 0.60 -> review |
| Pre-parsed values | which regex match plays the role; currency, country, credit-or-charge | candidate finding, normalization (E.164, `Decimal`) | 1 per question (9 for the three demos) | `none` hatch; P(credit) > 0.5 |
| Structure recovery | per line pair "split mid-sentence?" (Noul); per block type + companions | line splitting, merge thresholds, Markdown rendering | 2, in sequence | join bar 0.2 / 0.5 by punctuation; list ordered if mean step >= 0.5 |
| Citation check | supports / contradicts / says nothing (1 Choice) | quote existence (string match), section lookup | 1 per citation that reaches the model | confidence >= 0.8 -> auto |
| Function calling | function (Choice) + every closed-set argument + "was it stated?" | reading signatures, defaults, invocation | 1 per command | weakest judgement |
| Line-by-line search | which line id answers (Choice over up to 255 ids) + "does any line answer?" (Noul) | line tagging, ranking, verdict bands | 1 per query | exists >= 0.7 found, < 0.35 absent |

This document is a condensed, faithful port of those six pages. For each: the problem, the exact questions, the call, the code-side logic, an offline run, the published results, and the pitfalls - including several the cookbook pages do not mention, found by running their code. It goes deeper than the dev-suite skill `typesafe-jev` (which covers the API surface); read that first for request/response shapes, limits and errors.

---

## Table of Contents

1. [How to read the code in this document](#how-to-read-the-code-in-this-document)
2. [Confidence math every recipe depends on](#confidence-math-every-recipe-depends-on)
3. [The offline harness](#the-offline-harness)
4. [Recipe 1: Date extraction](#recipe-1-date-extraction)
5. [Recipe 2: Pre-parsed value extraction](#recipe-2-pre-parsed-value-extraction)
6. [Recipe 3: Structure recovery](#recipe-3-structure-recovery)
7. [Recipe 4: Double-checking citations](#recipe-4-double-checking-citations)
8. [Recipe 5: Function calling](#recipe-5-function-calling)
9. [Recipe 6: Line-by-line search](#recipe-6-line-by-line-search)
10. [Cross-recipe pitfalls](#cross-recipe-pitfalls)
11. [Published numbers, with sources](#published-numbers-with-sources)
12. [What could not be verified](#what-could-not-be-verified)

---

## How to read the code in this document

Every code block carries one of these labels on the line above it:

| Label | Meaning |
|---|---|
| **verbatim** | Copied unchanged from the cited cookbook page (fetched 2026-10-03 as `<page>.md`; identical to the `llms-full.txt` dump apart from image URLs). |
| **verbatim, executed** | As above, and executed offline on 2026-10-03 with the real `typesafe-sdk` 0.7.2 client over a mock HTTP transport (no API key, no network call to TypeSafe). |
| **condensed, executed** | Derived from the cited page by the listed mechanical edits only, then executed offline. The edits are always some of: `@json_cache` decorator removed; `if __name__ == "__cookbook__":` guard removed and its body dedented; `cooksafe` / `IPython` / `json_cache` lines dropped; the `Path` import dropped where only the cache used it; in one block (structure recovery's fetch) a final `print` dropped. Each block's label names its own edits. |
| **original, executed** | Written for this document (harness, canned answers, edge-case probes, reconstructions), executed offline. |
| **original, type-checked + executed** | TypeScript, `tsc --noEmit` clean under `strict`, and run with `node --experimental-strip-types` against a fake `fetch`. |

The cookbooks wrap every network call in `cooksafe.JsonCache` (PyPI `cooksafe` 0.2.0; the pages pin `cooksafe>=0.2.0,<0.3.0`) and ship a `json_cache.json`, so re-rendering a page replays published answers. The ports here drop that cache, which matters in one place: see [Date extraction pitfalls](#pitfalls-1).

"Published output" blocks are copied from the page. Where the offline run reproduced a published output byte-for-byte (it did for every table except wall-clock timings), that is stated.

Within a recipe, run the blocks in document order, with two exceptions: in Recipe 3 the canned-answer cell ("Offline run") must run right after the setup cell, before pass 1, because pass 1 already calls the client (the setup cell also reads `os.environ["TYPESAFE_API_KEY"]`, so set it to any value offline); in Recipe 5 the `plot_price` signature needs `from typing import Literal`, which the tools cell further down imports.

---

## Confidence math every recipe depends on

Source: https://docs.typesafe.ai/confidence (fetched 2026-10-03).

- **Noul**: the answer is `noul` = P(yes). There is no separate `confidence`; the documented confidence-style equivalent is `|2p - 1|`.
- **Choice**: `confidence = (p_max - 1/n) / (1 - 1/n)` for `n` options. It is **not** the top probability. It rescales `p_max` so an even split is 0 and certainty is 1.
- **Score**: a mean-absolute-deviation formula over ordered levels (not used by these six recipes).

The `n`-dependence changes what each recipe's threshold actually means. Converting each published gate back to the top probability it requires (arithmetic on the documented formula, computed 2026-10-03):

| Gate (recipe) | Options `n` | Confidence | Required `p_max` |
|---|---|---|---|
| `REVIEW_BELOW` on `mode` (date) | 3 | 0.60 | 0.733 |
| `REVIEW_BELOW` on `month` (date) | 13 | 0.60 | 0.631 |
| `REVIEW_BELOW` on `day` (date) | 32 | 0.60 | 0.613 |
| `REVIEW_BELOW` on `year` (date) | 153 | 0.60 | 0.603 |
| `AUTO_ACCEPT` (citation check) | 3 | 0.80 | 0.867 |
| suggested review bar for block type (structure recovery; see note below) | 6 | 0.55 | 0.625 |
| *illustrative only, not a cookbook gate:* 0.50 on `where` over 218 line ids (search gates on the `exists` Noul, not on `where`) | 218 | 0.50 | 0.502 |

One published value checks the formula: structure recovery's least certain block has `paragraph 0.53` over 6 types, giving `(0.53 - 1/6) / (5/6) = 0.436`, printed as `0.43`. The other direction is a derivation, not a check (the page publishes no probabilities for it): citation check's `pii_encryption` verdict at confidence 0.27 over 3 options implies `p_max = 0.513` - the model leaned toward "says nothing" but only just.

Note on the 0.55 bar: the structure-recovery page words it as *"any block whose type confidence (the probability behind the winning choice) is under 0.55"*. That parenthetical describes `p_max`, not `confidence`, and the page's own numbers show the two differ (confidence 0.43 vs probability 0.53 for the same block). Read as `confidence`, the bar needs `p_max >= 0.625`; read as `p_max`, it is 0.55. The page does not say which it means.

Consequence: one `REVIEW_BELOW` constant applied across Choices of different sizes (as date extraction does) is a different bar for each part. Over 100+ options, `confidence` and `p_max` are almost the same number; over 3 options they are far apart.

---

## The offline harness

**original, executed.** A real `TypeSafeClient` (constructor parameter `transport: httpx2.BaseTransport | None`, confirmed with `inspect.signature` on 0.7.2) over `httpx2.MockTransport`. The response JSON follows the OpenAPI `ChoiceAnswer`/`NoulAnswer` shapes (`type`, `choice`, `confidence`, `probabilities`; `type`, `noul`), and the SDK parses it into its own `ChoiceAnswer`/`NoulAnswer` models, so every recipe below runs through the same code path a live call would.

```python
import json

import httpx2  # the HTTP library typesafe-sdk 0.7.2 is built on
from typesafe_sdk import TypeSafeClient


def choice_answer(options: list[str], pick: str, confidence: float) -> dict:
    """A wire-format Choice answer whose confidence follows the documented formula
    confidence = (p_max - 1/n) / (1 - 1/n); the rest of the mass is spread evenly."""
    n = len(options)
    p_max = confidence * (1 - 1 / n) + 1 / n
    rest = (1 - p_max) / (n - 1)
    probabilities = {o: (p_max if o == pick else rest) for o in options}
    return {"type": "choice", "choice": pick, "confidence": confidence,
            "probabilities": probabilities}


def noul_answer(p: float) -> dict:
    return {"type": "noul", "noul": p}


def fake_client(answer_fn, model: str = "jev-1.13.0") -> tuple[TypeSafeClient, list]:
    """A real TypeSafeClient whose transport never leaves the process.

    answer_fn(state, questions_json) -> {question_id: wire answer}. Every request body
    is appended to the returned list so tests can inspect what would have been sent."""
    sent: list[dict] = []

    def handler(request: httpx2.Request) -> httpx2.Response:
        assert request.url.path == "/v1/systemone", request.url
        body = json.loads(request.content)
        sent.append(body)
        answers = answer_fn(body["state"], body["questions"])
        return httpx2.Response(200, json={
            "model": model, "answers": answers,
            "usage": {"input_tokens": 1000, "output_tokens": 0}})

    return TypeSafeClient(api_key="offline", transport=httpx2.MockTransport(handler)), sent
```

Every recipe's canned answers are built from the numbers the cookbook publishes; where the page publishes only an aggregate (for example a date's final confidence), the per-question split is marked synthetic in the code.

---

## Recipe 1: Date extraction

Source: https://docs.typesafe.ai/cookbooks/date_extraction_cookbook (difficulty "Beginner" on https://docs.typesafe.ai/cookbooks).

### Problem

`extract_date(document, role)` takes a document and a phrase naming the date wanted ("the deadline to return the form") and returns a `date`, a confidence, and a review flag. It must handle spelled-out dates ("August 14, 2027"), dates without a year, relative dates ("tomorrow", "next Thursday"), and dates the document never states.

The reason for the design is a documented weakness: *"`jev-1.13` reads dates as text, not as ordered quantities. Asking which of two dates comes first, how far apart they are, or whether one falls inside a window is unreliable."* (https://docs.typesafe.ai/model-jaggedness/jev-1.13, last reviewed 2026-10-02). The fix the vendor prescribes: every part of a date is a small closed set, so extraction becomes Choices; code owns ordering, duration, offsets and weekdays.

### Question design

Seven Choices in one request. `mode` decides which of the other six code reads:

| Question | Options (`n`) | Read when |
|---|---|---|
| `mode` | `absolute`, `relative`, `none` (3) | always |
| `month` | 12 months + `none` (13) | `mode == absolute` |
| `day` | `"1"`..`"31"` + `none` (32) | `mode == absolute` |
| `year` | `"1900"`..`"2050"` + `out_of_range` + `none` (153) | `mode == absolute` |
| `day_anchor` | `today`, `tomorrow`, `day_after`, `weekday`, `none` (5) | `mode == relative` |
| `weekday` | 7 days + `none` (8) | `day_anchor == weekday` |
| `week_offset` | `current`, `next`, `none` (3) | `day_anchor == weekday` |

Design points worth copying:

- **Two distinct escapes on `year`**: `none` ("no year stated", code infers it) versus `out_of_range` ("a year is stated but not listed", code flags it). Collapsing them would make code guess on a stated 1850.
- **Option descriptions only where needed**: most options are `None`; the `none` / `out_of_range` escapes on the six part questions carry a sentence (`absent`, or their own text on `year`) so "not stated" has a definition. `mode`'s own `none` has no description; it is defined in `mode`'s `instructions` instead.
- **The role phrase goes into every `instructions` string**, so the same seven questions are reused for any date in any document.
- The page suggests, if a 153-option year list bothers you, extracting year-like numbers from the text first and offering only those - the pre-parsed pattern of Recipe 2.

**condensed, executed** (setup; `cooksafe`, `IPython`, `Path`, `json_cache` lines removed):

```python
import os
from datetime import date, timedelta

from typesafe_sdk import Choice, TypeSafeClient

TYPESAFE_MODEL = "jev-1.12"
TODAY = date(
    2026, 7, 30
)  # fixed reference "today" so relative dates resolve reproducibly
REVIEW_BELOW = 0.60  # gate: a date below this confidence is flagged for a human

MONTHS = {
    "January": 1,
    "February": 2,
    "March": 3,
    "April": 4,
    "May": 5,
    "June": 6,
    "July": 7,
    "August": 8,
    "September": 9,
    "October": 10,
    "November": 11,
    "December": 12,
}
WEEKDAYS = [
    "Monday",
    "Tuesday",
    "Wednesday",
    "Thursday",
    "Friday",
    "Saturday",
    "Sunday",
]
YEAR_WINDOW = list(range(1900, 2051))  # 1900..2050
```

**condensed, executed** (client; `__cookbook__` guard removed):

```python
client = TypeSafeClient(
    api_key=os.environ.get(
        "TYPESAFE_API_KEY", "cache-only"
    ),  # cached re-renders need no key
    base_url=os.environ.get("TYPESAFE_BASE_URL"),
    timeout=30.0,
)
```

**verbatim, executed:**

```python
def date_questions(role: str) -> dict[str, Choice]:
    """Seven typed choices that read a date's shape and parts off the text -- no math."""
    absent = "The document does not state this, or it is not this kind of date."
    return {
        "mode": Choice(
            instructions=(
                f"How is {role} written? 'absolute' = a calendar date naming a month (e.g. "
                "'August 14', 'the 3rd of March'); 'relative' = given relative to today (today, "
                "tomorrow, the day after tomorrow, or a named weekday such as 'next Thursday'); "
                "'none' = the document does not state this date."
            ),
            criteria={"absolute": None, "relative": None, "none": None},
        ),
        "month": Choice(
            instructions=f"If {role} is an absolute calendar date, which month is it in?",
            criteria={m: None for m in MONTHS} | {"none": absent},
        ),
        "day": Choice(
            instructions=f"If {role} is an absolute calendar date, which day of the month (1-31)?",
            criteria={str(d): None for d in range(1, 32)} | {"none": absent},
        ),
        "year": Choice(
            instructions=(
                f"If {role} is an absolute calendar date, which year? Pick 'none' if the document "
                "states no year (code infers it), or 'out_of_range' if a year is stated but not "
                "in the list."
            ),
            criteria={str(y): None for y in YEAR_WINDOW}
            | {
                "out_of_range": "A year is stated for this date but is outside the listed range.",
                "none": "No year is stated for this date.",
            },
        ),
        "day_anchor": Choice(
            instructions=(
                f"If {role} is relative to today, which day is it? 'today', 'tomorrow', "
                "'day_after' (the day after tomorrow), or 'weekday' (a named day of the week)."
            ),
            criteria={
                "today": None,
                "tomorrow": None,
                "day_after": None,
                "weekday": None,
                "none": absent,
            },
        ),
        "weekday": Choice(
            instructions=f"If {role} names a day of the week, which one?",
            criteria={w: None for w in WEEKDAYS} | {"none": absent},
        ),
        "week_offset": Choice(
            instructions=(
                f"If {role} names a weekday, which week is it in? 'next' for 'next Thursday' or "
                "'Thursday next week'; 'current' for 'this Thursday'; 'none' for a bare weekday "
                "with no qualifier (just 'Thursday' / 'on Thursday')."
            ),
            criteria={"current": None, "next": None, "none": absent},
        ),
    }
```

### The call and the code-side logic

**condensed, executed** (`@json_cache` removed). `read_parts` is the only function that talks to Jev; `resolve_weekday` and `assemble` are pure and unit-testable:

```python
def read_parts(document: str, role: str) -> dict:
    """One TypeSafe call -> {part: {choice, confidence}} for the seven questions."""
    answers = client.system_one(
        state=document, questions=date_questions(role), model=TYPESAFE_MODEL
    ).answers
    return {
        part: {"choice": ans.choice, "confidence": ans.confidence}
        for part, ans in answers.items()
    }


def resolve_weekday(today: date, weekday: str, week_offset: str) -> date:
    """Which date a named weekday points to, by our stated convention: a bare weekday is the next
    occurrence on or after today; 'next' is the following calendar week; 'current' is this week."""
    w = WEEKDAYS.index(weekday)
    this_monday = today - timedelta(days=today.weekday())
    if week_offset == "next":
        return this_monday + timedelta(days=7 + w)
    if week_offset == "current":
        return this_monday + timedelta(days=w)
    return today + timedelta(days=(w - today.weekday()) % 7)


def assemble(parts: dict, today: date = TODAY) -> dict:
    """Resolve the parts TypeSafe read into a concrete date, in code. Confidence is the weakest of
    the parts the shape actually used."""
    mode = parts["mode"]["choice"]
    confs = [parts["mode"]["confidence"]]

    def result(resolved: date | None, note: str) -> dict:
        usable = [c for c in confs if c is not None]
        confidence = min(usable) if usable else None
        needs_review = (
            resolved is None or confidence is None or confidence < REVIEW_BELOW
        )
        return {
            "date": resolved,
            "confidence": confidence,
            "needs_review": needs_review,
            "note": note,
        }

    if mode == "none":
        return result(None, "no such date stated")

    if mode == "absolute":
        month, day, year = (
            parts["month"]["choice"],
            parts["day"]["choice"],
            parts["year"]["choice"],
        )
        confs += [
            parts["month"]["confidence"],
            parts["day"]["confidence"],
            parts["year"]["confidence"],
        ]
        if "none" in (month, day) or not day.isdigit() or month not in MONTHS:
            return result(None, "absolute date incomplete")
        if (
            year == "out_of_range"
        ):  # a year is stated but off the list -> flag, don't guess
            return result(None, f"year outside {YEAR_WINDOW[0]}-{YEAR_WINDOW[-1]}")
        if (
            year == "none"
        ):  # no year stated -> infer this year, bumped to next if well past
            try:
                resolved = date(today.year, MONTHS[month], int(day))
            except (
                ValueError
            ):  # e.g. February 30 -- an inconsistent read, not a real date
                return result(None, f"impossible date: {month} {day}")
            if resolved < today - timedelta(days=31):
                resolved = date(today.year + 1, MONTHS[month], int(day))
            return result(resolved, "")
        try:  # a stated, in-range year
            return result(date(int(year), MONTHS[month], int(day)), "")
        except ValueError:
            return result(None, f"impossible date: {year}-{month}-{day}")

    if mode == "relative":
        anchor = parts["day_anchor"]["choice"]
        confs.append(parts["day_anchor"]["confidence"])
        if anchor == "today":
            return result(today, "")
        if anchor == "tomorrow":
            return result(today + timedelta(days=1), "")
        if anchor == "day_after":
            return result(today + timedelta(days=2), "")
        if anchor == "weekday":
            weekday, offset = parts["weekday"]["choice"], parts["week_offset"]["choice"]
            confs += [
                parts["weekday"]["confidence"],
                parts["week_offset"]["confidence"],
            ]
            if weekday not in WEEKDAYS:
                return result(None, "relative weekday not read")
            return result(resolve_weekday(today, weekday, offset), "")
        return result(None, "relative day not read")

    return result(None, f"unrecognized mode: {mode}")


def extract_date(document: str, role: str) -> dict:
    return assemble(read_parts(document, role))
```

What `assemble` encodes:

- **Confidence of a date = the minimum confidence over the parts actually used.** An absolute date uses `mode`, `month`, `day`, `year`; a weekday date uses `mode`, `day_anchor`, `weekday`, `week_offset`. Unused speculative answers never lower it.
- **Validation is by construction**: `date(...)` raising `ValueError` turns an inconsistent read (February 30) into a review item instead of a wrong date.
- **Year inference**: no stated year -> `TODAY.year`, moved to next year only when the result is more than 31 days in the past.
- **Weekday convention**, stated in the code: bare weekday = next occurrence on or after today; `next` = the following calendar week; `current` = this calendar week.
- Anything `None` - including a correctly detected "not stated" - is `needs_review=True`.

### Offline run

**original, executed** - canned answers. Only each date's final confidence is published; the per-part split is synthetic:

```python
from harness import choice_answer, fake_client

# Canned (choice, confidence) per part. The cookbook publishes only each date's final
# confidence (the minimum over the parts assemble() uses); the per-part split is synthetic.
CANNED = {
    "the date the agreement takes effect": {
        "mode": ("absolute", 0.99), "month": ("January", 0.97),
        "day": ("1", 0.98), "year": ("2025", 0.99)},
    "the date the agreement expires": {
        "mode": ("absolute", 0.99), "month": ("December", 0.95),
        "day": ("31", 0.91), "year": ("2027", 0.97)},
    "the deadline to return the form": {
        "mode": ("absolute", 0.98), "month": ("August", 0.99),
        "day": ("14", 0.95), "year": ("none", 0.96)},
    "the date of the kickoff call": {
        "mode": ("absolute", 0.62), "month": ("none", 0.46),
        "day": ("none", 0.55), "year": ("none", 0.80)},
    "the date the survey closes": {
        "mode": ("relative", 0.94), "day_anchor": ("today", 0.97)},
    "the date of the design review": {
        "mode": ("relative", 0.97), "day_anchor": ("weekday", 0.95),
        "weekday": ("Thursday", 0.99), "week_offset": ("next", 0.92)},
}


def date_answers(state, questions):
    role = next(r for r in CANNED if r in questions["mode"]["instructions"])
    canned = CANNED[role]
    return {
        qid: choice_answer(list(q["criteria"]), *canned.get(qid, ("none", 0.90)))
        for qid, q in questions.items()
    }


client, sent = fake_client(date_answers)
```

**condensed, executed** (`__cookbook__` guard removed). The offline output below is byte-identical to the published output:

```python
CONTRACT = "This agreement is effective January 1, 2025 and expires December 31, 2027."
FORM = "Please return the signed form by August 14."
SURVEY = "Heads up - the customer survey closes today at 5pm."
REVIEW = "Let's schedule the design review for next Thursday."

# (document, question phrase, expected date) -- the expected value is only for the scorecard.
EXAMPLES = [
    (CONTRACT, "the date the agreement takes effect", date(2025, 1, 1)),
    (CONTRACT, "the date the agreement expires", date(2027, 12, 31)),
    (FORM, "the deadline to return the form", date(2026, 8, 14)),
    (FORM, "the date of the kickoff call", None),
    (SURVEY, "the date the survey closes", date(2026, 7, 30)),
    (REVIEW, "the date of the design review", date(2026, 8, 6)),
]

print(f"{'':3}{'question':<38}{'expected':<12}{'got':<12}{'conf':>6}  flags")
print("-" * 84)
for document, role, expected in EXAMPLES:
    r = extract_date(document, role)
    got = r["date"].isoformat() if r["date"] else "none"
    exp = expected.isoformat() if expected else "none"
    mark = "OK" if r["date"] == expected else "XX"
    conf = f"{r['confidence']:.2f}" if r["confidence"] is not None else " n/a"
    flags = "  <== review" if r["needs_review"] else ""
    if r["note"]:
        flags += f"  ({r['note']})"
    print(f"{mark:<3}{role:<38}{exp:<12}{got:<12}{conf:>6}{flags}")
```

```text
   question                              expected    got           conf  flags
------------------------------------------------------------------------------------
OK the date the agreement takes effect   2025-01-01  2025-01-01    0.97
OK the date the agreement expires        2027-12-31  2027-12-31    0.91
OK the deadline to return the form       2026-08-14  2026-08-14    0.95
OK the date of the kickoff call          none        none          0.46  <== review  (absolute date incomplete)
OK the date the survey closes            2026-07-30  2026-07-30    0.94
OK the date of the design review         2026-08-06  2026-08-06    0.92
```

**condensed, executed** - the routing cell:

```python
confident = [
    (doc, role)
    for doc, role, _ in EXAMPLES
    if not extract_date(doc, role)["needs_review"]
]
review = [
    (doc, role)
    for doc, role, _ in EXAMPLES
    if extract_date(doc, role)["needs_review"]
]
print(f"auto-accept ({len(confident)}):")
for _doc, role in confident:
    print(f"  - {role}")
print(f"\nsend to review ({len(review)}):")
for _doc, role in review:
    r = extract_date(_doc, role)
    print(
        f"  - {role}  (conf {r['confidence']:.2f} / {r['note'] or 'low confidence'})"
    )
```

```text
auto-accept (5):
  - the date the agreement takes effect
  - the date the agreement expires
  - the deadline to return the form
  - the date the survey closes
  - the date of the design review

send to review (1):
  - the date of the kickoff call  (conf 0.46 / absolute date incomplete)
```

### Published results

From the page (model `jev-1.12`, `TODAY = 2026-07-30`, a Thursday): all six rows correct. Five dates auto-accepted at confidence 0.91-0.97; the kickoff call, which the form never mentions, came back `mode=absolute` with no month (note `absolute date incomplete`), confidence 0.46, flagged. No cost or latency is published for this cookbook.

### Pitfalls

Probed offline against `assemble` (pure code, `TODAY = 2026-07-30`). **original, executed:**

```python
def parts(**chosen: str) -> dict:
    """Every part 'none' at confidence 0.9, overridden by the keyword arguments."""
    names = ("mode", "month", "day", "year", "day_anchor", "weekday", "week_offset")
    return {p: {"choice": chosen.get(p, "none"), "confidence": 0.9} for p in names}


EDGE_CASES = {
    "February 30, no year": parts(mode="absolute", month="February", day="30"),
    "June 1, no year (59 days back)": parts(mode="absolute", month="June", day="1"),
    "July 5, no year (25 days back)": parts(mode="absolute", month="July", day="5"),
    "May 2, year off the list": parts(
        mode="absolute", month="May", day="2", year="out_of_range"),
    "bare 'Thursday' on a Thursday": parts(
        mode="relative", day_anchor="weekday", weekday="Thursday"),
    "'this Monday' on a Thursday": parts(
        mode="relative", day_anchor="weekday", weekday="Monday", week_offset="current"),
    "'next Monday' on a Thursday": parts(
        mode="relative", day_anchor="weekday", weekday="Monday", week_offset="next"),
    "mode none at 0.9": parts(mode="none"),
}
for label, p in EDGE_CASES.items():
    r = assemble(p)  # TODAY = 2026-07-30, a Thursday
    print(f"{label:<32}{str(r['date']):<12}review={r['needs_review']!s:<6}{r['note']}")

# What went over the (fake) wire: the routing cell calls extract_date() again per row.
print(f"\nrequests sent: {len(sent)} for {len(EXAMPLES)} dates")
print({qid: len(q["criteria"]) for qid, q in sent[0]["questions"].items()})
```

```text
February 30, no year            None        review=True  impossible date: February 30
June 1, no year (59 days back)  2027-06-01  review=False 
July 5, no year (25 days back)  2026-07-05  review=False 
May 2, year off the list        None        review=True  year outside 1900-2050
bare 'Thursday' on a Thursday   2026-07-30  review=False 
'this Monday' on a Thursday     2026-07-27  review=False 
'next Monday' on a Thursday     2026-08-03  review=False 
mode none at 0.9                None        review=True  no such date stated

requests sent: 19 for 6 dates
{'mode': 3, 'month': 13, 'day': 32, 'year': 153, 'day_anchor': 5, 'weekday': 8, 'week_offset': 3}
```

1. **Uncached, the routing cell re-calls the API.** It calls `extract_date` once per example in each of two list comprehensions, plus again for each review row: 19 requests for 6 dates in the run above. The cookbook is free only because `@json_cache` replays them. In production, compute each result once and partition the list.
2. **A correctly detected "not stated" always goes to review.** `mode == "none"` returns `date=None`, and `None` forces `needs_review=True` even at confidence 0.9. If "the document does not say" is a legitimate answer for your workflow, route it separately on `mode`'s confidence.
3. **`current` can resolve to the past.** "this Monday" said on a Thursday resolves to 2026-07-27 and is not flagged. A bare weekday equal to today resolves to *today*. `next Monday` on a Thursday is 2026-08-03 (4 days away), by the stated calendar-week convention. These are conventions, not model errors - pick yours and write it down.
4. **Year inference is role-blind.** "June 1" with no year, 59 days back, becomes 2027-06-01. Right for a deadline, wrong for a signing date. Make the bump rule depend on whether the role is past- or future-facing.
5. **One gate, several bars.** `REVIEW_BELOW = 0.60` requires `p_max` 0.733 on `mode` but 0.603 on `year` (see the table above).
6. **Option order.** `year` lists 1900 first; the jaggedness page reports that Jev "leans toward the option that comes first" in some cases. Re-run the important Choices with the list rotated or reversed and compare.
7. **Model pin.** The page pins `jev-1.12`; `jev-1.13.0` is current. The models page says to pin the versioned ID you tuned thresholds on and move on your own schedule.

---

## Recipe 2: Pre-parsed value extraction

Source: https://docs.typesafe.ai/cookbooks/pre_parsed_value_extraction_cookbook (difficulty "Beginner").

### Problem

Get an exact value - an email address, a phone number, an amount - that plays a specific role in a document. Jev cannot generate text (*"For data extraction, it is better to extract possible options using regex or a generative model and let `jev-1.13` pick the correct extraction."* - jaggedness page), so:

1. a regex, tuned to over-find, produces candidate spans;
2. a Choice whose **options are the spans themselves** picks one (or `none`), and other questions read attributes (country, currency, credit vs charge);
3. code copies the picked span and normalizes it.

Because the answer is one of the option keys, the value is a verbatim copy of a regex match: it *"cannot invent a value or transpose a digit"*.

### Question design and the call

**condensed, executed** (setup; `cooksafe`, `IPython`, `Path`, `json_cache` lines removed):

```python
import os
import re
from decimal import Decimal

import phonenumbers
from typesafe_sdk import Choice, Noul, TypeSafeClient

TYPESAFE_MODEL = "jev-1.12"
NONE = "none"  # the escape hatch on every selection: "none of the candidates fits"

# base_url defaults to https://api.typesafe.ai/ ; the env override points at another deployment.
ts = TypeSafeClient(
    api_key=os.environ.get(
        "TYPESAFE_API_KEY", "cache-only"
    ),  # cached re-renders need no key
    base_url=os.environ.get("TYPESAFE_BASE_URL"),
    timeout=30.0,
)
```

**condensed, executed** (`@json_cache` removed from `pick`, `classify`, `is_true`):

```python
EMAIL_RE = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")
PHONE_RE = re.compile(r"\(?\+?\d[\d\s()\-.]{6,}\d")
MONEY_RE = re.compile(r"[$€£¥]\s?\d[\d,]*(?:\.\d{2})?")


def find(pattern: re.Pattern, text: str) -> list[str]:
    """Code-side candidate finder: recall-tuned regex, deduped, in document order."""
    seen: set[str] = set()
    out: list[str] = []
    for match in pattern.findall(text):
        span = match.strip()
        if span and span not in seen:
            seen.add(span)
            out.append(span)
    return out


def pick(document: str, candidates: list[str], question: str) -> dict:
    """TypeSafe selects which found span plays the role. Returns {choice, confidence}.

    The options ARE the candidate spans, so ``choice`` is a verbatim copy of one of them (or the
    ``none`` hatch) - the model chooses, code owns the string."""
    criteria = {c: None for c in candidates} | {
        NONE: "None of these is the requested value."
    }
    answer = ts.system_one(
        state=document,
        questions={"pick": Choice(instructions=question, criteria=criteria)},
        model=TYPESAFE_MODEL,
    ).answers["pick"]
    return {"choice": answer.choice, "confidence": answer.confidence}


def classify(document: str, question: str, options: list[str]) -> dict:
    """A small Choice over a fixed label set (currency, country, ...). Returns {choice, confidence}."""
    answer = ts.system_one(
        state=document,
        questions={
            "q": Choice(instructions=question, criteria={o: None for o in options})
        },
        model=TYPESAFE_MODEL,
    ).answers["q"]
    return {"choice": answer.choice, "confidence": answer.confidence}


def is_true(document: str, question: str) -> float:
    """A yes/no Noul. Returns P(yes)."""
    return (
        ts.system_one(
            state=document,
            questions={"q": Noul(instructions=question)},
            model=TYPESAFE_MODEL,
        )
        .answers["q"]
        .noul
    )
```

The three question shapes:

| Helper | Type | Options |
|---|---|---|
| `pick` | Choice | each candidate span (description `None`) + `none: "None of these is the requested value."` |
| `classify` | Choice | a fixed label list (country codes, currency codes), no escape |
| `is_true` | Noul | - |

### Offline run

**original, executed** - canned answers (published confidences where the page prints them):

```python
from harness import choice_answer, fake_client, noul_answer

# Canned answers keyed by the question text; confidences are the published ones.
PICKS = {
    "Which email address does the sender want their receipt sent to?": ("dana.personal@gmail.com", 0.98),
    "Which email address did this message come from (the From line)?": ("dana.whit@acme-corp.com", 1.00),
    "Which of these is the direct mobile / cell number?": ("(415) 555-0177", 1.00),
    "In what country is this office located?": ("US", 0.90),
    "What currency are these amounts in?": ("USD", 0.97),  # confidence not published
    "Which amount is the total the customer must pay?": ("$1,315.50", 0.99),  # not published
    "Which amount is the courtesy credit that was applied?": ("$50.00", 0.99),  # not published
}
P_CREDIT = {"$1,315.50": 0.01, "$50.00": 0.99, "$1,200.00": 0.01, "$115.50": 0.01}  # last two synthetic


def pp_answers(state, questions):
    out = {}
    for qid, q in questions.items():
        if q["type"] == "noul":
            amount = next(a for a in P_CREDIT if a in q["instructions"])
            out[qid] = noul_answer(P_CREDIT[amount])
        else:
            out[qid] = choice_answer(list(q["criteria"]), *PICKS[q["instructions"]])
    return out


ts, sent = fake_client(pp_answers)
```

**verbatim, executed** - email (output byte-identical to the published output):

```python
EMAIL_DOC = """From: Dana Whit <dana.whit@acme-corp.com>
To: billing@acme-corp.com
Cc: orders@acme-corp.com
Reply-To: dana.personal@gmail.com

Hi team - please don't use the billing alias for this one. Send my receipt to my
personal address instead. Thanks, Dana."""

emails = find(EMAIL_RE, EMAIL_DOC)
receipt = pick(
    EMAIL_DOC, emails, "Which email address does the sender want their receipt sent to?"
)
sender = pick(
    EMAIL_DOC, emails, "Which email address did this message come from (the From line)?"
)

print("candidates :", emails)
# code copies the picked value verbatim and normalizes (lowercase); it never re-types it
print(
    f"receipt -> : {receipt['choice'].lower():<28} (conf {receipt['confidence']:.2f})"
)
print(f"sender  -> : {sender['choice'].lower():<28} (conf {sender['confidence']:.2f})")
```

```text
candidates : ['dana.whit@acme-corp.com', 'billing@acme-corp.com', 'orders@acme-corp.com', 'dana.personal@gmail.com']
receipt -> : dana.personal@gmail.com      (conf 0.98)
sender  -> : dana.whit@acme-corp.com      (conf 1.00)
```

**verbatim, executed** - phone, normalized to E.164 with `phonenumbers` using the model-read country:

```python
PHONE_DOC = """Reach our San Francisco office at these numbers: main desk (415) 555-0199,
billing fax (415) 555-0142, and my direct cell (415) 555-0177. Call the cell if it's urgent."""

phones = find(PHONE_RE, PHONE_DOC)
mobile = pick(PHONE_DOC, phones, "Which of these is the direct mobile / cell number?")
region = classify(
    PHONE_DOC,
    "In what country is this office located?",
    ["US", "GB", "DE", "FR", "CA", "AU"],
)

# code copies the picked value and normalizes it with the model-supplied country
parsed = phonenumbers.parse(mobile["choice"], region["choice"])
e164 = phonenumbers.format_number(parsed, phonenumbers.PhoneNumberFormat.E164)

print("candidates :", phones)
print(f"mobile  -> : {mobile['choice']}  (conf {mobile['confidence']:.2f})")
print(f"country -> : {region['choice']}  (conf {region['confidence']:.2f})")
print(f"E.164   -> : {e164}")
```

```text
candidates : ['(415) 555-0199', '(415) 555-0142', '(415) 555-0177']
mobile  -> : (415) 555-0177  (conf 1.00)
country -> : US  (conf 0.90)
E.164   -> : +14155550177
```

**verbatim, executed** - money:

```python
MONEY_DOC = """Invoice INV-2087.
Subtotal: $1,200.00
Sales tax: $115.50
Total due: $1,315.50
A $50.00 courtesy credit from last month has already been applied."""

amounts = find(MONEY_RE, MONEY_DOC)
currency = classify(
    MONEY_DOC,
    "What currency are these amounts in?",
    ["USD", "EUR", "GBP", "JPY", "CAD"],
)
total = pick(MONEY_DOC, amounts, "Which amount is the total the customer must pay?")
credit = pick(
    MONEY_DOC, amounts, "Which amount is the courtesy credit that was applied?"
)


def to_decimal(value: str) -> Decimal:
    """Copy the picked value and parse the number in code (US grouping/decimal here)."""
    return Decimal(re.sub(r"[^\d.]", "", value))


for label, chosen in [("total due", total), ("credit", credit)]:
    is_credit = is_true(
        MONEY_DOC,
        f"Is the amount {chosen['choice']} a credit or refund to the customer, not a charge?",
    )
    kind = "credit" if is_credit > 0.5 else "charge"
    print(
        f"{label:<10}: {chosen['choice']:<10} -> {to_decimal(chosen['choice'])} {currency['choice']} "
        f"({kind}, P(credit)={is_credit:.2f})"
    )
print("\ncandidates :", amounts)
```

```text
total due : $1,315.50  -> 1315.50 USD (charge, P(credit)=0.01)
credit    : $50.00     -> 50.00 USD (credit, P(credit)=0.99)

candidates : ['$1,200.00', '$115.50', '$1,315.50', '$50.00']
```

### Batching the questions (not in the cookbook)

The cookbook sends one question per request: 9 requests for its three documents (2 email, 2 phone, 5 money). Every question about one document shares one state, so they can go in one request, and the attribute question that depends on the pick ("is *this* amount a credit?") can be asked speculatively for every candidate - the vendor's Speculative fan-out pattern (https://docs.typesafe.ai/patterns/fan-out). **original, executed:**

```python
def read_invoice(document: str) -> dict:
    """One request instead of five: both picks, the currency, and a credit Noul for every
    candidate (speculative fan-out - code reads only the Nouls of the amounts it picked)."""
    amounts = find(MONEY_RE, document)
    spans = {a: None for a in amounts} | {NONE: "None of these is the requested value."}
    questions = {
        "total": Choice(instructions="Which amount is the total the customer must pay?", criteria=spans),
        "credit": Choice(instructions="Which amount is the courtesy credit that was applied?", criteria=spans),
        "currency": Choice(instructions="What currency are these amounts in?",
                           criteria={c: None for c in ["USD", "EUR", "GBP", "JPY", "CAD"]}),
    } | {
        f"is_credit_{i}": Noul(instructions=f"Is the amount {a} a credit or refund to the customer, not a charge?")
        for i, a in enumerate(amounts)
    }
    answers = ts.system_one(state=document, questions=questions, model=TYPESAFE_MODEL).answers
    out = {"currency": answers["currency"].choice}
    for role in ("total", "credit"):
        span = answers[role].choice
        if span == NONE:
            out[role] = None  # the hatch: nothing to normalize
            continue
        p_credit = answers[f"is_credit_{amounts.index(span)}"].noul
        sign = -1 if p_credit > 0.5 else 1
        out[role] = (sign * to_decimal(span), answers[role].confidence)
    return out


before = len(sent)
print(read_invoice(MONEY_DOC), "| requests:", len(sent) - before)
```

```text
{'currency': 'USD', 'total': (Decimal('1315.50'), 0.99), 'credit': (Decimal('-50.00'), 0.99)} | requests: 1
```

Pricing is per input token and the state is sent once per request (https://docs.typesafe.ai/models, 2026-10-03), so batching cuts both cost and round trips. Note this variant also handles the `none` hatch, which the cookbook code does not (pitfall 1).

### Published results

`receipt -> dana.personal@gmail.com (0.98)` (the body overrides the `To:` billing alias, so the answer depends on reading prose, not headers), `sender -> dana.whit@acme-corp.com (1.00)`, `mobile -> (415) 555-0177 (1.00)`, `country -> US (0.90)`, `E.164 -> +14155550177`, total `$1,315.50` with P(credit) 0.01, credit `$50.00` with P(credit) 0.99 (model `jev-1.12`; no cost published).

The page's two stated limits: a Choice allows at most 255 options (narrow in two stages - section, then span); and finding candidates is the hard part - names have no regex, so their candidates come from a roster, an NER model or an LLM.

### Pitfalls

**original, executed** probes:

```python
# 1. The `none` hatch reaches the normalizer unless code stops it.
try:
    phonenumbers.parse(NONE, "US")
except phonenumbers.NumberParseException as exc:
    print("parse('none') ->", type(exc).__name__)

# 2. to_decimal() is US-format only.
print("to_decimal('€1.315,50') ->", to_decimal("€1.315,50"))

# 3. The recall-tuned PHONE_RE also matches dates and order numbers.
print(find(PHONE_RE, "Order 2026-07-30, ref 4417-2209-881, call +44 20 7946 0958"))

# 4. MONEY_RE needs a currency symbol in front.
print(find(MONEY_RE, "Total: 1,315.50 USD; deposit EUR 200; fee $15"))
```

```text
parse('none') -> NumberParseException
to_decimal('€1.315,50') -> 1.31550
['2026-07-30', '4417-2209-881', '+44 20 7946 0958']
['$15']
```

1. **The `none` hatch is not handled downstream.** If `pick` returns `none`, `phonenumbers.parse("none", ...)` raises `NumberParseException` and `to_decimal("none")` raises. Check `choice == NONE` (and the confidence) before normalizing.
2. **`to_decimal` is US-format only**: `€1.315,50` parses as `1.31550`. The page itself says to ask a Noul which convention the document uses and branch in code.
3. **Recall-tuned regexes over-find on purpose** - `PHONE_RE` also matches `2026-07-30` and `4417-2209-881`. That is fine (the Choice rejects them) as long as the candidate list stays under 255 and the extra spans do not crowd the real one; it is not fine if any code path trusts `find()` without the pick.
4. **`MONEY_RE` requires a leading currency symbol**: `1,315.50 USD` and `EUR 200` are invisible. Candidates the regex misses can never be picked.
5. **`classify` has no escape option.** The country list is `US, GB, DE, FR, CA, AU`; a Toronto office forced into it is fine, a Milan office is not. Add `other` (the Choice primitive page recommends a catch-all whenever the list might not cover every input).
6. **Option order = document order**, so the first header address is always option 1. Check important picks with a rotated candidate list (jaggedness item 8).
7. **The SDK does not enforce the 255-option limit client-side**: `Choice(criteria={... 256 keys ...})` constructs without error on 0.7.2 (checked 2026-10-03), and the OpenAPI spec declares no `maxProperties`. Enforce the cap yourself before the request.

---

## Recipe 3: Structure recovery

Source: https://docs.typesafe.ai/cookbooks/autoformat (difficulty "Beginner").

### Problem

Plain text that lost its markup - hard-wrapped mid-sentence, no heading markers, no bullets - must come back as Markdown without the risk a rewriting LLM brings (changing words). Here *"the model never generates text"*; it answers narrow questions and code renders, so every output character comes from the input.

Two requests in sequence:

- **Pass 1, stitch**: one Noul per adjacent line pair (pairs across a blank line skipped): did this line break split a sentence?
- **Pass 2, classify**: one Choice per merged block (heading, paragraph, list item, quote, code, callout) plus companion questions (heading level, step order, callout kind) asked up front for every block. The blocks do not exist until pass 1 answers, which is why this is the rare recipe that needs a second request.
- **Direct evidence stays in code**: blank lines (and, per the prose, explicit markers) are read in code, not sent to the model.

### Input preparation

**condensed, executed** (setup; `cooksafe`/`IPython`/`json_cache` lines removed):

```python
import os
import re
import urllib.request
from pathlib import Path
from time import perf_counter

from typesafe_sdk import Choice, Noul, NoulCriteria, TypeSafeClient

TYPESAFE_MODEL = "jev-1.12"
PRICE = (0.042, 0.00)  # $ per 1M tokens (input, output); TypeSafe jev-1.12 as of 2026-09
client = TypeSafeClient(api_key=os.environ["TYPESAFE_API_KEY"], timeout=120.0)
```

**condensed, executed** (`@json_cache` removed, final `print` dropped; the fetch is a plain HTTPS GET of a pinned public gist and was run live):

```python
GIST = (
    "https://gist.githubusercontent.com/eugene-shvarts/6df7daf97233bf92bcdd6b386a0fa561"
    "/raw/5da03690611fb6ddcbaabdb91fb9f91d9751b113/build-memo.txt"
)


def fetch_document(url: str) -> str:
    request = urllib.request.Request(url, headers={"User-Agent": "typesafe-cookbook/1.0"})
    with urllib.request.urlopen(request) as response:
        return response.read().decode()


RAW = fetch_document(GIST)
```

**verbatim, executed** - line ids are plain text in the state; questions refer to lines by them:

```python
def to_lines(text: str) -> list[dict]:
    lines, gap = [], False
    for raw in text.split("\n"):
        stripped = re.sub(r"[\t ]+", " ", raw).strip()
        if not stripped:
            gap = bool(lines)  # a leading blank is not a break
            continue
        lines.append({"text": stripped, "gap": gap})
        gap = False
    return lines


def tag(items: list[dict], prefix: str) -> str:
    return "\n".join(
        f"{chr(10) if item['gap'] else ''}{prefix}{i:03d}| {item['text']}"
        for i, item in enumerate(items)
    )


def line_id(i: int) -> str:
    return f"L{i:03d}"


def block_id(i: int) -> str:
    return f"B{i:03d}"


LINES = to_lines(RAW)
print(f"{len(LINES)} non-blank lines. The model sees, e.g.:")
print("\n".join(tag(LINES, "L").splitlines()[19:24]))
```

```text
28 non-blank lines. The model sees, e.g.:
L013| The cutover touches three teams, so check whether you are on this
L014| list before you plan anything for Monday:
L015| The platform team
L016| The web client team
L017| Whoever still owns the release tooling
```

### Pass 1: stitch

**condensed, executed** (`@json_cache` removed):

```python
def join_question(i: int) -> Noul:
    return Noul(
        instructions=f"Does line {line_id(i)} pick up mid-sentence, continuing a sentence left unfinished at the end of line {line_id(i - 1)}?",
        criteria=NoulCriteria(
            true="The line starts in the middle of a sentence that began on the previous line - the line break tore the sentence apart",
            false="The line begins a new sentence, item, heading, or thought of its own",
        ),
    )


def stitch(wording: str = "mid-sentence") -> dict:
    make = join_question if wording == "mid-sentence" else naive_join_question
    questions = {line_id(i): make(i) for i in range(1, len(LINES)) if not LINES[i]["gap"]}
    started = perf_counter()
    response = client.system_one(
        state=tag(LINES, "L"), questions=questions, model=TYPESAFE_MODEL
    )
    return {
        "joins": [
            response.answers[line_id(i)].noul if line_id(i) in response.answers else 0.0
            for i in range(len(LINES))
        ],
        "seconds": round(perf_counter() - started, 2),
        "usage": [response.usage.input_tokens, response.usage.output_tokens],
    }


result = stitch()
print(f"{sum(1 for l in LINES if not l['gap']) - 1} pair questions, one request, "
      f"{result['seconds']}s")
```

**verbatim, executed** - the merge rule, with a punctuation-dependent bar:

```python
JOIN_AFTER_DANGLING, JOIN_AFTER_TERMINAL = 0.2, 0.5


def ends_terminal(text: str) -> bool:
    return re.search(r'[.!?:;…]["\')\]]*$', text) is not None


def merge(joins: list[float]) -> list[dict]:
    blocks = []
    for i, line in enumerate(LINES):
        bar = (
            JOIN_AFTER_TERMINAL
            if i and ends_terminal(LINES[i - 1]["text"])
            else JOIN_AFTER_DANGLING
        )
        if blocks and not line["gap"] and joins[i] >= bar:
            blocks[-1]["text"] += " " + line["text"]
            blocks[-1]["lines"].append(i)
        else:
            blocks.append({"text": line["text"], "lines": [i], "gap": line["gap"]})
    return blocks


blocks = merge(result["joins"])
healed = len(LINES) - len(blocks)
print(f"{len(LINES)} lines -> {len(blocks)} blocks ({healed} line breaks healed)")
for i, block in enumerate(blocks):
    n = len(block["lines"])
    print(f"{block_id(i)}  {n} line{'s' if n > 1 else ' '}  {block['text'][:62]}")
```

```text
28 lines -> 17 blocks (11 line breaks healed)
B000  1 line   Migration to the new build system
B001  4 lines  Hi everyone, quick heads up about the build system migration t
B002  1 line   What changes for you
B003  3 lines  The old make targets keep working until the end of the month. 
B004  1 line   bun run build
B005  3 lines  Generated artifacts no longer need to be committed. The new pi
B006  2 lines  The cutover touches three teams, so check whether you are on t
B007  1 line   The platform team
B008  1 line   The web client team
B009  1 line   Whoever still owns the release tooling
B010  1 line   Things to do before Monday
B011  1 line   Update your local toolchain to version 2.4 or later
B012  1 line   Delete the old build cache directory
B013  1 line   Run the doctor script and fix anything it flags
B014  3 lines  If the doctor script reports a red result on the toolchain che
B015  2 lines  As Dana put it in the kickoff, "a migration nobody notices is 
B016  1 line   Thanks, and shout if anything looks off.
```

Why two bars (published appendix): true continuations scored as low as 0.39 (`L004| make the switch for real.`), so a single 0.5 cutoff would break paragraphs; but `L015| The platform team` follows a colon and scored 0.22 - a list item that a single 0.2 cutoff would glue onto the sentence introducing it. Checking whether the previous line ends in `.` `!` `?` `:` `;` (code can read that exactly) separates the two bands.

### Pass 2: classify

**verbatim, executed** - per the page, these dicts plus the step criteria are *the entire specification of the classifier*:

```python
TYPE_CRITERIA = {
    "heading": "A short label or title that names the document or the section that follows it - not a full sentence of content",
    "paragraph": "Running prose: one or more complete sentences of explanatory or narrative text",
    "list_item": "One entry in a list of parallel items - an ingredient, a feature, a task, an attendee; reads as one of several sibling entries",
    "quote": "Words attributed to a person or source - quoted speech, a citation, an excerpt someone else wrote",
    "code": "Computer code, a shell command, terminal output, or a config snippet meant to be read verbatim",
    "callout": "A warning, tip, or important note that interrupts the flow to flag something the reader must not miss",
}
HLEVEL_CRITERIA = {
    "title": "The title of the whole document",
    "section": "A major section heading within the document",
    "subsection": "A minor heading nested under a section",
}
CALLOUT_CRITERIA = {
    "note": "Neutral extra information the reader should be aware of",
    "tip": "A helpful suggestion or shortcut that makes things easier",
    "warning": "A caution about something that can go wrong or cause harm",
}
```

**condensed, executed** (`@json_cache` removed). Note `HEADING_MAX_CHARS`: blocks longer than 90 characters get no heading-level question:

```python
HEADING_MAX_CHARS = 90  # longer blocks can't render as headings, so don't ask


def classify_questions(texts: list[str]) -> dict:
    questions = {}
    for i, text in enumerate(texts):
        bid = block_id(i)
        questions[f"type_{bid}"] = Choice(
            instructions=f"What kind of content is block {bid}?", criteria=TYPE_CRITERIA
        )
        if len(text) <= HEADING_MAX_CHARS:
            questions[f"hlevel_{bid}"] = Choice(
                instructions=f"As a heading, what level would block {bid} occupy in this document's structure?",
                criteria=HLEVEL_CRITERIA,
            )
        questions[f"step_{bid}"] = Noul(
            instructions=f"Is block {bid} an instruction in a sequence where the order of the items matters?",
            criteria=NoulCriteria(
                true="It is one step of a procedure - the items around it must happen in order",
                false="Order is irrelevant - it is a loose collection, or not a list item at all",
            ),
        )
        questions[f"callout_{bid}"] = Choice(
            instructions=f"What kind of aside is block {bid}?", criteria=CALLOUT_CRITERIA
        )
    return questions


def classify(texts: list[str], gaps: list[bool]) -> dict:
    tagged = tag([{"text": t, "gap": g} for t, g in zip(texts, gaps)], "B")
    questions = classify_questions(texts)
    started = perf_counter()
    response = client.system_one(state=tagged, questions=questions, model=TYPESAFE_MODEL)
    judgments = []
    for i in range(len(texts)):
        bid = block_id(i)
        type_answer = response.answers[f"type_{bid}"]
        hlevel = response.answers.get(f"hlevel_{bid}")
        judgments.append(
            {
                "type": type_answer.choice,
                "confidence": type_answer.confidence,
                "probabilities": type_answer.probabilities,
                "hlevel": hlevel.choice if hlevel else "section",
                "step": response.answers[f"step_{bid}"].noul,
                "callout": response.answers[f"callout_{bid}"].choice,
            }
        )
    return {
        "judgments": judgments,
        "n_questions": len(questions),
        "seconds": round(perf_counter() - started, 2),
        "usage": [response.usage.input_tokens, response.usage.output_tokens],
    }


classified = classify([b["text"] for b in blocks], [b["gap"] for b in blocks])
for block, judgment in zip(blocks, classified["judgments"]):
    block.update(judgment)
print(f"{classified['n_questions']} questions about {len(blocks)} blocks, one request, "
      f"{classified['seconds']}s\n")
print(f"{'block':<6}{'type':<11}{'conf':<6}{'companion used':<18}text")
for i, b in enumerate(blocks):
    companion = {
        "heading": f"level={b['hlevel']}",
        "list_item": f"step={b['step']:.2f}",
        "callout": f"kind={b['callout']}",
    }.get(b["type"], "-")
    print(f"{block_id(i):<6}{b['type']:<11}{b['confidence']:.2f}  {companion:<18}"
          f"{b['text'][:46]}")
```

Question count: 17 `type` + 17 `step` + 17 `callout` + 11 `hlevel` (only blocks of at most 90 characters) = 62 in one request. The page's argument for asking companions speculatively: *"the state is most of the tokens and is sent once either way, while an extra round trip adds a full request of latency."*

### Rendering

**verbatim, executed:**

````python
STEP_THRESHOLD = 0.5
HEADING_MARK = {"title": "#", "section": "##", "subsection": "###"}
CALLOUT_MARK = {"note": "NOTE", "tip": "TIP", "warning": "WARNING"}


def to_markdown(blocks: list[dict]) -> str:
    groups = []
    for b in blocks:
        if b["type"] in ("list_item", "code") and groups and groups[-1][0] == b["type"]:
            groups[-1][1].append(b)
        else:
            groups.append((b["type"], [b]))
    parts = []
    for kind, items in groups:
        if kind == "list_item":
            ordered = sum(b["step"] for b in items) / len(items) >= STEP_THRESHOLD
            parts.append("\n".join(
                f"{n + 1}. {b['text']}" if ordered else f"- {b['text']}"
                for n, b in enumerate(items)
            ))
        elif kind == "code":
            parts.append("```\n" + "\n".join(b["text"] for b in items) + "\n```")
        elif kind == "heading":
            parts.append(f"{HEADING_MARK[items[0]['hlevel']]} {items[0]['text']}")
        elif kind == "quote":
            parts.append(f"> {items[0]['text']}")
        elif kind == "callout":
            parts.append(f"> [!{CALLOUT_MARK[items[0]['callout']]}]\n> {items[0]['text']}")
        else:
            parts.append(items[0]["text"])
    return "\n\n".join(parts) + "\n"


markdown = to_markdown(blocks)
print(markdown)
````

### Offline run

**original, executed** - canned answers from the published appendix and table. Run this cell right after the setup cell, before pass 1 (it replaces `client`):

```python
import re as _re

from harness import choice_answer, fake_client, noul_answer

# Pass 1: published "mid-sentence" join probabilities (appendix table, lines 2-17 and 20-21);
# lines 23, 24 and 26 are not in the published table, so their values are synthetic.
MID = {2: 0.77, 3: 0.62, 4: 0.39, 7: 0.42, 8: 0.59, 11: 0.48, 12: 0.40, 14: 0.50,
       15: 0.22, 16: 0.11, 17: 0.12, 20: 0.08, 21: 0.05, 23: 0.90, 24: 0.90, 26: 0.90}
# Same-paragraph wording: published for 15, 16, 17, 20, 21; the rest reuse MID (synthetic).
NAIVE = MID | {15: 0.77, 16: 0.81, 17: 0.78, 20: 0.88, 21: 0.91}

# Pass 2: published type, confidence and companion per block. B006 carries the published
# top three probabilities; the remaining 0.04 is spread over the other three types.
TYPES = ["heading", "paragraph", "heading", "paragraph", "code", "paragraph", "paragraph",
         "list_item", "list_item", "list_item", "heading", "list_item", "list_item",
         "list_item", "callout", "quote", "paragraph"]
CONF = [0.99, 0.98, 0.75, 0.89, 1.00, 0.90, 0.43, 0.99, 1.00, 0.99, 0.96, 0.98, 0.99,
        0.92, 0.65, 0.99, 0.92]
HLEVEL = {0: "title", 2: "section", 10: "section"}
STEP = {7: 0.15, 8: 0.16, 9: 0.12, 11: 0.86, 12: 0.87, 13: 0.90}


def af_answers(state, questions):
    out = {}
    for qid, q in questions.items():
        if qid.startswith("L"):  # pass 1
            table = MID if "mid-sentence" in q["instructions"] else NAIVE
            out[qid] = noul_answer(table[int(qid[1:])])
            continue
        kind, i = qid.split("_B")[0], int(qid.split("_B")[1])
        options = list(q["criteria"])
        if kind == "type" and i == 6:
            p = {"paragraph": 0.53, "list_item": 0.24, "callout": 0.19}
            p |= {o: 0.04 / 3 for o in options if o not in p}
            out[qid] = {"type": "choice", "choice": "paragraph", "confidence": 0.43,
                        "probabilities": p}
        elif kind == "type":
            out[qid] = choice_answer(options, TYPES[i], CONF[i])
        elif kind == "hlevel":
            out[qid] = choice_answer(options, HLEVEL.get(i, "section"), 0.9)
        elif kind == "step":
            out[qid] = noul_answer(STEP.get(i, 0.1))
        else:  # callout kind
            out[qid] = choice_answer(options, "warning" if i == 14 else "note", 0.9)
    return out


client, sent = fake_client(af_answers)
```

With those answers, every printed table in the offline run is byte-identical to the published one except wall-clock time (`0.0s` offline versus 0.32 s / 0.51 s published), and the rendered Markdown equals the published Markdown exactly (the page's rendered output block saved verbatim as `published_autoformat.md`). **original, executed:**

```python
expected = Path("published_autoformat.md").read_text(encoding="utf-8")  # the page's output, verbatim
print("rendered == published:", markdown == expected)
print("requests:", len(sent), "| questions per request:", [len(b["questions"]) for b in sent])
```

```text
rendered == published: True
requests: 2 | questions per request: [16, 62]
```

(Run after the "same paragraph" ablation cell below, it prints `requests: 3 | questions per request: [16, 62, 16]`; the third request is the ablation.)

### Wording ablation: "mid-sentence" versus "same paragraph"

**condensed, executed** (`@json_cache` removed):

```python
def naive_join_question(i: int) -> Noul:
    return Noul(
        instructions=f"Are lines {line_id(i - 1)} and {line_id(i)} part of the same paragraph?",
        criteria=NoulCriteria(
            true="The two lines belong to the same paragraph of running text",
            false="The two lines belong to different paragraphs or different pieces of content",
        ),
    )


naive = stitch("same-paragraph")
print(f"{'':14}{'mid-sentence':>13}{'same paragraph':>16}")
for i in (15, 16, 17, 20, 21):
    print(f"{line_id(i)}{'':2}{LINES[i]['text'][:36]:<38}"
          f"{result['joins'][i]:>7.2f}{naive['joins'][i]:>13.2f}")
print(f"\nblocks after merge: {len(blocks)} (mid-sentence) vs "
      f"{len(merge(naive['joins']))} (same paragraph)")
```

```text
               mid-sentence  same paragraph
L015  The platform team                        0.22         0.77
L016  The web client team                      0.11         0.81
L017  Whoever still owns the release tooli     0.12         0.78
L020  Delete the old build cache directory     0.08         0.88
L021  Run the doctor script and fix anythi     0.05         0.91

blocks after merge: 17 (mid-sentence) vs 12 (same paragraph)
```

"Same paragraph" asks whether the topic carries over - and between list items it does, so every unmarked list item scores above 0.75 and both lists collapse. The page's rule: *"When a judgment call feeds a threshold, the question should name the narrowest fact that decides it."*

### The least certain block

**verbatim, executed:**

```python
uncertain = min(blocks, key=lambda b: b["confidence"])
print(f'"{uncertain["text"]}"')
print(f"confidence {uncertain['confidence']:.2f}: ", end="")
print(", ".join(f"{k} {v:.2f}" for k, v in
                sorted(uncertain["probabilities"].items(), key=lambda kv: -kv[1])[:3]))
```

```text
"The cutover touches three teams, so check whether you are on this list before you plan anything for Monday:"
confidence 0.43: paragraph 0.53, list_item 0.24, callout 0.19
```

The page suggests flagging for review any block whose type confidence is under 0.55 (but see the note on what "type confidence" means there, in the confidence section above).

### Published results and cost

From the page (model `jev-1.12`; price constant `PRICE = (0.042, 0.00)` commented "TypeSafe jev-1.12 as of 2026-09"): 28 lines -> 17 blocks (11 breaks healed); pass 1 = 16 questions in 0.32 s, pass 2 = 62 questions in 0.51 s; total **10,211 tokens, 0.8 s**. Cost is printed two ways on the same page: the cost cell's output says **$0.0003** and the prose says **$0.0015**. Checked on 2026-10-03: at $0.042 per million input tokens with free output (https://docs.typesafe.ai/models), even counting all 10,211 tokens as input gives $0.000429, so $0.0015 cannot follow from the page's own price constant; $0.0003 (rounded to four decimals) is consistent with roughly 6,000-8,300 of the tokens being input. Treat $0.0003 as the computed figure and the prose figure as stale.

### Pitfalls

1. **The renderer groups only `list_item` and `code`.** Two lists back to back with no heading or paragraph between them merge into one list, numbered or bulleted by their *joint* mean step probability.
2. **Explicit markers are not actually read by the published code.** The prose says markers (`- `, `1.`, `#`) stay in code, but `to_lines` only normalizes whitespace and tracks blank lines; this memo had no markers. A document with surviving markers needs a pre-pass that strips and records them.
3. **Unescaped output.** A paragraph that starts with `#` or `1.` renders as markup. Escape block text in `to_markdown` if inputs can contain such lines.
4. **Thresholds are tuned on one memo.** The join bars 0.2 / 0.5 rest on the 13 join probabilities the page publishes (11 in the appendix table, 2 more in the wording ablation; pass 1 asked 16), and the 0.5 step bar on six list-item step probabilities. Re-derive them on a labelled sample of your documents.
5. **Missing companion answers.** `hlevel` defaults to `section` for blocks over 90 characters; a long block classified `heading` silently becomes a level-2 heading.
6. **Cost note**: 62 questions about 17 blocks is fine, but companion questions grow linearly with blocks while the 64k-token request budget covers state + all questions (https://docs.typesafe.ai/models). Very long documents need chunking.

---

## Recipe 4: Double-checking citations

Source: https://docs.typesafe.ai/cookbooks/citation_check (difficulty "Beginner"; *"Numbers below came from `jev-1.12` on 2026-08-16."*).

### Problem

An LLM answers with citations: claim + quote + section. Some are wrong in two distinct ways - the quote is **not in the source** (fabricated), or it is there word for word but **its context does not back the claim** (contradicted or unsupported). `check_citation()` returns one of `verified`, `unsupported`, `contradicted`, `fabricated`, plus a confidence that routes to a human.

Division of labour: the existence check is an ordinary normalized substring match - no model; only the relation between section and claim is a judgement.

### Loading the source

**condensed, executed** (setup; `cooksafe`/`IPython`/`json_cache` lines removed). Note this page reads `TYPESAFE_ENDPOINT`, where the other cookbooks read `TYPESAFE_BASE_URL`. In practice `TYPESAFE_BASE_URL` still works here: when `base_url` is `None`, `typesafe-sdk` 0.7.2 falls back to the `TYPESAFE_BASE_URL` variable itself (checked offline on 2026-10-03 with a mock transport: `base_url=None` sent the request to the `TYPESAFE_BASE_URL` host). `TYPESAFE_ENDPOINT` works only on this page:

```python
import json
import os
import re
from pathlib import Path
from time import perf_counter

from typesafe_sdk import Choice, TypeSafeClient

TYPESAFE_MODEL = "jev-1.12"
AUTO_ACCEPT = 0.8  # start high for more human review as you build trust in the model

client = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"),
    base_url=os.environ.get("TYPESAFE_ENDPOINT"),
    timeout=120.0,
)
```

**verbatim, executed** against RFC 7519 fetched from https://www.rfc-editor.org/rfc/rfc7519.txt on 2026-10-03. The first output line matches the published `58,365 characters, 45 numbered sections` exactly:

```python
def load_source() -> str:
    """RFC 7519 verbatim, minus the page headers and footers that interrupt its paragraphs."""
    lines = []
    for line in Path("rfc7519.txt").read_text().splitlines():
        bare = line.lstrip("\f")
        if re.match(r"Jones, et al\.\s.*\[Page \d+\]$", bare):
            continue
        if re.match(r"RFC 7519\s+JSON Web Token \(JWT\)\s+May 2015$", bare):
            continue
        lines.append(bare)
    return re.sub(r"\n{3,}", "\n\n", "\n".join(lines))


def split_sections(source: str) -> dict[str, str]:
    """Map each numbered section ("4.1.3") to its text, split on the RFC's header lines."""
    boundary = re.compile(r"(?m)^(?:(\d+(?:\.\d+)*)\.  .+|Appendix [A-Z]\..*)$")
    marks = list(boundary.finditer(source))
    sections = {}
    for mark, nxt in zip(marks, marks[1:] + [None]):
        if mark.group(1) is None:  # an appendix header only terminates the section before it
            continue
        sections[mark.group(1)] = source[mark.start() : nxt.start() if nxt else len(source)].strip()
    return sections


SOURCE = load_source()
SECTIONS = split_sections(SOURCE)
CITATIONS = json.loads(Path("citations.json").read_text())

print(f"{len(SOURCE):,} characters, {len(SECTIONS)} numbered sections, {len(CITATIONS)} citations")
print("\nA citation with a quote:")
print(json.dumps(CITATIONS[1], indent=2))
print("\nA claim-only citation:")
print(json.dumps(next(c for c in CITATIONS if c["quote"] is None), indent=2))
```

```text
58,365 characters, 45 numbered sections, 8 citations

A citation with a quote:
{
  "id": "aud_reject",
  "claim": "If a validator does not find itself in a token's audience list, it has to reject the token.",
  "quote": "If the principal processing the claim does not identify itself with a value in the \"aud\" claim when this claim is present, then the JWT MUST be rejected.",
  "section": "4.1.3"
}

A claim-only citation:
{
  "id": "iat_future",
  "claim": "The \"iat\" claim requires validators to reject tokens whose issue time is in the future.",
  "quote": null,
  "section": "4.1.6"
}
```

`citations.json` is not published. Only `aud_reject` and `iat_future` are printed on the page; the other six citations used offline were written for this document against the same sections, with the page's ids. With them, step 1 below reproduced the published section sizes exactly, confirming which RFC section each published citation pointed at: `epoch_seconds` -> 2 (Terminology), `clock_skew` and `exp_required` -> 4.1.4, `pii_encryption` -> 3 (JWT Overview), `duplicate_names` -> 4 (JWT Claims).

### Step 1: find the quote (code only)

**verbatim, executed:**

```python
def normalize(text: str) -> str:
    """Collapse whitespace and fold curly quotes, so a quote matches across line wraps."""
    table = str.maketrans({"“": '"', "”": '"', "‘": "'", "’": "'"})
    return re.sub(r"\s+", " ", text.translate(table)).strip()


def find_quote(sections: dict[str, str], quote: str) -> str | None:
    """The number of the section that contains the quote verbatim, or None."""
    needle = normalize(quote)
    for number in sorted(sections, key=lambda n: [int(p) for p in n.split(".")]):
        if needle in normalize(sections[number]):
            return number
    return None


def locate(sections: dict[str, str], citation: dict) -> tuple[str, str | None]:
    """Step 1 for one citation: a status, plus the section step 2 will read."""
    if citation["quote"] is None:
        return "section-only", sections[citation["section"]]
    number = find_quote(sections, citation["quote"])
    if number is None:
        return "missing", None
    return "found", sections[number]


for citation in CITATIONS:
    status, section = locate(SECTIONS, citation)
    where = f"section of {len(section):,} chars" if section else "not in the source"
    print(f"{citation['id']:<18}{status:<14}{where}")
```

```text
epoch_seconds     found         section of 3,122 chars
aud_reject        found         section of 761 chars
sig_reporting     missing       not in the source
clock_skew        found         section of 529 chars
exp_required      found         section of 529 chars
pii_encryption    found         section of 1,653 chars
iat_future        section-only  section of 270 chars
duplicate_names   found         section of 918 chars
```

### Step 2: one Choice on the relation

**condensed, executed** (`@json_cache` removed). The state is an **object** `{"claim": ..., "section": ...}` - the section is the evidence, scoped to the one the quote lives in:

```python
QUESTIONS = {
    "relation": Choice(
        instructions="How does the section relate to the claim?",
        criteria={
            "supports": "The section states the claim or directly implies that it is true",
            "contradicts": "The section states the opposite of the claim or implies it is false",
            "says_nothing": "The section does not address what the claim asserts, either way",
        },
    ),
}

RELATION_TO_VERDICT = {
    "supports": "verified",
    "contradicts": "contradicted",
    "says_nothing": "unsupported",
}


def ask(claim: str, section: str) -> dict:
    started = perf_counter()
    response = client.system_one(
        state={"claim": claim, "section": section},
        questions=QUESTIONS,
        model=TYPESAFE_MODEL,
    )
    answer = response.answers["relation"]
    return {
        "choice": answer.choice,
        "probabilities": answer.probabilities,
        "confidence": answer.confidence,
        "seconds": round(perf_counter() - started, 2),
        "input_tokens": response.usage.input_tokens or 0,
        "output_tokens": response.usage.output_tokens or 0,
    }


def verdict(status: str, answer: dict | None) -> dict:
    """Fold step 1 and step 2 into one of the four labels, plus an auto-or-review flag."""
    if status == "missing":
        # confidence None: no model was called, so there is no model confidence to report
        return {"verdict": "fabricated", "confidence": None, "auto": True}
    return {
        "verdict": RELATION_TO_VERDICT[answer["choice"]],
        "confidence": answer["confidence"],
        "auto": answer["confidence"] >= AUTO_ACCEPT,
    }


def check_citation(sections: dict[str, str], citation: dict) -> dict:
    status, section = locate(sections, citation)
    answer = ask(citation["claim"], section) if section is not None else None
    return {"id": citation["id"], "status": status, "answer": answer, **verdict(status, answer)}
```

**original, executed** - canned relation answers (published choice and confidence per citation):

```python
from harness import choice_answer, fake_client

# The published relation and confidence per citation id, looked up by claim text.
RELATION = {"epoch_seconds": ("supports", 0.93), "aud_reject": ("supports", 0.95),
            "clock_skew": ("supports", 0.99), "exp_required": ("contradicts", 0.99),
            "pii_encryption": ("says_nothing", 0.27), "iat_future": ("says_nothing", 0.56),
            "duplicate_names": ("supports", 0.99)}


def cc_answers(state, questions):
    cid = next(c["id"] for c in CITATIONS if c["claim"] == state["claim"])
    options = list(questions["relation"]["criteria"])
    return {"relation": choice_answer(options, *RELATION[cid])}


client, sent = fake_client(cc_answers)
```

**verbatim, executed** - output byte-identical to the published table:

```python
print(f"{'citation':<18}{'quote':<14}{'relation':<14}{'conf':>6}  {'verdict':<13}{'action':>7}")
for citation in CITATIONS:
    result = check_citation(SECTIONS, citation)
    answer = result["answer"]
    relation = answer["choice"] if answer else "-"
    conf = f"{answer['confidence']:.2f}" if answer else "-"
    action = "auto" if result["auto"] else "review"
    print(
        f"{result['id']:<18}{result['status']:<14}{relation:<14}{conf:>6}"
        f"  {result['verdict']:<13}{action:>7}"
    )
```

```text
citation          quote         relation        conf  verdict       action
epoch_seconds     found         supports        0.93  verified        auto
aud_reject        found         supports        0.95  verified        auto
sig_reporting     missing       -                  -  fabricated      auto
clock_skew        found         supports        0.99  verified        auto
exp_required      found         contradicts     0.99  contradicted    auto
pii_encryption    found         says_nothing    0.27  unsupported   review
iat_future        section-only  says_nothing    0.56  unsupported   review
duplicate_names   found         supports        0.99  verified        auto
```

### Published results

The four accurate citations came back `verified` at 0.93 or higher; `sig_reporting` never reached the model (`fabricated` by string match); `exp_required` quotes 4.1.4 word for word, but the same section says *"Use of this claim is OPTIONAL"*, so `contradicted` at 0.99; `pii_encryption` (quote present, section silent on the claim) and `iat_future` (no quote) came back `unsupported` at 0.27 and 0.56, both sent to a human. All four planted failures were caught. No cost or latency totals are published (the `ask` function records per-call seconds and tokens, but the page does not print them).

### Pitfalls

**original, executed** probes:

```python
# A quote found in a different section than the one cited is not flagged:
wrong_section = {"id": "x", "claim": CITATIONS[1]["claim"], "quote": CITATIONS[1]["quote"],
                 "section": "4.1.4"}
print(locate(SECTIONS, wrong_section)[0], "| quote lives in", find_quote(SECTIONS, wrong_section["quote"]))

# A claim-only citation naming a section that does not exist raises:
try:
    locate(SECTIONS, {"id": "y", "claim": "...", "quote": None, "section": "9.9"})
except KeyError as exc:
    print("KeyError", exc)

# A one-word truncation is reported as fabricated:
print(locate(SECTIONS, CITATIONS[1] | {"quote": CITATIONS[1]["quote"][:-1] + " immediately."})[0])

# What the model sees: the state is an object, not a string.
print(sorted(sent[0]["state"]), "| requests:", len(sent))
```

```text
found | quote lives in 4.1.3
KeyError '9.9'
missing
['claim', 'section'] | requests: 7
```

1. **A wrong section number is not flagged.** When the quote is found, `locate` uses the section that *contains* it and ignores `citation["section"]`. A citation that names 4.1.4 for a quote from 4.1.3 is judged against 4.1.3 and can come back `verified`. Compare the two if section accuracy matters.
2. **A claim-only citation naming a section that does not exist raises `KeyError`** instead of returning `fabricated`. Guard `sections[citation["section"]]`.
3. **Exact matching after normalization** - a truncated or lightly reworded quote is `fabricated`; the page says production needs fuzzy matching.
4. **`fabricated` carries `confidence=None` and `auto=True`.** Any code formatting confidences as floats must handle `None` (the page's table does).
5. **The parser is RFC-shaped.** `load_source` strips RFC page headers and `split_sections` splits on `N.N.  Title` lines; any other document needs its own sectioning.
6. **`AUTO_ACCEPT = 0.8` over 3 options means `p_max >= 0.867`** - a strict bar, which is the page's intent ("start high ... lower the threshold as you see how the model does").

---

## Recipe 5: Function calling

Source: https://docs.typesafe.ai/cookbooks/function_calling (difficulty "Intermediate").

### Problem

Turn a sentence ("plot rolling correlation between nvda and spy for the past month") into a call to an ordinary typed function, with no free-text arguments reaching the function. The functions are unchanged; their `Literal` / `list[Literal]` / `bool` hints already define closed sets, and each closed set becomes a question whose options are exactly the values the function accepts.

**verbatim, executed** (one of the ten functions; body elided on the page; needs `from typing import Literal`, imported in the tools cell below):

```python
def plot_price(
    symbol: Literal["SPY", "NVDA", "AMD", "AAPL", "MSFT", "TSLA"],
    style: Literal["line", "candles"] = "line",
    resolution: Literal["1m", "5m", "15m", "1h", "1d"] = "15m",
    window: Literal["1d", "1w", "1mo", "3mo"] = "1w",
    include_volume: bool = False,
    moving_average: Literal["9", "20", "50"] | None = None,
    log_scale: bool = False,
): ...
```

### How the published design works

From the page (the implementation lives in `dispatch.py` and `trader.py`, which are **not published**):

- `closed_sets(fn)` classifies arguments into **choice** (`Literal`), **set** (`list[Literal]`), **flag** (`bool`). Anything else - `int`, free text, numbers, dates - gets no question and keeps its default (`top_movers.limit` stays 3).
- A hand- or LLM-written `spec.json` holds, per argument, a `question`, `options` (keys are the function's literal strings, so no label mapping), and optionally `stated` - a yes/no question "does the command say anything about this argument?" When the answer is no, the argument is **omitted** and the function's default applies. Without it, *"the choice would have to name some window, and it would have named one confidently."*
- Set arguments get one question per member, with `{}` replaced by the member name.
- One request per command carries the function Choice (`__tool__`) **and every function's argument questions**: 54 questions for the cookbook's ten functions. Code reads only the chosen function's answers.
- A call's `confidence` is *"the least certain judgement in the call, rather than the product of all of them"*: a product falls as a function takes more arguments whether or not any one judgement is shaky.
- Question-writing advice: write about the idea, not the user's words ("is amd tracking nvidia lately" reaches `rolling_correlation` with neither word in the spec); never name a question after the parameter ("Which resolution?" gives nothing to match); when two arguments share a value set (`symbol`, `benchmark`), spell out the roles - *the one being measured, named first* versus *the second one named, the yardstick*.

The two `spec.json` entries the page prints, **verbatim**:

```json
{
  "style": {
    "question": "Does the user want a plain line or candles?",
    "stated": "Does the user say how the chart should be drawn, such as a line, candles, or OHLC bars?",
    "options": {
      "line": "a simple line through the closing prices",
      "candles": "a candlestick or OHLC chart, showing each bar's open, high, low and close"
    }
  }
}
{
  "moving_average": {
    "question": "How many bars should the moving average cover - nine, twenty, or fifty?",
    "stated": "Does the user ask for a moving average or a smoothed line over the candles?",
    "options": {
      "9": "a nine-bar moving average, a fast one",
      "20": "a twenty-bar moving average",
      "50": "a fifty-bar moving average, a slow one"
    }
  }
}
```

The question count is consistent with the published signatures: 1 route + 22 choice arguments + 5 flags + 6 set-member Nouls = 34 (assuming `compare_returns.symbols` ranges over the same six tickers, which the page implies but does not print), so the remaining 20 of the 54 are `stated` questions; which arguments carry one is not published.

### A runnable reconstruction

Because `dispatch.py` is not published, the following is **written for this document**, following the published spec format and question ids (`__tool__`, `fn.arg`, `fn.arg?`, `fn.set.MEMBER`). It is not the vendor's code. The four signatures other than `plot_price` are illustrative (their exact signatures are not published, except that `rolling_correlation` defaults to one month and hourly bars per the page).

**original, executed** - tools:

```python
import typing
from typing import Literal

Ticker = Literal["SPY", "NVDA", "AMD", "AAPL", "MSFT", "TSLA"]
Window = Literal["1d", "1w", "1mo", "3mo"]


def list_symbols(): ...


def rolling_correlation(
    symbol: Ticker,
    benchmark: Ticker,
    window: Window = "1mo",
    resolution: Literal["15m", "1h", "1d"] = "1h",
): ...


def compare_returns(symbols: list[Ticker], window: Window = "1mo", normalize: bool = True): ...


def top_movers(
    window: Window = "1d",
    direction: Literal["gainers", "losers"] = "gainers",
    limit: int = 3,
): ...


TOOLS = {f.__name__: f for f in (list_symbols, rolling_correlation, compare_returns, top_movers)}
```

**original, executed** - reading the closed sets from type hints (the `plot_price` line reproduces the page's published shapes for it, in order):

```python
def closed_sets(fn) -> dict[str, tuple[str, tuple]]:
    """arg -> (shape, values): 'choice' for Literal (or Literal | None), 'set' for
    list[Literal], 'flag' for bool. Anything else (int, str, date) gets no question."""
    shapes = {}
    for name, hint in typing.get_type_hints(fn).items():
        if name == "return":
            continue
        args = [a for a in typing.get_args(hint) if a is not type(None)]
        if typing.get_origin(hint) is typing.Literal:
            shapes[name] = ("choice", typing.get_args(hint))
        elif typing.get_origin(hint) is list and typing.get_origin(args[0]) is typing.Literal:
            shapes[name] = ("set", typing.get_args(args[0]))
        elif args and typing.get_origin(args[0]) is typing.Literal:  # Literal[...] | None
            shapes[name] = ("choice", typing.get_args(args[0]))
        elif hint is bool:
            shapes[name] = ("flag", (True, False))
    return shapes


for name, fn in TOOLS.items():
    print(f"{name:<20}", {a: s for a, (s, _) in closed_sets(fn).items()})
print(f"{'plot_price':<20}", {a: s for a, (s, _) in closed_sets(plot_price).items()})
```

```text
list_symbols         {}
rolling_correlation  {'symbol': 'choice', 'benchmark': 'choice', 'window': 'choice', 'resolution': 'choice'}
compare_returns      {'symbols': 'set', 'window': 'choice', 'normalize': 'flag'}
top_movers           {'window': 'choice', 'direction': 'choice'}
plot_price           {'symbol': 'choice', 'style': 'choice', 'resolution': 'choice', 'window': 'choice', 'include_volume': 'flag', 'moving_average': 'choice', 'log_scale': 'flag'}
```

**original, executed** - a spec in the published shape:

```python
WINDOWS = {"1d": "today", "1w": "this week", "1mo": "the past month", "3mo": "the past quarter"}

SPEC = {
    "route": "What is the user asking the trading assistant to do?",
    "functions": {
        "list_symbols": {
            "description": "list the tickers the assistant has data for",
            "arguments": {},
        },
        "rolling_correlation": {
            "description": "measure how closely one ticker moves with another over time",
            "arguments": {
                "symbol": {
                    "question": "Which ticker is being measured - the one named first?",
                    "options": {t: None for t in typing.get_args(Ticker)},
                },
                "benchmark": {
                    "question": "Which ticker is the yardstick - the second one named?",
                    "options": {t: None for t in typing.get_args(Ticker)},
                },
                "window": {
                    "question": "How far back should the comparison look?",
                    "stated": "Does the user say how far back to look?",
                    "options": WINDOWS,
                },
                "resolution": {
                    "question": "What bar size should the correlation use?",
                    "stated": "Does the user say what bar size to use?",
                    "options": {"15m": "fifteen-minute bars", "1h": "hourly bars", "1d": "daily bars"},
                },
            },
        },
        "compare_returns": {
            "description": "compare the returns of several tickers side by side",
            "arguments": {
                "symbols": {"question": "Does the user want {} in the comparison?"},
                "window": {
                    "question": "Over what period should returns be compared?",
                    "stated": "Does the user say over what period?",
                    "options": WINDOWS,
                },
                "normalize": {
                    "question": "Should every series start from the same base?",
                    "stated": "Does the user say whether to rebase the series?",
                },
            },
        },
        "top_movers": {
            "description": "list the tickers that moved most",
            "arguments": {
                "window": {
                    "question": "Over what period?",
                    "stated": "Does the user name a period?",
                    "options": WINDOWS,
                },
                "direction": {
                    "question": "Biggest gainers or biggest losers?",
                    "options": {"gainers": "rose the most", "losers": "fell the most"},
                },
            },
        },
    },
}
```

**original, executed** - questions, dispatch, weakest-judgement confidence. Each judgement is put on one 0-1 scale: Choice `confidence`, `|2p - 1|` for Nouls (the documented equivalent), and `1 - P(stated)` for an omitted argument:

```python
from dataclasses import dataclass, field

from typesafe_sdk import Choice, Noul

TYPESAFE_MODEL = "jev-1.13.0"
ROUTE = "__tool__"


def build_questions(spec: dict, tools: dict) -> dict:
    """Every function's questions, so one request covers whichever function is chosen."""
    questions = {
        ROUTE: Choice(
            instructions=spec["route"],
            criteria={name: f["description"] for name, f in spec["functions"].items()},
        )
    }
    for name, fn in tools.items():
        args = spec["functions"][name]["arguments"]
        for arg, (shape, values) in closed_sets(fn).items():
            a, qid = args[arg], f"{name}.{arg}"
            if shape == "choice":
                questions[qid] = Choice(instructions=a["question"], criteria=a["options"])
            elif shape == "flag":
                questions[qid] = Noul(instructions=a["question"])
            else:  # set: one yes/no per member
                for member in values:
                    questions[f"{qid}.{member}"] = Noul(instructions=a["question"].format(member))
            if "stated" in a:
                questions[f"{qid}?"] = Noul(instructions=a["stated"])
    return questions


@dataclass
class Call:
    name: str
    kwargs: dict
    certainty: dict = field(default_factory=dict)  # judgement -> 0..1, one scale for all

    @property
    def confidence(self) -> float:  # the weakest judgement, not the product
        return min(self.certainty.values())

    def __str__(self) -> str:
        return f"{self.name}({', '.join(f'{k}={v!r}' for k, v in self.kwargs.items())})"


def dispatch(command: str) -> Call:
    answers = client.system_one(state=command, questions=QUESTIONS, model=TYPESAFE_MODEL).answers
    tool = answers[ROUTE]
    call = Call(tool.choice, {}, {ROUTE: tool.confidence})
    for arg, (shape, values) in closed_sets(TOOLS[tool.choice]).items():
        qid = f"{tool.choice}.{arg}"
        stated = answers[f"{qid}?"].noul if f"{qid}?" in answers else 1.0
        if stated < 0.5:  # the command says nothing about it: the default stands
            call.certainty[arg] = 1 - stated
            continue
        if shape == "choice":
            call.kwargs[arg] = answers[qid].choice
            call.certainty[arg] = min(stated, answers[qid].confidence)
        elif shape == "flag":
            call.kwargs[arg] = answers[qid].noul >= 0.5
            call.certainty[arg] = min(stated, abs(2 * answers[qid].noul - 1))
        else:
            p = {m: answers[f"{qid}.{m}"].noul for m in values}
            call.kwargs[arg] = [m for m in values if p[m] >= 0.5]
            call.certainty[arg] = min(abs(2 * v - 1) for v in p.values())
    return call


QUESTIONS = build_questions(SPEC, TOOLS)
print(len(QUESTIONS), "questions per command")
```

```text
20 questions per command
```

**original, executed** - synthetic answers for three of the page's commands:

```python
from harness import choice_answer, fake_client, noul_answer

# Synthetic answers for three commands (the cookbook does not publish its raw answers).
SCRIPT = {
    "is amd tracking nvidia lately": {
        ROUTE: ("rolling_correlation", 0.82),
        "rolling_correlation.symbol": ("AMD", 0.84),
        "rolling_correlation.benchmark": ("NVDA", 0.90),
        "rolling_correlation.window?": 0.04,
        "rolling_correlation.resolution?": 0.01,
    },
    "compare nvda amd and msft over the past three months": {
        ROUTE: ("compare_returns", 0.97),
        "compare_returns.window?": 0.98,
        "compare_returns.window": ("3mo", 0.94),
        "compare_returns.normalize?": 0.03,
        **{
            f"compare_returns.symbols.{t}": (0.99 if t in ("NVDA", "AMD", "MSFT") else 0.02)
            for t in typing.get_args(Ticker)
        },
    },
    "what tickers do you have": {ROUTE: ("list_symbols", 1.00)},
}


def fc_answers(state, questions):
    script, out = SCRIPT[state], {}
    for qid, q in questions.items():
        value = script.get(qid)
        if q["type"] == "noul":
            out[qid] = noul_answer(0.05 if value is None else value)
        else:  # unscripted Choice: first option at confidence 0.5
            out[qid] = choice_answer(list(q["criteria"]), *(value or (next(iter(q["criteria"])), 0.5)))
    return out


client, sent = fake_client(fc_answers)
for command in SCRIPT:
    call = dispatch(command)
    weakest = min(call.certainty, key=call.certainty.get)
    print(f"{str(call):<62} confidence {call.confidence:.2f} (weakest: {weakest})")
print("requests:", len(sent), "| questions in each:", {len(b["questions"]) for b in sent})
```

```text
rolling_correlation(symbol='AMD', benchmark='NVDA')            confidence 0.82 (weakest: __tool__)
compare_returns(symbols=['NVDA', 'AMD', 'MSFT'], window='3mo') confidence 0.94 (weakest: window)
list_symbols()                                                 confidence 1.00 (weakest: __tool__)
requests: 3 | questions in each: {20}
```

### Published results

Fourteen commands were run (model `jev-1.12`); the full published table. Its `tool` column is printed from `call.tool.probability` (the page's print statement), not from a Choice `confidence`:

```text
  "show nvda 1h"
      plot_price(symbol='NVDA', resolution='1h')                        confidence 0.78   tool 1.00
  "plot rolling correlation between nvda and spy for the past month"
      rolling_correlation(symbol='NVDA', benchmark='SPY', window='1mo') confidence 0.91   tool 1.00
  "when during the day does nvda trade the most"
      intraday_pattern(symbol='NVDA')                                   confidence 0.53   tool 1.00
  "what moved today"
      top_movers(window='1d', direction='gainers')                      confidence 0.90   tool 0.90
  "what tickers do you have"
      list_symbols()                                                    confidence 1.00   tool 1.00
  "how did the market do this week"
      market_summary(window='1w')                                       confidence 0.96   tool 0.99
  "candles for tesla with a 20 period moving average"
      plot_price(symbol='TSLA', style='candles', moving_average='20')   confidence 0.69   tool 0.97
  "compare nvda amd and msft over the past three months"
      compare_returns(symbols=['NVDA', 'AMD', 'MSFT'], window='3mo')    confidence 0.94   tool 1.00
  "how volatile is tsla"
      volatility(symbol='TSLA')                                         confidence 0.96   tool 1.00
  "biggest losers today"
      top_movers(window='1d', direction='losers')                       confidence 0.98   tool 0.98
  "worst drawdown for nvda this quarter, and chart it please"
      drawdown(symbol='NVDA', window='3mo', plot=True)                  confidence 0.84   tool 0.84
  "spy stats for the last month"
      summary_stats(symbol='SPY', window='1mo')                         confidence 0.88   tool 0.88
  "show me apple daily with volume"
      plot_price(symbol='AAPL', resolution='1d', include_volume=True)   confidence 0.75   tool 0.85
  "is amd tracking nvidia lately"
      rolling_correlation(symbol='AMD', benchmark='NVDA')               confidence 0.82   tool 0.82
```

And the per-argument breakdown the page prints:

```text
"is amd tracking nvidia lately"  ->  rolling_correlation(symbol='AMD', benchmark='NVDA')   confidence 0.82
  symbol      'AMD'                     p 0.87   AMD 0.87  NVDA 0.13  AAPL 0.00
  benchmark   'NVDA'                    p 0.78   NVDA 0.92  AMD 0.08  AAPL 0.00
  window      omitted, default stands   p 0.96   
  resolution  omitted, default stands   p 0.99   
  weakest argument: benchmark
```

### Pitfalls

1. **The published per-argument and per-call numbers are not reproducible from what is published.** In the breakdown above, `benchmark` shows `p 0.78` while its top option is `NVDA 0.92` (the Choice confidence for 0.92 over six tickers would be 0.904), and the call's `confidence 0.82` is *higher* than the weakest argument's `p 0.78`, although the prose says the call confidence is the least certain judgement. The page prints these columns from `argument.probability` and `call.tool.probability`, not from the API's Choice `confidence` field, and for `symbol` the printed `p 0.87` equals the top probability `AMD 0.87` - so the published dispatcher appears to work on probabilities, while the reconstruction above uses Choice `confidence` (a different scale; see the confidence section). The aggregation in `dispatch.py` therefore includes something not shown; do not try to match its numbers, define your own.
2. **One request carries every function's questions.** That is cheap per the fan-out pattern, but it grows with the catalog; a Choice is capped at 255 options and a request at 64k tokens (state + all questions, https://docs.typesafe.ai/models).
3. **`stated` decides omission at 0.5 in this reconstruction.** For arguments with consequential defaults (a 3-month versus 1-day window on a trade), use an explicit middle band that asks the user.
4. **Never let the call run on low confidence for side-effecting functions.** The confidence page's own example gates a destructive action at a higher bar than a read-only one; the jaggedness page notes that state (here, the user's sentence) can steer the model adversarially.

---

## Recipe 6: Line-by-line search

Source: https://docs.typesafe.ai/cookbooks/semantic_find (difficulty "Beginner").

### Problem

Given GitHub's Terms of Service and a plain-language question, return the lines that answer it **and** detect when the document has no answer. `find()` returns P(exists) and one relevance score per line.

Three parts:

1. Tag each line with an id (`L052| ...`) so the model can point at it.
2. A Choice whose options are the line ids ranks lines. Choice probabilities sum to 1, so *some* line always ranks first.
3. In the same request, a Noul asks whether any line answers at all. A Noul is absolute, so it can fall near zero.

### Code

**condensed, executed** (setup; `cooksafe`, `Path`, `json_cache` lines removed):

```python
import os
import urllib.request

from typesafe_sdk import Choice, Noul, NoulCriteria, TypeSafeClient

TYPESAFE_MODEL = "jev-1.12"

client = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"), timeout=120.0
)
```

**condensed, executed** (`@json_cache` removed; fetched live from the pinned public gist):

```python
GIST = (
    "https://gist.githubusercontent.com/eugene-shvarts/900632789a24983d5678ffd508dd01f6"
    "/raw/cf9c2ab422d568deade949ef0a06bed6896964b9/github-tos.txt"
)


def fetch_document(url: str) -> str:
    request = urllib.request.Request(
        url, headers={"User-Agent": "typesafe-cookbook/1.0"}
    )
    with urllib.request.urlopen(request) as response:
        return response.read().decode()


LINES = fetch_document(GIST).splitlines()
```

**verbatim, executed:**

```python
def line_id(i: int) -> str:
    return f"L{i:03d}"


DOCUMENT = "\n".join(f"{line_id(i)}| {line}" for i, line in enumerate(LINES))
```

**verbatim, executed** - the ranking question. Descriptions are `None` because the state already holds each line's text; the query lives in `instructions`, so the state is identical across searches:

```python
def where_question(query: str) -> Choice:
    return Choice(
        instructions=f'Which line of the document contains the answer to: "{query}"?',
        criteria={line_id(i): None for i in range(len(LINES))},
    )
```

**verbatim, executed** - the existence question:

```python
def exists_question(query: str) -> Noul:
    return Noul(
        instructions=f'Does any line of the document address or answer: "{query}"?',
        criteria=NoulCriteria(
            true="At least one line of the document states or directly implies the answer",
            false="No line of the document addresses this",
        ),
    )
```

**condensed, executed** (`@json_cache` removed) - both questions, one request:

```python
def _find(
    model: str,
    state: str,
    where: Choice,
    exists: Noul,
) -> dict:
    response = client.system_one(
        state=state,
        questions={"where": where, "exists": exists},
        model=model,
    )
    probabilities = response.answers["where"].probabilities
    return {
        "exists": response.answers["exists"].noul,
        "relevance": [probabilities.get(line_id(i), 0.0) for i in range(len(LINES))],
    }


def find(query: str) -> dict:
    return _find(
        TYPESAFE_MODEL,
        DOCUMENT,
        where_question(query),
        exists_question(query),
    )
```

**verbatim, executed** - verdict bands and display:

```python
FOUND, ABSENT = 0.7, 0.35  # present answers typically read >=0.9, absent <=0.05


def verdict(exists: float) -> str:
    if exists >= FOUND:
        return "answered in this document"
    return "not in this document" if exists < ABSENT else "partially addressed"


def show(query: str, top: int = 4) -> dict:
    result = find(query)
    print(f'"{query}"')
    print(f"  exists {result['exists']:.2f} -> {verdict(result['exists'])}")
    ranked = sorted(
        range(len(LINES)), key=lambda i: result["relevance"][i], reverse=True
    )
    for i in ranked[:top]:
        bar = "#" * max(1, round(result["relevance"][i] * 12))
        preview = LINES[i][:58].rstrip()
        print(f"  {line_id(i)}  {result['relevance'][i]:.2f}  {bar:<12}  {preview}")
    return result
```

### Offline run

**original, executed** - canned answers from the published scores:

```python
from harness import fake_client, noul_answer

# Published per query: P(exists) and the top relevance scores. The leftover mass is
# spread evenly over the other line ids (synthetic, and below 0.001 per line).
PUBLISHED = {
    "who owns the code I upload?": (0.98, {"L052": 0.95, "L046": 0.02, "L051": 0.02, "L217": 0.01}),
    "can GitHub kick me off the platform without warning?": (0.97, {"L168": 0.97, "L167": 0.03}),
    "do I have to take disputes to arbitration?": (0.14, {"L205": 0.86, "L168": 0.02}),
    "can minors use GitHub with parental permission?": (0.46, {"L029": 0.90, "L012": 0.07}),
}


def sf_answers(state, questions):
    query = next(q for q in PUBLISHED if q in questions["exists"]["instructions"])
    exists, top = PUBLISHED[query]
    options = list(questions["where"]["criteria"])
    rest = (1 - sum(top.values())) / (len(options) - len(top))
    probabilities = {o: top.get(o, rest) for o in options}
    n, p_max = len(options), max(probabilities.values())
    return {
        "where": {"type": "choice", "choice": max(probabilities, key=probabilities.get),
                  "confidence": (p_max - 1 / n) / (1 - 1 / n), "probabilities": probabilities},
        "exists": noul_answer(exists),
    }


client, sent = fake_client(sf_answers)
```

**verbatim, executed** - output byte-identical to the published output (the document was fetched live and has the published 218 lines and 43,980 tagged characters):

```python
print(f"{len(LINES)} lines, {len(DOCUMENT):,} characters\n")
show("who owns the code I upload?")
print()
show("can GitHub kick me off the platform without warning?")
print()
show("do I have to take disputes to arbitration?", top=2)
print()
show("can minors use GitHub with parental permission?", top=2)
```

```text
218 lines, 43,980 characters

"who owns the code I upload?"
  exists 0.98 -> answered in this document
  L052  0.95  ###########   You own Your Content. If you post Content you did not crea
  L046  0.02  #             Short version: You own content you create, but you allow u
  L051  0.02  #             3. Ownership and License Grants
  L217  0.01  #             Questions about the Terms of Service? Contact us through t

"can GitHub kick me off the platform without warning?"
  exists 0.97 -> answered in this document
  L168  0.97  ############  GitHub has the right to suspend or terminate your access t
  L167  0.03  #             3. GitHub May Terminate
  L000  0.00  #             Effective date: April 27, 2026 · A. Definitions
  L001  0.00  #             Short version: We use these basic terms throughout the agr

"do I have to take disputes to arbitration?"
  exists 0.14 -> not in this document
  L205  0.86  ##########    Except to the extent applicable law provides otherwise, th
  L168  0.02  #             GitHub has the right to suspend or terminate your access t

"can minors use GitHub with parental permission?"
  exists 0.46 -> partially addressed
  L029  0.90  ###########   You must be age 13 or older. While we are thrilled to see
  L012  0.07  #             “User,” “You,” and “Your” refer to the individual person,
```

### Published results

On `jev-1.12`: two direct answers (`exists` 0.98 and 0.97, top line 0.95 and 0.97); arbitration - top line at 0.86 but `exists` 0.14, so *not in this document*; parental permission - the age rule ranks first at 0.90 but `exists` 0.46, *partially addressed*. *"The ranking tells you where to look; the `exists` score tells you whether the result answers the question."* The page's comment on the bands: present answers typically read >= 0.9, absent <= 0.05; tune before production. No cost is published.

### TypeScript port

**original, type-checked + executed** with `@typesafe-ai/sdk` 0.6.0 (`choice`, `noul` helpers; the `fetch` client option is documented in the SDK's `TypeSafeClientConfig` as "Custom HTTP fetch implementation for transport configuration or tests"). The answer types are inferred from the question map, so `answers.where.probabilities` and `answers.exists.noul` type-check without casts:

```typescript
import { choice, noul, TypeSafeClient, type Fetch } from "@typesafe-ai/sdk";

const lineId = (i: number): string => `L${String(i).padStart(3, "0")}`;

export async function find(client: TypeSafeClient, lines: string[], query: string) {
  if (lines.length > 255) throw new Error("a Choice takes at most 255 options: search in two passes");
  const ids = Object.fromEntries(lines.map((_, i) => [lineId(i), null]));
  const { answers } = await client.systemOne({
    state: lines.map((line, i) => `${lineId(i)}| ${line}`).join("\n"),
    model: "jev-1.13.0",
    questions: {
      where: choice(`Which line of the document contains the answer to: "${query}"?`, ids),
      exists: noul(`Does any line of the document address or answer: "${query}"?`, {
        true: "At least one line of the document states or directly implies the answer",
        false: "No line of the document addresses this",
      }),
    },
  });
  const relevance = lines.map((_, i) => answers.where.probabilities[lineId(i)] ?? 0);
  return { exists: answers.exists.noul, relevance };
}

// Offline check: a fetch that answers in the documented response shape.
const fakeFetch: Fetch = async (_url, init) => {
  const body = JSON.parse(String(init?.body));
  const options = Object.keys(body.questions.where.criteria);
  const probabilities = Object.fromEntries(options.map((o) => [o, o === "L001" ? 0.9 : 0.1 / (options.length - 1)]));
  return new Response(
    JSON.stringify({
      model: "jev-1.13.0",
      usage: { input_tokens: 50, output_tokens: 0 },
      answers: {
        where: { type: "choice", choice: "L001", confidence: 0.85, probabilities },
        exists: { type: "noul", noul: 0.97 },
      },
    }),
    { status: 200, headers: { "content-type": "application/json" } },
  );
};

const client = new TypeSafeClient({ apiKey: "offline", fetch: fakeFetch });
const result = await find(client, ["Effective date: today", "You own Your Content.", "Contact us."], "who owns my code?");
console.log(result.exists, result.relevance.map((r) => r.toFixed(2)).join(" "));
```

Output: `0.97 0.05 0.90 0.05`.

### Pitfalls

1. **255 options is a hard cap.** Past 255 lines, the page prescribes two passes: one Choice picks a window, a second ranks lines inside it. The Python SDK will not stop you client-side (Recipe 2, pitfall 7); the TS port above checks explicitly.
2. **The top line is meaningless without `exists`.** 0.86 on the arbitration line looked like an answer.
3. **Zero-probability ties sort by line order**, so `L000`, `L001` appear in the top 4 when only two lines carry mass. Hide rows below a floor.
4. **Option-order bias**: line ids are presented in document order; the jaggedness page reports a lean toward the first option. For a critical search, re-ask with the id list reversed.
5. **Large state**: accuracy falls as the state fills with irrelevant text (jaggedness item 5); the whole document is in every request. The 32k-token budget for state + longest question counts the `where` question, which lists every id, on top of the whole document.

---

## Cross-recipe pitfalls

| Pitfall | Where it bites | Fix |
|---|---|---|
| Model invents nothing - but code must still handle the escape (`none`, `out_of_range`, `says_nothing`) | Recipes 1, 2, 4 | branch on the escape before normalizing; give escapes descriptions |
| Confidence depends on the number of options | every Choice gate | convert to `p_max`, or gate per question |
| Option order bias toward the first option (jaggedness item 8) | Choices built from document order: 2, 6 | rotate or reverse and compare |
| One question per request | Recipe 2 as published | batch per state (Speculative fan-out) |
| Uncached demo code re-calls the API | Recipe 1 routing cell | compute once |
| `jev-1.12` pinned in all six | all | thresholds were tuned on 1.12; re-validate on `jev-1.13.0` before moving the pin |
| Arithmetic, counting, dates in the model | - | the jaggedness page: keep them in code; all six recipes already do |

---

## Published numbers, with sources

All model outputs below are from `jev-1.12` as published on the cited pages (fetched 2026-10-03); only the citation page dates its run (2026-08-16).

| Number | Value | Source |
|---|---|---|
| Price | $0.042 per million input tokens, output free | https://docs.typesafe.ai/models (2026-10-03) |
| Context | 64k tokens per request; 32k for state + longest question | https://docs.typesafe.ai/models (2026-10-03) |
| Rate limits | 100K tokens/s, 80 requests/s (stated as adjusting dynamically) | https://docs.typesafe.ai/models (2026-10-03) |
| Choice option cap | 255 | https://docs.typesafe.ai/primitives/choice; repeated on the pre-parsed and semantic-find pages |
| Date extraction | 6/6 correct; confidences 0.97, 0.91, 0.95, 0.46, 0.94, 0.92; gate 0.60 | date extraction page |
| Pre-parsed values | confidences 0.98, 1.00, 1.00, 0.90; P(credit) 0.01 / 0.99 | pre-parsed page |
| Structure recovery | 28 lines -> 17 blocks; 16 + 62 questions; 0.32 s + 0.51 s; 10,211 tokens; $0.0003 (cell) vs $0.0015 (prose) | autoformat page |
| Citation check | 58,365 chars, 45 sections, 8 citations; 4 verified >= 0.93; gate 0.8 | citation page (run 2026-08-16) |
| Function calling | 10 functions, 156,780 one-minute bars, 28 fillable arguments, 54 questions per command, 14 commands | function calling page |
| Line-by-line search | 218 lines, 43,980 characters; bands 0.7 / 0.35 | semantic find page |

---

## What could not be verified

- **No call was made to the real API** (no key). Every "executed" block ran against canned answers in the documented response shape; the published model outputs are quoted, not reproduced. Whether `jev-1.13.0` gives the same answers as `jev-1.12` on these inputs is unknown.
- **Whether the API still accepts `jev-1.12`** is unverified. The models page lists only `jev-1.13.0` and its aliases and says versioned ids are accepted "whether or not they appear in the list"; it does not say how long old versions are served.
- **Function calling**: `dispatch.py`, `trader.py` and `spec.json` are not published (probed `https://docs.typesafe.ai/cookbooks/function_calling/dispatch.py` and siblings on 2026-10-03: 404; no public cookbook repository under the `typesafe-ai` GitHub organization). The reconstruction above is this document's, and the aggregation behind the published `p` and `confidence` columns could not be determined.
- **Citation check**: `citations.json` is not published; six of the eight citations used offline are reconstructions (see the recipe).
- **Structure recovery**: the join probabilities for lines 23, 24 and 26 and most companion answers are not published; the offline values are synthetic and chosen to match the published blocks.
- **Costs** are published only for structure recovery, and that page contradicts itself ($0.0003 vs $0.0015); the arithmetic above favours $0.0003.
