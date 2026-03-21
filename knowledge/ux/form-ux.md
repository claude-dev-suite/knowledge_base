# Form UX

Forms are the primary way users give data to applications. Poor form UX directly costs revenue — Expedia removed one optional "Company Name" field and gained **$12 million in annual revenue**. Every unnecessary field, confusing label, or late error message is a conversion killer.

---

## Core Research Findings

| Finding | Data | Source |
|---------|------|--------|
| Single-column vs multi-column | Single-column completes **15.4% faster** | Various A/B studies |
| Optional field removal | Expedia: +$12M/year from removing 1 field | Luke Wroblewski, "Web Form Design" |
| Mobile form abandonment | **61% of users abandon** non-optimized mobile forms | Various studies |
| Inline validation | Reduces errors and completion time | Holst (2011), CXL studies |
| Label position (top vs side) | Top labels complete **faster** in eye-tracking studies | NNGroup |

---

## Layout Rules

### Single-column layout (always preferred)

```
✓ Correct — single column:        ✗ Avoid — multi-column:
┌────────────────────────────┐    ┌──────────────┬──────────────┐
│ First Name                 │    │ First Name   │ Last Name    │
│ [                        ] │    │ [          ] │ [          ] │
│                            │    │              │              │
│ Last Name                  │    │ Email        │ Phone        │
│ [                        ] │    │ [          ] │ [          ] │
│                            │    └──────────────┴──────────────┘
│ Email                      │    (Users skip columns, lose their place,
│ [                        ] │    and complete more slowly)
└────────────────────────────┘
```

**Exception**: Two logically paired fields (e.g., city + postal code) can sit side-by-side on desktop only when they form a single conceptual unit.

### Label position

Always **above** the field. Never use placeholder text as the only label.

```
✓ Label above (always use):    ✗ Placeholder only (never use):
  Email address                   [Email address          ]
  [mario@example.com    ]         ↑ Label disappears on focus
```

**Why placeholder-as-label fails**:
1. Disappears on focus — user can't verify what field they're filling
2. Placeholder styling often has insufficient contrast (fails WCAG)
3. Screen readers may not announce placeholder as label
4. No room to show helper text

---

## Validation Timing

| Rule | Timing | Why |
|------|--------|-----|
| Show errors | **On blur** (when user leaves field) | Not while typing — too aggressive |
| Clear errors | As user types (once field has been blurred) | Real-time recovery feels responsive |
| Show success | On blur (for critical fields) | Optional, but reassuring |
| Form-level errors | On submit | For server-side validation |

**Research**: Validating on blur (versus on submit or on every keystroke) reduces errors, increases completion rates, and is rated as less frustrating by users.

### Validation implementation

```tsx
function useFieldValidation(value: string, validator: (v: string) => string | null) {
  const [dirty, setDirty] = useState(false);
  const error = dirty ? validator(value) : null;

  return {
    error,
    onBlur: () => setDirty(true),
    // Clear error as user types once field has been touched
    onChange: (newValue: string) => {
      if (dirty && !validator(newValue)) {
        setDirty(false);  // Will re-validate on next blur
      }
    },
  };
}
```

---

## Error Message Design

**Rule**: Errors must be adjacent to the problem and in plain language explaining HOW to fix it.

| Property | Correct | Wrong |
|----------|---------|-------|
| Placement | Immediately below the field | Top of form only |
| Language | "Enter a valid email (e.g., mario@example.com)" | "Invalid input" |
| Tone | Helpful, neutral | Accusatory ("You entered a wrong value") |
| Color | `--color-error` + icon | Color only |
| ARIA | `role="alert"`, `aria-describedby` | None |

```tsx
// Complete accessible form field with error state
function TextField({
  id,
  label,
  type = 'text',
  value,
  error,
  hint,
  required,
  onChange,
  onBlur,
}: TextFieldProps) {
  const errorId = `${id}-error`;
  const hintId  = `${id}-hint`;
  const describedBy = [
    hint  ? hintId  : null,
    error ? errorId : null,
  ].filter(Boolean).join(' ') || undefined;

  return (
    <div className="space-y-1.5">
      <label htmlFor={id} className="block text-sm font-medium text-foreground">
        {label}
        {!required && (
          <span className="ml-1 text-muted-foreground font-normal">(optional)</span>
        )}
      </label>

      {hint && (
        <p id={hintId} className="text-sm text-muted-foreground">
          {hint}
        </p>
      )}

      <input
        id={id}
        type={type}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        onBlur={onBlur}
        required={required}
        aria-required={required}
        aria-invalid={error ? 'true' : undefined}
        aria-describedby={describedBy}
        className={cn(
          'flex h-11 w-full rounded-md border bg-background px-3 py-2',
          'text-sm ring-offset-background',
          'placeholder:text-muted-foreground',
          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring',
          'disabled:cursor-not-allowed disabled:opacity-50',
          error
            ? 'border-destructive focus-visible:ring-destructive'
            : 'border-input'
        )}
      />

      {error && (
        <p
          id={errorId}
          role="alert"
          className="flex items-center gap-1.5 text-sm text-destructive"
        >
          <AlertCircleIcon className="h-4 w-4 flex-shrink-0" aria-hidden="true" />
          {error}
        </p>
      )}
    </div>
  );
}
```

---

## Required vs Optional Fields

**Mark optional fields, not required ones.** This is counterintuitive but correct when most fields are required:

```
If 8 of 10 fields are required:
  ✓ Mark 2 optional fields with "(optional)"
  ✗ Mark 8 required fields with "*" — visual noise, user anxiety
```

Keep fields to the **minimum necessary**. Before adding a field, ask:
1. Do we actually use this data?
2. Can we get it later (after signup)?
3. Can we infer it from other data?

---

## Multi-Step Forms

For long forms (> 7 fields), break into logical steps:

```tsx
// Progress indicator
function FormProgress({ currentStep, totalSteps, stepLabels }: FormProgressProps) {
  return (
    <nav aria-label="Form progress">
      <ol className="flex items-center gap-2">
        {stepLabels.map((label, i) => (
          <li key={i} className="flex items-center gap-2">
            <span
              className={cn(
                'flex h-8 w-8 items-center justify-center rounded-full text-sm font-medium',
                i < currentStep && 'bg-primary text-primary-foreground',
                i === currentStep && 'border-2 border-primary text-primary',
                i > currentStep && 'border-2 border-muted text-muted-foreground'
              )}
              aria-current={i === currentStep ? 'step' : undefined}
            >
              {i < currentStep ? <CheckIcon className="h-4 w-4" /> : i + 1}
            </span>
            <span className="text-sm hidden sm:block">{label}</span>
            {i < totalSteps - 1 && (
              <ChevronRightIcon className="h-4 w-4 text-muted-foreground" />
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}
```

**Multi-step rules**:
- Show progress clearly (step X of Y, or visual progress bar)
- Allow going back without losing data
- Save progress (localStorage or server) to prevent loss on navigation
- Put the hardest/most personal data last (name/email first, payment last)
- Summarize what was entered before final submission

---

## Mobile Form Optimization

Mobile users account for >60% of web traffic. Forms must be mobile-first.

```html
<!-- Use the correct input type — triggers the right keyboard -->
<input type="email"   />  <!-- Email keyboard: @ and . prominent -->
<input type="tel"     />  <!-- Numeric keyboard on mobile -->
<input type="number"  />  <!-- Numeric input -->
<input type="search"  />  <!-- Search keyboard with search action key -->
<input type="url"     />  <!-- URL keyboard with .com key -->
<input type="password"/>  <!-- Masked input + password managers -->

<!-- Prevent iOS auto-zoom (font-size must be >= 16px) -->
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=5">
```

```css
/* iOS auto-zoom prevention — all inputs must be at least 16px */
input, select, textarea {
  font-size: max(16px, 1rem);
}
```

```html
<!-- Autocomplete attributes improve fill rate by 30%+ -->
<input autocomplete="given-name"   />
<input autocomplete="family-name"  />
<input autocomplete="email"        />
<input autocomplete="tel"          />
<input autocomplete="postal-code"  />
<input autocomplete="cc-number"    />
<input autocomplete="cc-exp"       />
<input autocomplete="cc-csc"       />
<input autocomplete="new-password" />
```

---

## References

- Luke Wroblewski — *Web Form Design: Filling in the Blanks* (2008, still foundational)
- CXL — [Form Design Best Practices: 13 Empirically Backed Best Practices](https://cxl.com/blog/form-design-best-practices/)
- NNGroup — [Website Forms Usability: Top 10 Recommendations](https://www.nngroup.com/articles/web-form-design/)
- Holst, C. (2011) — "An Eye-Tracking Study on Web Form Validation"
- Buildform — [8 Form Design Best Practices for 2025](https://buildform.ai/blog/form-design-best-practices/)
