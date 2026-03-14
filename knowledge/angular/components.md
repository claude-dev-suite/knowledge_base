# Angular Components

> Official Documentation: https://angular.dev/guide/components

## Standalone Components

Starting with Angular 17, `standalone: true` is the default. Standalone components declare their own dependencies via the `imports` array instead of relying on NgModules.

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-user-card',
  imports: [CommonModule, RouterLink],
  template: `
    <div class="user-card">
      <h2>{{ user().name }}</h2>
      <a routerLink="/users/{{ user().id }}">View Profile</a>
    </div>
  `,
  styles: [`
    .user-card {
      padding: 1rem;
      border: 1px solid #e2e8f0;
      border-radius: 0.5rem;
    }
  `]
})
export class UserCardComponent {
  user = input.required<User>();
}
```

### Bootstrapping with Standalone Components

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch(err => console.error(err));
```

```typescript
// app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
  ]
};
```

---

## Component Decorator

### Full @Component Options

```typescript
@Component({
  selector: 'app-dashboard',          // CSS selector for the component
  template: `<h1>Dashboard</h1>`,     // Inline template
  templateUrl: './dashboard.component.html', // External template (use one or the other)
  styles: [`:host { display: block; }`],     // Inline styles
  styleUrl: './dashboard.component.css',     // External style (use one or the other)
  imports: [CommonModule, ChildComponent],   // Standalone dependencies
  providers: [DashboardService],             // Component-level DI providers
  changeDetection: ChangeDetectionStrategy.OnPush, // Change detection strategy
  encapsulation: ViewEncapsulation.Emulated, // Style encapsulation
  host: {                                    // Host element bindings
    'class': 'dashboard-wrapper',
    '[class.active]': 'isActive()',
    '(click)': 'onClick($event)',
    'role': 'main'
  },
  animations: [fadeInAnimation]              // Animation triggers
})
export class DashboardComponent {}
```

### Selector Types

| Selector Type | Example | Usage |
|---------------|---------|-------|
| Element | `'app-card'` | `<app-card>` (most common) |
| Attribute | `'[appTooltip]'` | `<div appTooltip>` (for directives) |
| Class | `'.app-widget'` | `<div class="app-widget">` (rare) |

---

## Signal-Based Inputs

Angular 17+ introduces signal-based inputs using the `input()` function. These are the preferred approach over the `@Input()` decorator.

### input() - Optional Inputs

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-product-card',
  template: `
    <div class="product" [class.featured]="featured()">
      <h3>{{ title() }}</h3>
      <p class="price">{{ price() | currency }}</p>
      @if (discount()) {
        <span class="badge">{{ discount() }}% OFF</span>
      }
    </div>
  `
})
export class ProductCardComponent {
  title = input<string>('Untitled');        // Default value
  price = input<number>(0);                  // Default value with type
  discount = input<number | undefined>();    // Optional, undefined by default
  featured = input(false);                   // Type inferred as boolean
}
```

### input.required() - Required Inputs

```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div class="profile">
      <img [src]="avatarUrl()" [alt]="name()">
      <h2>{{ name() }}</h2>
      <p>{{ email() }}</p>
    </div>
  `
})
export class UserProfileComponent {
  name = input.required<string>();
  email = input.required<string>();
  avatarUrl = input.required<string>();
}
```

Usage in parent template:

```html
<app-user-profile
  [name]="user.name"
  [email]="user.email"
  [avatarUrl]="user.avatar"
/>
```

### Input Transforms

```typescript
import { booleanAttribute, numberAttribute } from '@angular/core';

@Component({
  selector: 'app-paginator',
  template: `<p>Page {{ page() }} of {{ total() }}</p>`
})
export class PaginatorComponent {
  // Transforms string "5" to number 5: <app-paginator page="5">
  page = input(1, { transform: numberAttribute });
  total = input(1, { transform: numberAttribute });

  // Transforms presence to boolean: <app-paginator disabled>
  disabled = input(false, { transform: booleanAttribute });

  // Custom transform
  label = input('', { transform: (v: string) => v.trim().toLowerCase() });
}
```

### Input Aliases

```typescript
@Component({ selector: 'app-tooltip' })
export class TooltipComponent {
  // Use 'message' in template but 'tooltipText' internally
  tooltipText = input.required<string>({ alias: 'message' });
}
```

```html
<app-tooltip [message]="'Hello World'" />
```

---

## Outputs with OutputEmitterRef

Angular 17+ introduces signal-style outputs using the `output()` function.

```typescript
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-search-bar',
  template: `
    <input
      type="text"
      [value]="query"
      (input)="onInput($event)"
      (keyup.enter)="onSearch()"
      placeholder="Search..."
    />
    <button (click)="onSearch()">Search</button>
    <button (click)="onClear()">Clear</button>
  `
})
export class SearchBarComponent {
  query = '';

  // Outputs using output() function
  search = output<string>();              // Emits string
  clear = output<void>();                 // Emits void
  inputChange = output<string>();         // Emits on every keystroke

  onInput(event: Event): void {
    this.query = (event.target as HTMLInputElement).value;
    this.inputChange.emit(this.query);
  }

  onSearch(): void {
    this.search.emit(this.query);
  }

  onClear(): void {
    this.query = '';
    this.clear.emit();
  }
}
```

Usage in parent:

```html
<app-search-bar
  (search)="handleSearch($event)"
  (clear)="handleClear()"
  (inputChange)="handleInputChange($event)"
/>
```

### Output Aliases

```typescript
@Component({ selector: 'app-dropdown' })
export class DropdownComponent {
  selectionChange = output<string>({ alias: 'change' });
}
```

---

## model() - Two-Way Binding

The `model()` function creates a writable signal that supports two-way binding with the `[()]` (banana-in-a-box) syntax.

```typescript
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-rating',
  template: `
    <div class="stars">
      @for (star of stars; track star) {
        <button
          (click)="value.set(star)"
          [class.filled]="star <= value()"
        >
          ★
        </button>
      }
    </div>
    <span>{{ value() }} / 5</span>
  `
})
export class RatingComponent {
  value = model(0);               // Two-way bindable signal
  readonly stars = [1, 2, 3, 4, 5];
}
```

Usage with two-way binding:

```html
<!-- Parent component -->
<app-rating [(value)]="userRating" />
<p>You rated: {{ userRating }}</p>
```

### model.required()

```typescript
@Component({ selector: 'app-toggle' })
export class ToggleComponent {
  checked = model.required<boolean>();
}
```

```html
<app-toggle [(checked)]="isEnabled" />
```

---

## Lifecycle Hooks

| Hook | Timing | Use Case |
|------|--------|----------|
| `ngOnInit` | After first input binding | Fetch data, initialize logic |
| `ngOnChanges` | Before ngOnInit and on input changes | React to input changes (decorator-based) |
| `ngDoCheck` | Every change detection run | Custom change detection logic |
| `ngAfterContentInit` | After content projection | Query projected content |
| `ngAfterContentChecked` | After every content check | Respond to projected content changes |
| `ngAfterViewInit` | After view initialization | Access ViewChild, DOM manipulation |
| `ngAfterViewChecked` | After every view check | Respond to view changes |
| `ngOnDestroy` | Before component destruction | Cleanup subscriptions, timers |

### Lifecycle Example

```typescript
import { Component, OnInit, OnDestroy, AfterViewInit, ViewChild, ElementRef, DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-dashboard',
  template: `
    <div #chartContainer class="chart"></div>
    <div class="data">
      @for (item of items(); track item.id) {
        <app-item [data]="item" />
      }
    </div>
  `
})
export class DashboardComponent implements OnInit, AfterViewInit, OnDestroy {
  private dataService = inject(DataService);
  private destroyRef = inject(DestroyRef);

  @ViewChild('chartContainer') chartContainer!: ElementRef<HTMLDivElement>;

  items = signal<Item[]>([]);
  private chart: Chart | null = null;

  ngOnInit(): void {
    // Preferred: use takeUntilDestroyed for automatic cleanup
    this.dataService.getItems()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(items => this.items.set(items));
  }

  ngAfterViewInit(): void {
    // DOM is ready, safe to initialize third-party libraries
    this.chart = new Chart(this.chartContainer.nativeElement, {
      type: 'bar',
      data: this.getChartData()
    });
  }

  ngOnDestroy(): void {
    // Clean up non-RxJS resources
    this.chart?.destroy();
  }
}
```

### afterNextRender and afterRender

These are for code that must run in the browser after rendering. They are safe for SSR because they only execute in the browser.

```typescript
import { Component, afterNextRender, afterRender, ElementRef, viewChild } from '@angular/core';

@Component({
  selector: 'app-canvas',
  template: `<canvas #myCanvas width="800" height="600"></canvas>`
})
export class CanvasComponent {
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('myCanvas');
  private ctx: CanvasRenderingContext2D | null = null;

  constructor() {
    // Runs ONCE after the next render cycle (browser only)
    afterNextRender(() => {
      this.ctx = this.canvas().nativeElement.getContext('2d');
      this.initializeCanvas();
    });

    // Runs after EVERY render cycle (browser only)
    afterRender(() => {
      this.updateCanvasSize();
    });
  }

  private initializeCanvas(): void {
    if (!this.ctx) return;
    this.ctx.fillStyle = '#000';
    this.ctx.fillRect(0, 0, 800, 600);
  }

  private updateCanvasSize(): void {
    const el = this.canvas().nativeElement;
    el.width = el.parentElement?.clientWidth ?? 800;
  }
}
```

---

## Content Projection

### Single Slot Projection

```typescript
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-body">
        <ng-content />
      </div>
    </div>
  `
})
export class CardComponent {}
```

```html
<app-card>
  <h2>Title</h2>
  <p>Content goes here.</p>
</app-card>
```

### Multi-Slot Projection

```typescript
@Component({
  selector: 'app-dialog',
  template: `
    <div class="dialog-overlay">
      <div class="dialog">
        <header class="dialog-header">
          <ng-content select="[dialog-title]" />
          <button (click)="close.emit()">X</button>
        </header>
        <div class="dialog-body">
          <ng-content />
        </div>
        <footer class="dialog-footer">
          <ng-content select="[dialog-actions]" />
        </footer>
      </div>
    </div>
  `
})
export class DialogComponent {
  close = output<void>();
}
```

```html
<app-dialog (close)="closeDialog()">
  <h2 dialog-title>Confirm Delete</h2>
  <p>Are you sure you want to delete this item?</p>
  <div dialog-actions>
    <button (click)="cancel()">Cancel</button>
    <button (click)="confirm()">Delete</button>
  </div>
</app-dialog>
```

### @ContentChild and @ContentChildren

```typescript
import { Component, ContentChild, ContentChildren, QueryList, AfterContentInit, TemplateRef } from '@angular/core';

@Component({
  selector: 'app-tab-group',
  template: `
    <div class="tabs">
      @for (tab of tabs; track tab.label) {
        <button
          [class.active]="tab === activeTab"
          (click)="activeTab = tab"
        >
          {{ tab.label }}
        </button>
      }
    </div>
    <div class="tab-content">
      @if (activeTab) {
        <ng-container [ngTemplateOutlet]="activeTab.content" />
      }
    </div>
  `,
  imports: [NgTemplateOutlet]
})
export class TabGroupComponent implements AfterContentInit {
  @ContentChildren(TabComponent) tabList!: QueryList<TabComponent>;

  tabs: TabComponent[] = [];
  activeTab: TabComponent | null = null;

  ngAfterContentInit(): void {
    this.tabs = this.tabList.toArray();
    this.activeTab = this.tabs[0] ?? null;

    // React to dynamic tab additions/removals
    this.tabList.changes.subscribe(() => {
      this.tabs = this.tabList.toArray();
    });
  }
}

@Component({
  selector: 'app-tab',
  template: `<ng-template #content><ng-content /></ng-template>`
})
export class TabComponent {
  @ViewChild('content') content!: TemplateRef<unknown>;
  label = input.required<string>();
}
```

---

## View Queries

### Signal-Based View Queries (Preferred in Angular 17+)

```typescript
import { Component, viewChild, viewChildren, ElementRef, AfterViewInit } from '@angular/core';

@Component({
  selector: 'app-form',
  template: `
    <input #nameInput type="text" placeholder="Name" />
    <input #emailInput type="email" placeholder="Email" />
    <div class="items">
      @for (item of items(); track item.id) {
        <app-item-row #itemRow [data]="item" />
      }
    </div>
  `,
  imports: [ItemRowComponent]
})
export class FormComponent implements AfterViewInit {
  // Signal-based view queries
  nameInput = viewChild.required<ElementRef<HTMLInputElement>>('nameInput');
  emailInput = viewChild<ElementRef<HTMLInputElement>>('emailInput');
  itemRows = viewChildren(ItemRowComponent);

  items = signal<Item[]>([]);

  ngAfterViewInit(): void {
    // Access signal-based view queries
    this.nameInput().nativeElement.focus();
    console.log('Item rows:', this.itemRows().length);
  }
}
```

### Decorator-Based View Queries

```typescript
@Component({
  selector: 'app-gallery',
  template: `
    <div class="gallery">
      @for (image of images(); track image.id) {
        <img #imageEl [src]="image.url" [alt]="image.alt" />
      }
    </div>
  `
})
export class GalleryComponent implements AfterViewInit {
  @ViewChild('imageEl') firstImage!: ElementRef<HTMLImageElement>;
  @ViewChildren('imageEl') allImages!: QueryList<ElementRef<HTMLImageElement>>;

  images = signal<Image[]>([]);

  ngAfterViewInit(): void {
    console.log('First image:', this.firstImage?.nativeElement.src);
    console.log('Total images:', this.allImages.length);
  }
}
```

---

## Change Detection

### OnPush Strategy

```typescript
import { Component, ChangeDetectionStrategy, ChangeDetectorRef, inject } from '@angular/core';

@Component({
  selector: 'app-todo-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h2>Todos ({{ todos.length }})</h2>
    <ul>
      @for (todo of todos; track todo.id) {
        <li [class.done]="todo.completed">{{ todo.title }}</li>
      }
    </ul>
    <button (click)="refresh()">Refresh</button>
  `
})
export class TodoListComponent {
  private cdr = inject(ChangeDetectorRef);
  private todoService = inject(TodoService);

  todos: Todo[] = [];

  refresh(): void {
    this.todoService.getTodos().subscribe(todos => {
      this.todos = todos; // New reference triggers OnPush
      // If mutating instead: this.cdr.markForCheck();
    });
  }
}
```

### Signals with Change Detection

Signals automatically trigger change detection in OnPush components, making them the preferred approach:

```typescript
@Component({
  selector: 'app-counter',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Count: {{ count() }}</p>
    <p>Doubled: {{ doubled() }}</p>
    <button (click)="increment()">+1</button>
  `
})
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);

  increment(): void {
    this.count.update(c => c + 1); // Automatically triggers CD
  }
}
```

---

## Component Communication Patterns

### Parent to Child (Inputs)

```typescript
// Parent
@Component({
  template: `<app-child [data]="parentData()" [config]="config" />`
})
export class ParentComponent {
  parentData = signal({ name: 'Angular' });
  config = { theme: 'dark' };
}

// Child
@Component({ selector: 'app-child' })
export class ChildComponent {
  data = input.required<{ name: string }>();
  config = input<{ theme: string }>({ theme: 'light' });
}
```

### Child to Parent (Outputs)

```typescript
// Child
@Component({
  selector: 'app-child',
  template: `<button (click)="notify.emit('hello')">Notify</button>`
})
export class ChildComponent {
  notify = output<string>();
}

// Parent
@Component({
  template: `<app-child (notify)="handleNotify($event)" />`
})
export class ParentComponent {
  handleNotify(message: string): void {
    console.log(message);
  }
}
```

### Service-Based Communication

```typescript
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notificationsSubject = new Subject<Notification>();
  notifications$ = this.notificationsSubject.asObservable();

  send(notification: Notification): void {
    this.notificationsSubject.next(notification);
  }
}

// Any component can inject and use
@Component({ /* ... */ })
export class HeaderComponent {
  private notificationService = inject(NotificationService);
  private destroyRef = inject(DestroyRef);

  notifications = signal<Notification[]>([]);

  constructor() {
    this.notificationService.notifications$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(n => this.notifications.update(list => [...list, n]));
  }
}
```

---

## Host Bindings and Listeners

### Using the host Property (Preferred)

```typescript
@Component({
  selector: 'app-button',
  host: {
    'role': 'button',
    '[attr.aria-disabled]': 'disabled()',
    '[class.primary]': 'variant() === "primary"',
    '[class.secondary]': 'variant() === "secondary"',
    '[style.opacity]': 'disabled() ? "0.5" : "1"',
    '[tabindex]': 'disabled() ? -1 : 0',
    '(click)': 'handleClick($event)',
    '(keydown.enter)': 'handleClick($event)',
    '(keydown.space)': 'handleClick($event)',
  },
  template: `<ng-content />`
})
export class ButtonComponent {
  variant = input<'primary' | 'secondary'>('primary');
  disabled = input(false, { transform: booleanAttribute });
  clicked = output<void>();

  handleClick(event: Event): void {
    if (!this.disabled()) {
      this.clicked.emit();
    }
  }
}
```

### Using @HostBinding and @HostListener (Legacy)

```typescript
@Component({
  selector: 'app-resizable',
  template: `<ng-content />`
})
export class ResizableComponent {
  @HostBinding('class.dragging') isDragging = false;
  @HostBinding('style.width.px') width = 200;

  @HostListener('mousedown', ['$event'])
  onMouseDown(event: MouseEvent): void {
    this.isDragging = true;
  }

  @HostListener('document:mouseup')
  onMouseUp(): void {
    this.isDragging = false;
  }
}
```

---

## Best Practices

### Component Design

| Practice | Recommendation |
|----------|---------------|
| Standalone | Always use standalone components (default in Angular 17+) |
| Inputs | Prefer `input()` and `input.required()` over `@Input()` |
| Outputs | Prefer `output()` over `@Output()` with EventEmitter |
| Change Detection | Use `OnPush` with signals for optimal performance |
| Templates | Use `@if`, `@for`, `@switch` control flow (Angular 17+) |
| Style Encapsulation | Keep default `Emulated` unless you need `None` |
| Component Size | Keep components focused on a single responsibility |
| Smart vs Dumb | Separate container (smart) from presentational (dumb) components |

### Template Best Practices

```typescript
// Use new control flow syntax (Angular 17+)
@Component({
  template: `
    <!-- Conditional rendering -->
    @if (user(); as user) {
      <app-user-card [user]="user" />
    } @else {
      <app-login-prompt />
    }

    <!-- Iteration with track -->
    @for (item of items(); track item.id) {
      <app-item [data]="item" />
    } @empty {
      <p>No items found.</p>
    }

    <!-- Switch -->
    @switch (status()) {
      @case ('loading') { <app-spinner /> }
      @case ('error') { <app-error [message]="errorMessage()" /> }
      @case ('success') { <app-content [data]="data()" /> }
    }
  `
})
export class SmartComponent {}
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Mutating OnPush inputs | Component does not re-render | Always create new object references or use signals |
| Accessing ViewChild too early | `undefined` in constructor/ngOnInit | Use `ngAfterViewInit` or signal-based `viewChild()` |
| Missing `track` in `@for` | Poor rendering performance | Always provide a unique `track` expression |
| Forgetting imports array | Template compilation errors | Add all used components, directives, pipes to `imports` |
| Memory leaks from subscriptions | Subscriptions persist after destroy | Use `takeUntilDestroyed()` or `DestroyRef` |
| Heavy computation in templates | Recalculates every change detection | Use `computed()` signals or pure pipes |
| Direct DOM manipulation | Breaks SSR and encapsulation | Use template bindings, `Renderer2`, or `afterNextRender` |
| Large component files | Hard to maintain and test | Extract logic into services, split into sub-components |
