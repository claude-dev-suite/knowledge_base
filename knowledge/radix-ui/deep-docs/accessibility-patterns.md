# Radix UI Accessibility & Advanced Patterns

## Keyboard Navigation

### Built-in Keyboard Support

Radix UI components include comprehensive keyboard navigation:

| Component | Keys | Action |
|-----------|------|--------|
| Dialog | `Escape` | Close dialog |
| Dialog | `Tab` | Cycle through focusable elements |
| Dropdown Menu | `Space`/`Enter` | Open/select item |
| Dropdown Menu | `Arrow Keys` | Navigate items |
| Dropdown Menu | `Escape` | Close menu |
| Select | `Space`/`Enter` | Open/select |
| Select | `Arrow Keys` | Navigate options |
| Tooltip | `Escape` | Dismiss |

### Custom Keyboard Handlers

```tsx
<Dialog.Content onEscapeKeyDown={(e) => {
  // Prevent closing if form is dirty
  if (isDirty) {
    e.preventDefault();
    setShowConfirmation(true);
  }
}}>
  {/* Content */}
</Dialog.Content>
```

## Focus Management

### Auto Focus on Mount

```tsx
<Dialog.Content>
  <input autoFocus />
  {/* or */}
  <Dialog.Close ref={initialFocusRef} />
</Dialog.Content>
```

### Focus Trapping

```tsx
import { FocusScope } from '@radix-ui/react-focus-scope';

<FocusScope trapped>
  <div>
    <input />
    <button>Submit</button>
  </div>
</FocusScope>
```

### Return Focus

```tsx
// Focus returns to trigger automatically
<Dialog.Root>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Content>
    {/* When closed, focus returns to trigger */}
  </Dialog.Content>
</Dialog.Root>
```

## Screen Reader Support

### ARIA Labels

```tsx
<Dialog.Root>
  <Dialog.Trigger aria-label="Open user settings">
    <SettingsIcon />
  </Dialog.Trigger>
  <Dialog.Content aria-describedby="dialog-desc">
    <Dialog.Title id="dialog-title">Settings</Dialog.Title>
    <Dialog.Description id="dialog-desc">
      Manage your account settings and preferences
    </Dialog.Description>
  </Dialog.Content>
</Dialog.Root>
```

### Visually Hidden Elements

```tsx
import { VisuallyHidden } from '@radix-ui/react-visually-hidden';

<button>
  <DeleteIcon />
  <VisuallyHidden>Delete item</VisuallyHidden>
</button>
```

### Accessible Names

```tsx
// Dropdown with accessible name
<DropdownMenu.Root>
  <DropdownMenu.Trigger aria-label="User menu">
    <Avatar />
  </DropdownMenu.Trigger>
  <DropdownMenu.Content>
    <DropdownMenu.Label>Account</DropdownMenu.Label>
    <DropdownMenu.Item>Profile</DropdownMenu.Item>
  </DropdownMenu.Content>
</DropdownMenu.Root>
```

## Advanced Dialog Patterns

### Nested Dialogs

```tsx
function ParentDialog() {
  return (
    <Dialog.Root>
      <Dialog.Trigger>Open Parent</Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay />
        <Dialog.Content>
          <Dialog.Title>Parent Dialog</Dialog.Title>
          <ChildDialog />
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}

function ChildDialog() {
  return (
    <Dialog.Root>
      <Dialog.Trigger>Open Child</Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay />
        <Dialog.Content style={{ zIndex: 1001 }}>
          <Dialog.Title>Child Dialog</Dialog.Title>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

### Controlled Dialog State

```tsx
function FormDialog() {
  const [open, setOpen] = useState(false);
  const [data, setData] = useState({});

  const handleClose = () => {
    if (isDirty && !confirm('Discard changes?')) {
      return;
    }
    setOpen(false);
  };

  return (
    <Dialog.Root open={open} onOpenChange={setOpen}>
      <Dialog.Trigger>Edit</Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Content>
          <form onSubmit={(e) => {
            e.preventDefault();
            submitData(data);
            setOpen(false);
          }}>
            {/* Form fields */}
            <Dialog.Close onClick={handleClose}>Cancel</Dialog.Close>
          </form>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

### Modal vs Non-Modal

```tsx
// Modal (default) - blocks interaction with rest of page
<Dialog.Root modal={true}>
  <Dialog.Content>Modal content</Dialog.Content>
</Dialog.Root>

// Non-modal - allows interaction with page
<Dialog.Root modal={false}>
  <Dialog.Content>Non-modal content</Dialog.Content>
</Dialog.Root>
```

## Dropdown Menu Advanced

### Submenu

```tsx
<DropdownMenu.Root>
  <DropdownMenu.Trigger>Menu</DropdownMenu.Trigger>
  <DropdownMenu.Content>
    <DropdownMenu.Item>Edit</DropdownMenu.Item>

    <DropdownMenu.Sub>
      <DropdownMenu.SubTrigger>
        More <ChevronRightIcon />
      </DropdownMenu.SubTrigger>
      <DropdownMenu.Portal>
        <DropdownMenu.SubContent>
          <DropdownMenu.Item>Archive</DropdownMenu.Item>
          <DropdownMenu.Item>Duplicate</DropdownMenu.Item>
        </DropdownMenu.SubContent>
      </DropdownMenu.Portal>
    </DropdownMenu.Sub>

    <DropdownMenu.Separator />
    <DropdownMenu.Item>Delete</DropdownMenu.Item>
  </DropdownMenu.Content>
</DropdownMenu.Root>
```

### Checkbox Items

```tsx
function MenuWithSelection() {
  const [checked, setChecked] = useState({
    bold: false,
    italic: false,
  });

  return (
    <DropdownMenu.Root>
      <DropdownMenu.Content>
        <DropdownMenu.CheckboxItem
          checked={checked.bold}
          onCheckedChange={(val) => setChecked(prev => ({ ...prev, bold: val }))}
        >
          <DropdownMenu.ItemIndicator>
            <CheckIcon />
          </DropdownMenu.ItemIndicator>
          Bold
        </DropdownMenu.CheckboxItem>

        <DropdownMenu.CheckboxItem
          checked={checked.italic}
          onCheckedChange={(val) => setChecked(prev => ({ ...prev, italic: val }))}
        >
          <DropdownMenu.ItemIndicator>
            <CheckIcon />
          </DropdownMenu.ItemIndicator>
          Italic
        </DropdownMenu.CheckboxItem>
      </DropdownMenu.Content>
    </DropdownMenu.Root>
  );
}
```

### Radio Group Items

```tsx
function AlignmentMenu() {
  const [alignment, setAlignment] = useState('left');

  return (
    <DropdownMenu.Root>
      <DropdownMenu.Content>
        <DropdownMenu.RadioGroup value={alignment} onValueChange={setAlignment}>
          <DropdownMenu.RadioItem value="left">
            <DropdownMenu.ItemIndicator>
              <CheckIcon />
            </DropdownMenu.ItemIndicator>
            Left
          </DropdownMenu.RadioItem>
          <DropdownMenu.RadioItem value="center">
            <DropdownMenu.ItemIndicator>
              <CheckIcon />
            </DropdownMenu.ItemIndicator>
            Center
          </DropdownMenu.RadioItem>
          <DropdownMenu.RadioItem value="right">
            <DropdownMenu.ItemIndicator>
              <CheckIcon />
            </DropdownMenu.ItemIndicator>
            Right
          </DropdownMenu.RadioItem>
        </DropdownMenu.RadioGroup>
      </DropdownMenu.Content>
    </DropdownMenu.Root>
  );
}
```

## Select Advanced Patterns

### Grouped Options

```tsx
<Select.Root>
  <Select.Trigger>
    <Select.Value placeholder="Select country..." />
  </Select.Trigger>
  <Select.Content>
    <Select.Group>
      <Select.Label>Europe</Select.Label>
      <Select.Item value="uk">United Kingdom</Select.Item>
      <Select.Item value="de">Germany</Select.Item>
      <Select.Item value="fr">France</Select.Item>
    </Select.Group>

    <Select.Separator />

    <Select.Group>
      <Select.Label>Americas</Select.Label>
      <Select.Item value="us">United States</Select.Item>
      <Select.Item value="ca">Canada</Select.Item>
      <Select.Item value="mx">Mexico</Select.Item>
    </Select.Group>
  </Select.Content>
</Select.Root>
```

### Custom Option Rendering

```tsx
<Select.Root>
  <Select.Content>
    <Select.Viewport>
      {users.map(user => (
        <Select.Item key={user.id} value={user.id}>
          <Select.ItemText>
            <div className="flex items-center gap-2">
              <Avatar src={user.avatar} />
              <div>
                <div>{user.name}</div>
                <div className="text-sm text-gray-500">{user.email}</div>
              </div>
            </div>
          </Select.ItemText>
          <Select.ItemIndicator>
            <CheckIcon />
          </Select.ItemIndicator>
        </Select.Item>
      ))}
    </Select.Viewport>
  </Select.Content>
</Select.Root>
```

## Popover Positioning

### Side and Alignment

```tsx
<Popover.Root>
  <Popover.Trigger>Open</Popover.Trigger>
  <Popover.Content
    side="top"        // top | right | bottom | left
    align="start"     // start | center | end
    sideOffset={5}    // Distance from trigger
    alignOffset={0}   // Offset along align axis
  >
    Content
  </Popover.Content>
</Popover.Root>
```

### Collision Handling

```tsx
<Popover.Content
  collisionPadding={10}  // Padding from viewport edge
  avoidCollisions={true}  // Flip to opposite side if needed
  sticky="partial"        // Stick to edge: "partial" | "always"
>
  Content
</Popover.Content>
```

## Animation Patterns

### CSS Transitions

```tsx
// Dialog with animation
<Dialog.Overlay className="dialog-overlay" />
<Dialog.Content className="dialog-content" />

// CSS
.dialog-overlay {
  background: rgba(0, 0, 0, 0.5);
  transition: opacity 200ms ease-out;
}

.dialog-overlay[data-state="open"] {
  animation: fadeIn 200ms ease-out;
}

.dialog-overlay[data-state="closed"] {
  animation: fadeOut 200ms ease-in;
}

.dialog-content[data-state="open"] {
  animation: slideIn 300ms cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translate(-50%, -48%) scale(0.96);
  }
  to {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}
```

### Framer Motion Integration

```tsx
import { motion, AnimatePresence } from 'framer-motion';

<Dialog.Root open={open} onOpenChange={setOpen}>
  <Dialog.Portal forceMount>
    <AnimatePresence>
      {open && (
        <>
          <Dialog.Overlay asChild>
            <motion.div
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
            />
          </Dialog.Overlay>

          <Dialog.Content asChild>
            <motion.div
              initial={{ opacity: 0, scale: 0.95, y: 10 }}
              animate={{ opacity: 1, scale: 1, y: 0 }}
              exit={{ opacity: 0, scale: 0.95, y: 10 }}
              transition={{ duration: 0.2 }}
            >
              {/* Content */}
            </motion.div>
          </Dialog.Content>
        </>
      )}
    </AnimatePresence>
  </Dialog.Portal>
</Dialog.Root>
```

## Form Integration

### Checkbox in Forms

```tsx
import { useForm } from 'react-hook-form';

function CheckboxForm() {
  const { register, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Checkbox.Root
        {...register('terms')}
        onCheckedChange={(checked) => {
          // Sync with react-hook-form
        }}
      >
        <Checkbox.Indicator>
          <CheckIcon />
        </Checkbox.Indicator>
      </Checkbox.Root>
      <label>Accept terms</label>
    </form>
  );
}
```

### Select in Forms

```tsx
<Controller
  name="country"
  control={control}
  render={({ field }) => (
    <Select.Root value={field.value} onValueChange={field.onChange}>
      <Select.Trigger>
        <Select.Value />
      </Select.Trigger>
      <Select.Content>
        <Select.Item value="us">United States</Select.Item>
        <Select.Item value="uk">United Kingdom</Select.Item>
      </Select.Content>
    </Select.Root>
  )}
/>
```

## Styling with Tailwind

### Complete Styled Dialog

```tsx
<Dialog.Root>
  <Dialog.Trigger className="btn-primary">
    Open
  </Dialog.Trigger>

  <Dialog.Portal>
    <Dialog.Overlay className="fixed inset-0 bg-black/50 data-[state=open]:animate-fade-in" />

    <Dialog.Content className="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-white rounded-lg shadow-lg p-6 w-[450px] max-w-[90vw] max-h-[85vh] overflow-y-auto data-[state=open]:animate-content-show">
      <Dialog.Title className="text-lg font-semibold mb-2">
        Dialog Title
      </Dialog.Title>

      <Dialog.Description className="text-sm text-gray-600 mb-4">
        Dialog description
      </Dialog.Description>

      <div className="space-y-4">
        {/* Content */}
      </div>

      <div className="flex justify-end gap-2 mt-6">
        <Dialog.Close className="btn-secondary">
          Cancel
        </Dialog.Close>
        <button className="btn-primary">
          Confirm
        </button>
      </div>

      <Dialog.Close className="absolute top-4 right-4 text-gray-400 hover:text-gray-600">
        <XIcon />
      </Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>

// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      keyframes: {
        'fade-in': {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        'content-show': {
          '0%': { opacity: '0', transform: 'translate(-50%, -48%) scale(0.96)' },
          '100%': { opacity: '1', transform: 'translate(-50%, -50%) scale(1)' },
        },
      },
      animation: {
        'fade-in': 'fade-in 200ms ease-out',
        'content-show': 'content-show 150ms cubic-bezier(0.16, 1, 0.3, 1)',
      },
    },
  },
};
```

## Best Practices

1. ✅ Use `asChild` to avoid wrapper divs
2. ✅ Always provide accessible labels
3. ✅ Use Portal for proper z-index stacking
4. ✅ Handle controlled state for complex interactions
5. ✅ Test keyboard navigation
6. ✅ Test with screen readers
7. ✅ Provide visual focus indicators
8. ✅ Use data attributes for state-specific styling
9. ✅ Implement proper loading states
10. ✅ Handle edge cases (empty states, errors)

## References

- [Radix UI Documentation](https://www.radix-ui.com/primitives)
- [WAI-ARIA Practices](https://www.w3.org/WAI/ARIA/apg/)
- [Accessibility Guidelines](https://www.a11yproject.com/)
