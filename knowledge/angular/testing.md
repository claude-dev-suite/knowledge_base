# Angular Testing

> Official Documentation: https://angular.dev/guide/testing

## TestBed Configuration

TestBed is Angular's primary testing utility for configuring and creating test modules.

### Basic TestBed Setup

```typescript
import { TestBed, ComponentFixture } from '@angular/core/testing';

describe('UserProfileComponent', () => {
  let component: UserProfileComponent;
  let fixture: ComponentFixture<UserProfileComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserProfileComponent],  // Standalone components go in imports
      providers: [
        { provide: UserService, useClass: MockUserService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(UserProfileComponent);
    component = fixture.componentInstance;
    fixture.detectChanges(); // Trigger initial change detection
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });
});
```

### Overriding Component Dependencies

```typescript
beforeEach(async () => {
  await TestBed.configureTestingModule({
    imports: [ProductListComponent]
  })
  .overrideComponent(ProductListComponent, {
    set: {
      providers: [
        { provide: ProductService, useClass: MockProductService }
      ]
    }
  })
  .compileComponents();
});
```

---

## Component Testing

### Testing Component Rendering

```typescript
@Component({
  selector: 'app-greeting',
  template: `
    <h1>Hello, {{ name() }}!</h1>
    @if (showDetails()) {
      <p class="details">{{ details() }}</p>
    }
  `
})
export class GreetingComponent {
  name = input.required<string>();
  showDetails = input(false);
  details = input('');
}

describe('GreetingComponent', () => {
  let fixture: ComponentFixture<GreetingComponent>;
  let component: GreetingComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [GreetingComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(GreetingComponent);
    component = fixture.componentInstance;
  });

  it('should display the name', () => {
    fixture.componentRef.setInput('name', 'Angular');
    fixture.detectChanges();

    const h1 = fixture.nativeElement.querySelector('h1');
    expect(h1.textContent).toBe('Hello, Angular!');
  });

  it('should show details when showDetails is true', () => {
    fixture.componentRef.setInput('name', 'Test');
    fixture.componentRef.setInput('showDetails', true);
    fixture.componentRef.setInput('details', 'Some details');
    fixture.detectChanges();

    const details = fixture.nativeElement.querySelector('.details');
    expect(details).toBeTruthy();
    expect(details.textContent).toBe('Some details');
  });

  it('should hide details by default', () => {
    fixture.componentRef.setInput('name', 'Test');
    fixture.detectChanges();

    const details = fixture.nativeElement.querySelector('.details');
    expect(details).toBeNull();
  });
});
```

### Testing Component with Events

```typescript
@Component({
  selector: 'app-counter',
  template: `
    <span class="count">{{ count() }}</span>
    <button class="increment" (click)="increment()">+</button>
    <button class="decrement" (click)="decrement()">-</button>
  `
})
export class CounterComponent {
  count = signal(0);
  countChange = output<number>();

  increment(): void {
    this.count.update(c => c + 1);
    this.countChange.emit(this.count());
  }

  decrement(): void {
    this.count.update(c => c - 1);
    this.countChange.emit(this.count());
  }
}

describe('CounterComponent', () => {
  let fixture: ComponentFixture<CounterComponent>;
  let component: CounterComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [CounterComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should increment count on button click', () => {
    const button = fixture.nativeElement.querySelector('.increment') as HTMLButtonElement;
    button.click();
    fixture.detectChanges();

    const count = fixture.nativeElement.querySelector('.count');
    expect(count.textContent).toBe('1');
  });

  it('should emit countChange on increment', () => {
    let emittedValue: number | undefined;
    component.countChange.subscribe((value: number) => {
      emittedValue = value;
    });

    const button = fixture.nativeElement.querySelector('.increment') as HTMLButtonElement;
    button.click();

    expect(emittedValue).toBe(1);
  });

  it('should decrement count', () => {
    component.count.set(5);
    fixture.detectChanges();

    const button = fixture.nativeElement.querySelector('.decrement') as HTMLButtonElement;
    button.click();
    fixture.detectChanges();

    expect(component.count()).toBe(4);
  });
});
```

### Testing with DebugElement

```typescript
import { By } from '@angular/platform-browser';
import { DebugElement } from '@angular/core';

describe('TodoListComponent', () => {
  let fixture: ComponentFixture<TodoListComponent>;
  let debugElement: DebugElement;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TodoListComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(TodoListComponent);
    debugElement = fixture.debugElement;
    fixture.detectChanges();
  });

  it('should render todo items', () => {
    const component = fixture.componentInstance;
    component.todos.set([
      { id: 1, title: 'Test 1', completed: false },
      { id: 2, title: 'Test 2', completed: true }
    ]);
    fixture.detectChanges();

    const items = debugElement.queryAll(By.css('.todo-item'));
    expect(items.length).toBe(2);
    expect(items[0].nativeElement.textContent).toContain('Test 1');
  });

  it('should apply completed class', () => {
    const component = fixture.componentInstance;
    component.todos.set([
      { id: 1, title: 'Done', completed: true }
    ]);
    fixture.detectChanges();

    const item = debugElement.query(By.css('.todo-item'));
    expect(item.classes['completed']).toBeTrue();
  });

  it('should trigger event on child component', () => {
    // Query child component
    const childDebugEl = debugElement.query(By.directive(TodoItemComponent));
    const childComponent = childDebugEl.componentInstance as TodoItemComponent;

    spyOn(fixture.componentInstance, 'onTodoToggle');

    // Emit output from child
    childComponent.toggle.emit(1);

    expect(fixture.componentInstance.onTodoToggle).toHaveBeenCalledWith(1);
  });
});
```

---

## Service Testing

### Testing Services Without TestBed

```typescript
describe('CalculatorService', () => {
  let service: CalculatorService;

  beforeEach(() => {
    service = new CalculatorService();
  });

  it('should add two numbers', () => {
    expect(service.add(2, 3)).toBe(5);
  });

  it('should throw for division by zero', () => {
    expect(() => service.divide(10, 0)).toThrowError('Division by zero');
  });
});
```

### Testing Services With TestBed

```typescript
describe('ProductService', () => {
  let service: ProductService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        ProductService,
        provideHttpClient(),
        provideHttpClientTesting()
      ]
    });

    service = TestBed.inject(ProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should fetch all products', () => {
    const mockProducts: Product[] = [
      { id: 1, name: 'Widget', price: 9.99 },
      { id: 2, name: 'Gadget', price: 19.99 }
    ];

    service.getAll().subscribe(products => {
      expect(products.length).toBe(2);
      expect(products[0].name).toBe('Widget');
    });

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('GET');
    req.flush(mockProducts);
  });

  it('should create a product', () => {
    const newProduct = { name: 'New Item', price: 29.99 };

    service.create(newProduct).subscribe(product => {
      expect(product.id).toBe(3);
      expect(product.name).toBe('New Item');
    });

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newProduct);
    req.flush({ id: 3, ...newProduct });
  });

  it('should handle 404 error', () => {
    service.getById(999).subscribe({
      error: (err) => {
        expect(err.status).toBe(404);
      }
    });

    const req = httpMock.expectOne('/api/products/999');
    req.flush('Not Found', { status: 404, statusText: 'Not Found' });
  });
});
```

### Testing Services with Dependencies

```typescript
describe('AuthService', () => {
  let service: AuthService;
  let httpMock: HttpTestingController;
  let storageSpy: jasmine.SpyObj<StorageService>;

  beforeEach(() => {
    storageSpy = jasmine.createSpyObj('StorageService', ['get', 'set', 'remove']);

    TestBed.configureTestingModule({
      providers: [
        AuthService,
        provideHttpClient(),
        provideHttpClientTesting(),
        { provide: StorageService, useValue: storageSpy }
      ]
    });

    service = TestBed.inject(AuthService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('should store token on login', () => {
    const credentials = { email: 'test@example.com', password: 'secret' };

    service.login(credentials).subscribe(user => {
      expect(user.email).toBe('test@example.com');
    });

    const req = httpMock.expectOne('/api/auth/login');
    req.flush({ user: { email: 'test@example.com' }, token: 'abc123' });

    expect(storageSpy.set).toHaveBeenCalledWith('auth_token', 'abc123');
  });

  it('should clear token on logout', () => {
    service.logout();
    expect(storageSpy.remove).toHaveBeenCalledWith('auth_token');
  });
});
```

---

## Testing with Signals

### Testing Signal-Based Components

```typescript
@Component({
  selector: 'app-search',
  template: `
    <input (input)="query.set(getInputValue($event))" [value]="query()" />
    <p class="result-count">{{ resultCount() }} results</p>
    @for (item of filteredItems(); track item.id) {
      <div class="item">{{ item.name }}</div>
    }
  `
})
export class SearchComponent {
  items = input<Item[]>([]);
  query = signal('');

  filteredItems = computed(() => {
    const q = this.query().toLowerCase();
    return this.items().filter(i => i.name.toLowerCase().includes(q));
  });

  resultCount = computed(() => this.filteredItems().length);

  getInputValue(event: Event): string {
    return (event.target as HTMLInputElement).value;
  }
}

describe('SearchComponent', () => {
  let fixture: ComponentFixture<SearchComponent>;
  let component: SearchComponent;

  const mockItems: Item[] = [
    { id: 1, name: 'Apple' },
    { id: 2, name: 'Banana' },
    { id: 3, name: 'Apricot' }
  ];

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [SearchComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(SearchComponent);
    component = fixture.componentInstance;
  });

  it('should show all items when no query', () => {
    fixture.componentRef.setInput('items', mockItems);
    fixture.detectChanges();

    expect(component.filteredItems().length).toBe(3);
    const items = fixture.nativeElement.querySelectorAll('.item');
    expect(items.length).toBe(3);
  });

  it('should filter items based on query signal', () => {
    fixture.componentRef.setInput('items', mockItems);
    component.query.set('ap');
    fixture.detectChanges();

    expect(component.filteredItems().length).toBe(2);  // Apple, Apricot
    expect(component.resultCount()).toBe(2);

    const countEl = fixture.nativeElement.querySelector('.result-count');
    expect(countEl.textContent).toContain('2 results');
  });

  it('should update when input signal changes', () => {
    fixture.componentRef.setInput('items', [mockItems[0]]);
    fixture.detectChanges();
    expect(component.filteredItems().length).toBe(1);

    fixture.componentRef.setInput('items', mockItems);
    fixture.detectChanges();
    expect(component.filteredItems().length).toBe(3);
  });
});
```

### Testing Signal-Based Services

```typescript
describe('CartStore', () => {
  let store: CartStore;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [CartStore] });
    store = TestBed.inject(CartStore);
  });

  it('should start with empty cart', () => {
    expect(store.items().length).toBe(0);
    expect(store.totalPrice()).toBe(0);
    expect(store.itemCount()).toBe(0);
  });

  it('should add items and update computed values', () => {
    store.addItem({ id: 1, name: 'Widget', price: 10, quantity: 2 });

    expect(store.items().length).toBe(1);
    expect(store.totalPrice()).toBe(20);
    expect(store.itemCount()).toBe(2);
  });

  it('should remove items', () => {
    store.addItem({ id: 1, name: 'Widget', price: 10, quantity: 1 });
    store.addItem({ id: 2, name: 'Gadget', price: 20, quantity: 1 });

    store.removeItem(1);

    expect(store.items().length).toBe(1);
    expect(store.totalPrice()).toBe(20);
  });
});
```

---

## HTTP Testing

### Using HttpTestingController

```typescript
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('ApiService', () => {
  let service: ApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        ApiService,
        provideHttpClient(),
        provideHttpClientTesting()
      ]
    });

    service = TestBed.inject(ApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // Fail test if there are outstanding requests
  });

  it('should send correct query parameters', () => {
    service.search('angular', 1, 20).subscribe();

    const req = httpMock.expectOne(
      r => r.url === '/api/search' && r.params.get('q') === 'angular'
    );
    expect(req.request.params.get('page')).toBe('1');
    expect(req.request.params.get('size')).toBe('20');
    req.flush({ results: [] });
  });

  it('should send correct headers', () => {
    service.getProtectedResource().subscribe();

    const req = httpMock.expectOne('/api/protected');
    expect(req.request.headers.get('Authorization')).toBeTruthy();
    req.flush({});
  });

  it('should handle multiple concurrent requests', () => {
    service.getUsers().subscribe();
    service.getProducts().subscribe();

    const requests = httpMock.match(r => r.url.startsWith('/api/'));
    expect(requests.length).toBe(2);

    requests[0].flush([{ id: 1, name: 'User' }]);
    requests[1].flush([{ id: 1, name: 'Product' }]);
  });
});
```

### Testing Interceptors

```typescript
describe('authInterceptor', () => {
  let httpMock: HttpTestingController;
  let http: HttpClient;
  let authService: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    authService = jasmine.createSpyObj('AuthService', ['getAccessToken']);

    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withInterceptors([authInterceptor])),
        provideHttpClientTesting(),
        { provide: AuthService, useValue: authService }
      ]
    });

    http = TestBed.inject(HttpClient);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should add Authorization header when token exists', () => {
    authService.getAccessToken.and.returnValue('my-token');

    http.get('/api/data').subscribe();

    const req = httpMock.expectOne('/api/data');
    expect(req.request.headers.get('Authorization')).toBe('Bearer my-token');
    req.flush({});
  });

  it('should not add header when no token', () => {
    authService.getAccessToken.and.returnValue(null);

    http.get('/api/data').subscribe();

    const req = httpMock.expectOne('/api/data');
    expect(req.request.headers.has('Authorization')).toBeFalse();
    req.flush({});
  });
});
```

---

## Router Testing

### Using RouterTestingHarness

```typescript
import { RouterTestingHarness } from '@angular/router/testing';
import { provideRouter } from '@angular/router';

describe('ProductDetailComponent', () => {
  let harness: RouterTestingHarness;

  beforeEach(async () => {
    TestBed.configureTestingModule({
      providers: [
        provideRouter([
          { path: 'products/:id', component: ProductDetailComponent }
        ]),
        { provide: ProductService, useClass: MockProductService }
      ]
    });

    harness = await RouterTestingHarness.create();
  });

  it('should load product by route param', async () => {
    const component = await harness.navigateByUrl(
      '/products/42',
      ProductDetailComponent
    );
    harness.detectChanges();

    expect(component.product()?.id).toBe(42);
  });

  it('should navigate to products list', async () => {
    const component = await harness.navigateByUrl(
      '/products/1',
      ProductDetailComponent
    );

    component.goBack();

    expect(TestBed.inject(Router).url).toBe('/products');
  });
});
```

### Testing Route Guards

```typescript
describe('authGuard', () => {
  let authService: jasmine.SpyObj<AuthService>;
  let router: Router;

  beforeEach(() => {
    authService = jasmine.createSpyObj('AuthService', ['isAuthenticated']);

    TestBed.configureTestingModule({
      providers: [
        provideRouter([
          { path: 'login', component: LoginComponent },
          {
            path: 'dashboard',
            component: DashboardComponent,
            canActivate: [authGuard]
          }
        ]),
        { provide: AuthService, useValue: authService }
      ]
    });

    router = TestBed.inject(Router);
  });

  it('should allow access when authenticated', async () => {
    authService.isAuthenticated.and.returnValue(true);

    await router.navigate(['/dashboard']);

    expect(router.url).toBe('/dashboard');
  });

  it('should redirect to login when not authenticated', async () => {
    authService.isAuthenticated.and.returnValue(false);

    await router.navigate(['/dashboard']);

    expect(router.url).toBe('/login');
  });
});
```

---

## Testing Pipes and Directives

### Testing Pipes

```typescript
@Pipe({ name: 'truncate', standalone: true })
export class TruncatePipe implements PipeTransform {
  transform(value: string, maxLength: number = 50, suffix: string = '...'): string {
    if (!value || value.length <= maxLength) return value;
    return value.substring(0, maxLength) + suffix;
  }
}

describe('TruncatePipe', () => {
  let pipe: TruncatePipe;

  beforeEach(() => {
    pipe = new TruncatePipe();
  });

  it('should return original string if shorter than max', () => {
    expect(pipe.transform('Hello', 10)).toBe('Hello');
  });

  it('should truncate long strings with default suffix', () => {
    const result = pipe.transform('This is a very long string', 10);
    expect(result).toBe('This is a ...');
    expect(result.length).toBe(13);
  });

  it('should use custom suffix', () => {
    expect(pipe.transform('Hello World', 5, ' [more]')).toBe('Hello [more]');
  });

  it('should handle null/empty values', () => {
    expect(pipe.transform('', 10)).toBe('');
    expect(pipe.transform(null as any, 10)).toBeFalsy();
  });
});
```

### Testing Directives

```typescript
@Directive({
  selector: '[appHighlight]',
  standalone: true,
  host: {
    '(mouseenter)': 'onMouseEnter()',
    '(mouseleave)': 'onMouseLeave()',
    '[style.backgroundColor]': 'bgColor'
  }
})
export class HighlightDirective {
  color = input('yellow', { alias: 'appHighlight' });
  bgColor = '';

  onMouseEnter(): void {
    this.bgColor = this.color();
  }

  onMouseLeave(): void {
    this.bgColor = '';
  }
}

// Test with a host component
@Component({
  template: `
    <p appHighlight="cyan" class="test">Hover me</p>
    <p appHighlight class="default">Default color</p>
  `,
  imports: [HighlightDirective]
})
class TestHostComponent {}

describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestHostComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TestHostComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(TestHostComponent);
    fixture.detectChanges();
  });

  it('should highlight on mouseenter with custom color', () => {
    const el = fixture.debugElement.query(By.css('.test'));
    el.triggerEventHandler('mouseenter', null);
    fixture.detectChanges();

    expect(el.nativeElement.style.backgroundColor).toBe('cyan');
  });

  it('should remove highlight on mouseleave', () => {
    const el = fixture.debugElement.query(By.css('.test'));
    el.triggerEventHandler('mouseenter', null);
    el.triggerEventHandler('mouseleave', null);
    fixture.detectChanges();

    expect(el.nativeElement.style.backgroundColor).toBe('');
  });

  it('should use default color when none specified', () => {
    const el = fixture.debugElement.query(By.css('.default'));
    el.triggerEventHandler('mouseenter', null);
    fixture.detectChanges();

    expect(el.nativeElement.style.backgroundColor).toBe('yellow');
  });
});
```

---

## Async Testing

### fakeAsync and tick

```typescript
import { fakeAsync, tick, flush } from '@angular/core/testing';

describe('DebounceSearchComponent', () => {
  it('should debounce search input', fakeAsync(() => {
    fixture.componentRef.setInput('items', mockItems);
    fixture.detectChanges();

    const input = fixture.nativeElement.querySelector('input');
    input.value = 'ang';
    input.dispatchEvent(new Event('input'));

    // Results should NOT be updated yet (debounce is 300ms)
    expect(component.results().length).toBe(0);

    tick(300); // Advance time by 300ms
    fixture.detectChanges();

    // Now results should be updated
    expect(component.results().length).toBeGreaterThan(0);
  }));

  it('should cancel pending timer on destroy', fakeAsync(() => {
    component.startPolling();
    tick(1000);

    fixture.destroy();
    flush(); // Flush all remaining timers
    // No error means timers were properly cleaned up
  }));
});
```

### waitForAsync

```typescript
import { waitForAsync } from '@angular/core/testing';

describe('AsyncDataComponent', () => {
  it('should load data from service', waitForAsync(() => {
    const service = TestBed.inject(DataService);
    spyOn(service, 'getData').and.returnValue(
      of([{ id: 1, name: 'Test' }]).pipe(delay(100))
    );

    fixture.detectChanges();

    fixture.whenStable().then(() => {
      fixture.detectChanges();
      const items = fixture.nativeElement.querySelectorAll('.item');
      expect(items.length).toBe(1);
    });
  }));
});
```

---

## Mocking Dependencies

### Jasmine Spies

```typescript
// Creating spy objects
const userServiceSpy = jasmine.createSpyObj('UserService', [
  'getAll', 'getById', 'create', 'update', 'delete'
]);

// Setting return values
userServiceSpy.getAll.and.returnValue(of([{ id: 1, name: 'Test' }]));
userServiceSpy.getById.and.returnValue(of({ id: 1, name: 'Test' }));
userServiceSpy.create.and.returnValue(of({ id: 2, name: 'New' }));

// Using in TestBed
providers: [
  { provide: UserService, useValue: userServiceSpy }
]

// Verifying calls
expect(userServiceSpy.getAll).toHaveBeenCalledTimes(1);
expect(userServiceSpy.create).toHaveBeenCalledWith({ name: 'New' });
```

### Mock Classes

```typescript
class MockUserService {
  private users = signal<User[]>([
    { id: 1, name: 'Alice', email: 'alice@example.com' },
    { id: 2, name: 'Bob', email: 'bob@example.com' }
  ]);

  getAll(): Observable<User[]> {
    return of(this.users());
  }

  getById(id: number): Observable<User | undefined> {
    return of(this.users().find(u => u.id === id));
  }

  create(user: Partial<User>): Observable<User> {
    const newUser = { ...user, id: Date.now() } as User;
    this.users.update(list => [...list, newUser]);
    return of(newUser);
  }
}

// Usage
providers: [
  { provide: UserService, useClass: MockUserService }
]
```

---

## Testing Observables

```typescript
describe('DataService', () => {
  it('should emit values in sequence', (done) => {
    const service = TestBed.inject(DataService);
    const emitted: number[] = [];

    service.getStream().subscribe({
      next: (value) => emitted.push(value),
      complete: () => {
        expect(emitted).toEqual([1, 2, 3]);
        done();
      }
    });
  });

  it('should handle error observable', () => {
    const service = TestBed.inject(DataService);
    spyOn(service, 'riskyOperation').and.returnValue(
      throwError(() => new Error('Test error'))
    );

    service.riskyOperation().subscribe({
      error: (err) => {
        expect(err.message).toBe('Test error');
      }
    });
  });

  it('should test marble diagrams (with jasmine-marbles)', () => {
    // If using jasmine-marbles or rxjs/testing
    const scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    scheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('a-b-c|', { a: 1, b: 2, c: 3 });
      const result$ = source$.pipe(map(x => x * 10));
      expectObservable(result$).toBe('a-b-c|', { a: 10, b: 20, c: 30 });
    });
  });
});
```

---

## Component Harness (Angular CDK)

Component harnesses provide a way to test components through a stable, reusable API.

```typescript
import { HarnessLoader } from '@angular/cdk/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { MatButtonHarness } from '@angular/material/button/testing';
import { MatInputHarness } from '@angular/material/input/testing';

describe('LoginComponent (with harnesses)', () => {
  let loader: HarnessLoader;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [LoginComponent]
    }).compileComponents();

    const fixture = TestBed.createComponent(LoginComponent);
    loader = TestbedHarnessEnvironment.loader(fixture);
  });

  it('should fill and submit form', async () => {
    const emailInput = await loader.getHarness(
      MatInputHarness.with({ selector: '[formControlName="email"]' })
    );
    const passwordInput = await loader.getHarness(
      MatInputHarness.with({ selector: '[formControlName="password"]' })
    );
    const submitButton = await loader.getHarness(
      MatButtonHarness.with({ text: 'Login' })
    );

    await emailInput.setValue('test@example.com');
    await passwordInput.setValue('password123');

    expect(await submitButton.isDisabled()).toBeFalse();
    await submitButton.click();
  });
});
```

---

## Best Practices

| Practice | Recommendation |
|----------|---------------|
| Use `setInput()` for signal inputs | Use `fixture.componentRef.setInput('name', value)` |
| Isolate unit tests | Mock dependencies, test one thing at a time |
| Use `provideHttpClientTesting()` | Never make real HTTP calls in unit tests |
| Call `httpMock.verify()` | Always verify no outstanding requests in `afterEach` |
| Use `fakeAsync` for timers | Control time with `tick()` and `flush()` |
| Prefer testing behavior over implementation | Test what users see, not internal state |
| Create mock services as classes | Easier to maintain than spy objects for complex services |
| Use `By.directive()` to query child components | More robust than CSS selectors for component queries |
| Always call `fixture.detectChanges()` | After changing inputs or signals, trigger change detection |

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Not calling `detectChanges()` | DOM does not reflect signal/input changes | Call `fixture.detectChanges()` after state changes |
| Using `@Input()` setter in tests | Signal inputs cannot be set with assignment | Use `fixture.componentRef.setInput()` |
| Missing `compileComponents()` | Template compilation errors in tests | Always `await TestBed.configureTestingModule(...).compileComponents()` |
| Testing implementation details | Tests break on refactoring | Test observable behavior (DOM output, public API) |
| Not cleaning up subscriptions in tests | Test pollution, flaky tests | Use `done()` callback or `fakeAsync` |
| Forgetting `afterEach` for HttpMock | Undetected outstanding HTTP requests | Always call `httpMock.verify()` in `afterEach` |
| Component dependencies not mocked | Tests hit real services | Provide mocks for all service dependencies |
| Async operations not awaited | Test passes before async completes | Use `fakeAsync`/`tick`, `waitForAsync`, or `done()` |
