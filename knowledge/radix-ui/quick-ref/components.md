# Radix UI Components Quick Reference

## Dialog

```tsx
<Dialog.Root>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay />
    <Dialog.Content>
      <Dialog.Title>Title</Dialog.Title>
      <Dialog.Description>Description</Dialog.Description>
      <Dialog.Close>Close</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

## Dropdown Menu

```tsx
<DropdownMenu.Root>
  <DropdownMenu.Trigger>Menu</DropdownMenu.Trigger>
  <DropdownMenu.Content>
    <DropdownMenu.Item>Edit</DropdownMenu.Item>
    <DropdownMenu.Item>Delete</DropdownMenu.Item>
  </DropdownMenu.Content>
</DropdownMenu.Root>
```

## Select

```tsx
<Select.Root value={value} onValueChange={setValue}>
  <Select.Trigger>
    <Select.Value />
  </Select.Trigger>
  <Select.Content>
    <Select.Item value="option1">Option 1</Select.Item>
  </Select.Content>
</Select.Root>
```

## Popover

```tsx
<Popover.Root>
  <Popover.Trigger>Info</Popover.Trigger>
  <Popover.Content>Content</Popover.Content>
</Popover.Root>
```

## Tooltip

```tsx
<Tooltip.Provider>
  <Tooltip.Root>
    <Tooltip.Trigger>Hover me</Tooltip.Trigger>
    <Tooltip.Content>Tooltip text</Tooltip.Content>
  </Tooltip.Root>
</Tooltip.Provider>
```

## Checkbox

```tsx
<Checkbox.Root checked={checked} onCheckedChange={setChecked}>
  <Checkbox.Indicator>
    <CheckIcon />
  </Checkbox.Indicator>
</Checkbox.Root>
```
