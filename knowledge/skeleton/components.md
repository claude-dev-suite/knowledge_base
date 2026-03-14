# Skeleton UI Components

> Official Documentation: https://www.skeleton.dev/components

## Overview

Reference for all Skeleton UI components with usage examples and props.

---

## Feedback Components

### Toast

```svelte
<script lang="ts">
  import { Toast, getToastStore, type ToastSettings } from '@skeletonlabs/skeleton';

  const toastStore = getToastStore();

  function showToast() {
    const toast: ToastSettings = {
      message: 'Operation completed!',
      timeout: 3000,
      background: 'variant-filled-success',
      classes: 'border-l-4 border-success-500',
    };
    toastStore.trigger(toast);
  }

  function showWarning() {
    toastStore.trigger({
      message: 'Please review your input',
      background: 'variant-filled-warning',
      autohide: false,
      action: {
        label: 'Dismiss',
        response: () => toastStore.clear()
      }
    });
  }
</script>

<!-- Place Toast component in layout -->
<Toast position="br" />

<button class="btn variant-filled" onclick={showToast}>Show Toast</button>
```

### Modal

```svelte
<script lang="ts">
  import { Modal, getModalStore, type ModalSettings, type ModalComponent } from '@skeletonlabs/skeleton';
  import MyCustomModal from './MyCustomModal.svelte';

  const modalStore = getModalStore();

  // Simple confirm modal
  function showConfirm() {
    const modal: ModalSettings = {
      type: 'confirm',
      title: 'Confirm Action',
      body: 'Are you sure you want to proceed?',
      response: (confirmed: boolean) => {
        if (confirmed) console.log('Confirmed!');
      }
    };
    modalStore.trigger(modal);
  }

  // Prompt modal
  function showPrompt() {
    const modal: ModalSettings = {
      type: 'prompt',
      title: 'Enter Your Name',
      body: 'This will be used for personalization.',
      value: '',
      valueAttr: { type: 'text', required: true },
      response: (value: string) => {
        if (value) console.log('Name:', value);
      }
    };
    modalStore.trigger(modal);
  }

  // Custom component modal
  const modalComponentRegistry: Record<string, ModalComponent> = {
    myModal: { ref: MyCustomModal }
  };

  function showCustom() {
    const modal: ModalSettings = {
      type: 'component',
      component: 'myModal',
      meta: { customData: 'value' }
    };
    modalStore.trigger(modal);
  }
</script>

<!-- Place Modal in layout -->
<Modal components={modalComponentRegistry} />

<!-- MyCustomModal.svelte -->
<script lang="ts">
  import { getModalStore } from '@skeletonlabs/skeleton';

  export let parent: any;

  const modalStore = getModalStore();

  function onClose() {
    if ($modalStore[0].response) $modalStore[0].response(true);
    modalStore.close();
  }
</script>

<div class="card p-4 w-modal shadow-xl space-y-4">
  <header class="text-2xl font-bold">Custom Modal</header>
  <article>Your custom content here</article>
  <footer class="modal-footer flex justify-end gap-2">
    <button class="btn variant-ghost-surface" onclick={() => parent.onClose()}>Cancel</button>
    <button class="btn variant-filled" onclick={onClose}>Confirm</button>
  </footer>
</div>
```

### Drawer

```svelte
<script lang="ts">
  import { Drawer, getDrawerStore, type DrawerSettings } from '@skeletonlabs/skeleton';

  const drawerStore = getDrawerStore();

  function openDrawer() {
    const settings: DrawerSettings = {
      id: 'navigation',
      position: 'left',
      width: 'w-[280px] md:w-[480px]',
      padding: 'p-4',
      rounded: 'rounded-xl',
      meta: { someData: 'value' }
    };
    drawerStore.open(settings);
  }
</script>

<!-- Place in layout -->
<Drawer>
  {#if $drawerStore.id === 'navigation'}
    <nav class="list-nav">
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/about">About</a></li>
        <li><a href="/contact">Contact</a></li>
      </ul>
    </nav>
  {/if}
</Drawer>

<button class="btn variant-filled" onclick={openDrawer}>Open Drawer</button>
```

---

## Data Display

### Avatar

```svelte
<script>
  import { Avatar } from '@skeletonlabs/skeleton';
</script>

<!-- Image avatar -->
<Avatar src="https://example.com/avatar.jpg" width="w-16" rounded="rounded-full" />

<!-- Initials avatar -->
<Avatar initials="JD" background="bg-primary-500" width="w-12" />

<!-- With action -->
<Avatar
  src="https://example.com/avatar.jpg"
  action={() => console.log('Clicked')}
  actionParams="some-param"
  cursor="cursor-pointer"
/>

<!-- Sizes -->
<Avatar src="" width="w-8" />
<Avatar src="" width="w-12" />
<Avatar src="" width="w-16" />
<Avatar src="" width="w-24" />
```

### ProgressBar & ProgressRadial

```svelte
<script>
  import { ProgressBar, ProgressRadial } from '@skeletonlabs/skeleton';

  let progress = $state(50);
</script>

<!-- Linear progress -->
<ProgressBar value={progress} max={100} />

<!-- Determinate -->
<ProgressBar label="Progress" value={75} max={100} height="h-4" meter="bg-primary-500" track="bg-primary-500/30" />

<!-- Indeterminate (loading) -->
<ProgressBar />

<!-- Radial progress -->
<ProgressRadial value={progress} stroke={100} meter="stroke-primary-500" track="stroke-primary-500/30">
  {progress}%
</ProgressRadial>

<!-- Radial sizes -->
<ProgressRadial value={50} width="w-20" />
<ProgressRadial value={75} width="w-32" />
```

### Table

```svelte
<script lang="ts">
  import { Table, tableMapperValues, type TableSource } from '@skeletonlabs/skeleton';

  const sourceData = [
    { id: 1, name: 'Alice', email: 'alice@example.com', status: 'Active' },
    { id: 2, name: 'Bob', email: 'bob@example.com', status: 'Inactive' },
    { id: 3, name: 'Charlie', email: 'charlie@example.com', status: 'Active' },
  ];

  const tableSimple: TableSource = {
    head: ['Name', 'Email', 'Status'],
    body: tableMapperValues(sourceData, ['name', 'email', 'status']),
    meta: tableMapperValues(sourceData, ['id']),
    foot: ['Total', '', sourceData.length.toString()]
  };

  function onRowSelect(event: CustomEvent<string[]>): void {
    console.log('Selected row meta:', event.detail);
  }
</script>

<Table
  source={tableSimple}
  interactive={true}
  on:selected={onRowSelect}
/>

<!-- With custom cell rendering -->
<div class="table-container">
  <table class="table table-hover">
    <thead>
      <tr>
        <th>Name</th>
        <th>Email</th>
        <th>Status</th>
        <th>Actions</th>
      </tr>
    </thead>
    <tbody>
      {#each sourceData as row}
        <tr>
          <td>{row.name}</td>
          <td>{row.email}</td>
          <td>
            <span class="badge" class:variant-filled-success={row.status === 'Active'} class:variant-filled-error={row.status === 'Inactive'}>
              {row.status}
            </span>
          </td>
          <td>
            <button class="btn btn-sm variant-ghost-surface">Edit</button>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>
</div>
```

### Paginator

```svelte
<script lang="ts">
  import { Paginator, type PaginationSettings } from '@skeletonlabs/skeleton';

  let paginationSettings: PaginationSettings = {
    page: 0,
    limit: 5,
    size: 100,
    amounts: [5, 10, 25, 50]
  };

  function onPageChange(e: CustomEvent): void {
    console.log('Page:', e.detail);
  }

  function onAmountChange(e: CustomEvent): void {
    console.log('Amount:', e.detail);
  }
</script>

<Paginator
  bind:settings={paginationSettings}
  on:page={onPageChange}
  on:amount={onAmountChange}
  showFirstLastButtons={true}
  showPreviousNextButtons={true}
/>

<!-- Pagination with data -->
{#each paginatedData as item}
  <div>{item.name}</div>
{/each}
```

---

## Navigation

### TabGroup

```svelte
<script>
  import { TabGroup, Tab, TabAnchor } from '@skeletonlabs/skeleton';

  let tabSet = $state(0);
</script>

<!-- Tab buttons -->
<TabGroup>
  <Tab bind:group={tabSet} name="tab1" value={0}>Tab 1</Tab>
  <Tab bind:group={tabSet} name="tab2" value={1}>Tab 2</Tab>
  <Tab bind:group={tabSet} name="tab3" value={2}>Tab 3</Tab>

  <svelte:fragment slot="panel">
    {#if tabSet === 0}
      <p>Content for Tab 1</p>
    {:else if tabSet === 1}
      <p>Content for Tab 2</p>
    {:else if tabSet === 2}
      <p>Content for Tab 3</p>
    {/if}
  </svelte:fragment>
</TabGroup>

<!-- Tab anchors (links) -->
<TabGroup>
  <TabAnchor href="/dashboard" selected={$page.url.pathname === '/dashboard'}>
    Dashboard
  </TabAnchor>
  <TabAnchor href="/settings" selected={$page.url.pathname === '/settings'}>
    Settings
  </TabAnchor>
</TabGroup>
```

### Stepper

```svelte
<script lang="ts">
  import { Stepper, Step } from '@skeletonlabs/skeleton';

  function onComplete() {
    console.log('Stepper completed!');
  }
</script>

<Stepper on:complete={onComplete}>
  <Step>
    <svelte:fragment slot="header">Step 1: Account</svelte:fragment>
    <p>Create your account details.</p>
    <input type="text" class="input" placeholder="Username" />
  </Step>

  <Step>
    <svelte:fragment slot="header">Step 2: Profile</svelte:fragment>
    <p>Set up your profile.</p>
    <input type="text" class="input" placeholder="Full Name" />
  </Step>

  <Step locked={!formValid}>
    <svelte:fragment slot="header">Step 3: Review</svelte:fragment>
    <p>Review your information before submitting.</p>
  </Step>
</Stepper>
```

### TreeView

```svelte
<script lang="ts">
  import { TreeView, TreeViewItem } from '@skeletonlabs/skeleton';
</script>

<TreeView>
  <TreeViewItem>
    <svelte:fragment slot="lead">📁</svelte:fragment>
    Documents
    <svelte:fragment slot="children">
      <TreeViewItem>
        <svelte:fragment slot="lead">📄</svelte:fragment>
        report.pdf
      </TreeViewItem>
      <TreeViewItem>
        <svelte:fragment slot="lead">📄</svelte:fragment>
        notes.txt
      </TreeViewItem>
    </svelte:fragment>
  </TreeViewItem>

  <TreeViewItem>
    <svelte:fragment slot="lead">📁</svelte:fragment>
    Images
    <svelte:fragment slot="children">
      <TreeViewItem>
        <svelte:fragment slot="lead">🖼️</svelte:fragment>
        photo.jpg
      </TreeViewItem>
    </svelte:fragment>
  </TreeViewItem>
</TreeView>
```

---

## Form Components

### Autocomplete

```svelte
<script lang="ts">
  import { Autocomplete, popup, type AutocompleteOption, type PopupSettings } from '@skeletonlabs/skeleton';

  const options: AutocompleteOption[] = [
    { label: 'Apple', value: 'apple', meta: { emoji: '🍎' } },
    { label: 'Banana', value: 'banana', meta: { emoji: '🍌' } },
    { label: 'Cherry', value: 'cherry', meta: { emoji: '🍒' } },
    { label: 'Date', value: 'date', meta: { emoji: '📅' } },
  ];

  let inputValue = $state('');
  let inputPopup = $state(false);

  const popupSettings: PopupSettings = {
    event: 'focus-click',
    target: 'popupAutocomplete',
    placement: 'bottom'
  };

  function onSelection(event: CustomEvent<AutocompleteOption>): void {
    inputValue = event.detail.label;
    inputPopup = false;
  }
</script>

<input
  class="input"
  type="search"
  placeholder="Search..."
  bind:value={inputValue}
  use:popup={popupSettings}
/>

<div class="card w-full max-w-sm max-h-48 p-4 overflow-y-auto" data-popup="popupAutocomplete">
  <Autocomplete
    bind:input={inputValue}
    {options}
    on:selection={onSelection}
  />
</div>
```

### InputChip

```svelte
<script lang="ts">
  import { InputChip } from '@skeletonlabs/skeleton';

  let tags = $state(['svelte', 'tailwind', 'skeleton']);
</script>

<InputChip
  bind:value={tags}
  name="tags"
  placeholder="Add tags..."
  chips="variant-filled-primary"
/>

<!-- Display chips -->
<div class="flex gap-2 mt-4">
  {#each tags as tag}
    <span class="chip variant-filled">{tag}</span>
  {/each}
</div>
```

### ListBox

```svelte
<script lang="ts">
  import { ListBox, ListBoxItem } from '@skeletonlabs/skeleton';

  let valueSingle = $state('');
  let valueMultiple = $state<string[]>([]);
</script>

<!-- Single select -->
<ListBox>
  <ListBoxItem bind:group={valueSingle} name="medium" value="books">Books</ListBoxItem>
  <ListBoxItem bind:group={valueSingle} name="medium" value="movies">Movies</ListBoxItem>
  <ListBoxItem bind:group={valueSingle} name="medium" value="tv">Television</ListBoxItem>
</ListBox>

<!-- Multiple select -->
<ListBox multiple>
  <ListBoxItem bind:group={valueMultiple} name="genres" value="action">Action</ListBoxItem>
  <ListBoxItem bind:group={valueMultiple} name="genres" value="comedy">Comedy</ListBoxItem>
  <ListBoxItem bind:group={valueMultiple} name="genres" value="drama">Drama</ListBoxItem>
</ListBox>
```

### RangeSlider

```svelte
<script lang="ts">
  import { RangeSlider } from '@skeletonlabs/skeleton';

  let value = $state(50);
  let range = $state({ min: 25, max: 75 });
</script>

<!-- Single value -->
<RangeSlider name="slider" bind:value min={0} max={100} step={1}>
  <div class="flex justify-between items-center">
    <span>Volume</span>
    <span>{value}</span>
  </div>
</RangeSlider>

<!-- Range (min/max) -->
<RangeSlider name="range" bind:value={range.min} bind:valueHigh={range.max} min={0} max={100} step={5}>
  <div class="flex justify-between items-center">
    <span>Price Range</span>
    <span>${range.min} - ${range.max}</span>
  </div>
</RangeSlider>
```

### SlideToggle

```svelte
<script>
  import { SlideToggle } from '@skeletonlabs/skeleton';

  let enabled = $state(false);
</script>

<SlideToggle name="toggle" bind:checked={enabled} active="bg-primary-500" size="sm">
  {enabled ? 'Enabled' : 'Disabled'}
</SlideToggle>

<!-- Sizes -->
<SlideToggle name="sm" size="sm" />
<SlideToggle name="md" size="md" />
<SlideToggle name="lg" size="lg" />
```

---

## Utility Components

### Popup

```svelte
<script lang="ts">
  import { popup, type PopupSettings } from '@skeletonlabs/skeleton';

  const popupHover: PopupSettings = {
    event: 'hover',
    target: 'popupHover',
    placement: 'top'
  };

  const popupClick: PopupSettings = {
    event: 'click',
    target: 'popupClick',
    placement: 'bottom'
  };
</script>

<!-- Hover tooltip -->
<button class="btn variant-filled" use:popup={popupHover}>Hover Me</button>
<div class="card p-4 variant-filled-secondary" data-popup="popupHover">
  <p>This is a tooltip!</p>
  <div class="arrow variant-filled-secondary" />
</div>

<!-- Click popup -->
<button class="btn variant-filled" use:popup={popupClick}>Click Me</button>
<div class="card p-4 w-72 shadow-xl" data-popup="popupClick">
  <nav class="list-nav">
    <ul>
      <li><a href="/">Option 1</a></li>
      <li><a href="/">Option 2</a></li>
      <li><a href="/">Option 3</a></li>
    </ul>
  </nav>
</div>
```

### Accordion

```svelte
<script>
  import { Accordion, AccordionItem } from '@skeletonlabs/skeleton';
</script>

<Accordion>
  <AccordionItem open>
    <svelte:fragment slot="lead">🔵</svelte:fragment>
    <svelte:fragment slot="summary">First Item</svelte:fragment>
    <svelte:fragment slot="content">
      <p>Content for the first accordion item.</p>
    </svelte:fragment>
  </AccordionItem>

  <AccordionItem>
    <svelte:fragment slot="lead">🟢</svelte:fragment>
    <svelte:fragment slot="summary">Second Item</svelte:fragment>
    <svelte:fragment slot="content">
      <p>Content for the second accordion item.</p>
    </svelte:fragment>
  </AccordionItem>

  <AccordionItem disabled>
    <svelte:fragment slot="summary">Disabled Item</svelte:fragment>
    <svelte:fragment slot="content">
      <p>This item is disabled.</p>
    </svelte:fragment>
  </AccordionItem>
</Accordion>

<!-- Single open mode -->
<Accordion autocollapse>
  <!-- Items... -->
</Accordion>
```

### ConicGradient

```svelte
<script>
  import { ConicGradient, type ConicStop } from '@skeletonlabs/skeleton';

  const conicStops: ConicStop[] = [
    { color: 'transparent', start: 0, end: 25 },
    { color: 'rgb(var(--color-primary-500))', start: 75, end: 100 }
  ];
</script>

<ConicGradient stops={conicStops}>
  <div class="text-center">
    <span class="font-bold text-xl">75%</span>
  </div>
</ConicGradient>
```

---

## Card Component

```svelte
<script>
  import { Card } from '@skeletonlabs/skeleton';
</script>

<!-- Basic card -->
<div class="card p-4">
  <p>Simple card content</p>
</div>

<!-- Structured card -->
<div class="card card-hover overflow-hidden">
  <header class="card-header">
    <h3 class="h3">Card Title</h3>
  </header>

  <section class="p-4">
    <p>Card body content goes here.</p>
  </section>

  <footer class="card-footer flex justify-end gap-2">
    <button class="btn variant-ghost-surface">Cancel</button>
    <button class="btn variant-filled-primary">Submit</button>
  </footer>
</div>

<!-- Card with image -->
<div class="card">
  <img src="/image.jpg" class="rounded-t-container-token" alt="Card image" />
  <div class="p-4 space-y-4">
    <h3 class="h3">Image Card</h3>
    <p>Description text here.</p>
  </div>
</div>

<!-- Card variants -->
<div class="card variant-filled-primary p-4">Primary filled</div>
<div class="card variant-soft-secondary p-4">Secondary soft</div>
<div class="card variant-ringed p-4">Ringed</div>
```

---

## Related Topics

- [Skeleton Basics](basics.md) - Installation and theming
- [Svelte SKILL](../../frontend-frameworks/svelte/SKILL.md) - Svelte patterns
- [Tailwind Utilities](../tailwind/utilities.md) - Tailwind CSS reference
