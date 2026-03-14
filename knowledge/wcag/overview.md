# WCAG 2.2 Overview

> Source: https://www.w3.org/TR/WCAG22/

## What is WCAG?

Web Content Accessibility Guidelines (WCAG) are international standards for making web content accessible to people with disabilities.

## Conformance Levels

| Level | Description | Requirement |
|-------|-------------|-------------|
| **A** | Minimum accessibility | Essential for basic access |
| **AA** | Standard accessibility | Legal requirement (ADA, EN 301 549) |
| **AAA** | Enhanced accessibility | Specialized contexts |

### Level Requirements

- **Level A**: Web page satisfies all Level A success criteria
- **Level AA**: Web page satisfies all Level A and AA success criteria
- **Level AAA**: Web page satisfies all Level A, AA, and AAA success criteria

## POUR Principles

### 1. Perceivable

Information and UI must be presentable in ways users can perceive.

- **1.1 Text Alternatives** - Alt text for non-text content
- **1.2 Time-based Media** - Captions, audio descriptions
- **1.3 Adaptable** - Content structure conveyed programmatically
- **1.4 Distinguishable** - Color contrast, resize text, spacing

### 2. Operable

UI components and navigation must be operable.

- **2.1 Keyboard Accessible** - All functionality via keyboard
- **2.2 Enough Time** - Adjustable time limits
- **2.3 Seizures and Physical Reactions** - No flashing content
- **2.4 Navigable** - Skip links, focus order, focus visible
- **2.5 Input Modalities** - Beyond keyboard (touch, motion)

### 3. Understandable

Information and UI operation must be understandable.

- **3.1 Readable** - Language of page, unusual words
- **3.2 Predictable** - Consistent navigation, identification
- **3.3 Input Assistance** - Error identification, labels

### 4. Robust

Content must be robust enough for diverse user agents.

- **4.1 Compatible** - Valid markup, name/role/value

## New in WCAG 2.2

### Level AA (6 new criteria)

| Criterion | Description |
|-----------|-------------|
| **2.4.11 Focus Not Obscured (Minimum)** | Focused element at least partially visible |
| **2.4.13 Focus Appearance** | Focus indicator meets size/contrast requirements |
| **2.5.7 Dragging Movements** | Single pointer alternative to drag |
| **2.5.8 Target Size (Minimum)** | 24×24 CSS pixels minimum |
| **3.2.6 Consistent Help** | Help in consistent location |
| **3.3.7 Redundant Entry** | Don't require re-entering info |

### Level AAA (3 new criteria)

| Criterion | Description |
|-----------|-------------|
| **2.4.12 Focus Not Obscured (Enhanced)** | Focused element fully visible |
| **3.3.8 Accessible Authentication (Minimum)** | No cognitive function test |
| **3.3.9 Accessible Authentication (Enhanced)** | No object/content recognition |

## Removed in WCAG 2.2

**4.1.1 Parsing** - Obsolete (browsers handle parsing issues)

## Backwards Compatibility

WCAG 2.0, 2.1, and 2.2 are backwards compatible:
- Content conforming to WCAG 2.2 also conforms to 2.1 and 2.0
- All success criteria from 2.0 are in 2.1
- All success criteria from 2.1 are in 2.2 (except 4.1.1)

## Legal Requirements

| Region | Standard |
|--------|----------|
| USA (Section 508) | WCAG 2.0 AA |
| USA (ADA Title II) | WCAG 2.1 AA |
| Europe (EN 301 549) | WCAG 2.1 AA |
| Canada (AODA) | WCAG 2.1 AA |
| Australia | WCAG 2.1 AA |

## Key Success Criteria

### Most Common Failures

1. **1.1.1 Non-text Content** - Missing alt text
2. **1.4.3 Contrast (Minimum)** - Insufficient color contrast
3. **2.4.4 Link Purpose** - Non-descriptive links ("click here")
4. **3.3.2 Labels or Instructions** - Form inputs without labels
5. **4.1.2 Name, Role, Value** - Custom controls without ARIA

### Quick Wins

- Add `alt` to all images
- Ensure 4.5:1 contrast ratio
- Add `<label>` to all form inputs
- Use semantic HTML (`<nav>`, `<main>`, `<button>`)
- Add skip links
- Test keyboard navigation
