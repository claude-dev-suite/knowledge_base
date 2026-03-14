# Playwright Page Object Model

A comprehensive guide to implementing the Page Object Model (POM) pattern in Playwright
for maintainable, scalable, and readable end-to-end tests.

---

## Table of Contents

1. [Page Object Model Introduction](#1-page-object-model-introduction)
2. [Basic Page Object Structure](#2-basic-page-object-structure)
3. [Locators in Page Objects](#3-locators-in-page-objects)
4. [Actions in Page Objects](#4-actions-in-page-objects)
5. [Assertions in Page Objects](#5-assertions-in-page-objects)
6. [Component Page Objects](#6-component-page-objects)
7. [Page Object Inheritance](#7-page-object-inheritance)
8. [Page Object Factory Pattern](#8-page-object-factory-pattern)
9. [Fixtures with Page Objects](#9-fixtures-with-page-objects)
10. [Data-Driven Page Objects](#10-data-driven-page-objects)
11. [Navigation Methods](#11-navigation-methods)
12. [Form Page Objects](#12-form-page-objects)
13. [Table/List Page Objects](#13-tablelist-page-objects)
14. [Modal/Dialog Page Objects](#14-modaldialog-page-objects)
15. [Authentication Page Objects](#15-authentication-page-objects)
16. [API Integration in Page Objects](#16-api-integration-in-page-objects)
17. [Best Practices](#17-best-practices)

---

## 1. Page Object Model Introduction

### What is the Page Object Model?

The Page Object Model is a design pattern that creates an object repository for web UI
elements. It encapsulates page-specific elements and behaviors into dedicated classes,
separating test logic from page implementation details.

### Benefits

- **Maintainability**: Changes to UI only require updates in one place
- **Reusability**: Page objects can be shared across multiple tests
- **Readability**: Tests read like user stories
- **Encapsulation**: Implementation details hidden from tests
- **Reduced Duplication**: Common operations defined once

### Core Principles

```typescript
// ❌ Without Page Objects - fragile, repetitive
test('login test', async ({ page }) => {
  await page.goto('/login');
  await page.locator('#email').fill('user@example.com');
  await page.locator('#password').fill('password123');
  await page.locator('button[type="submit"]').click();
  await expect(page.locator('.welcome-message')).toBeVisible();
});

// ✅ With Page Objects - clean, maintainable
test('login test', async ({ loginPage, dashboardPage }) => {
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');
  await dashboardPage.expectWelcomeMessage();
});
```

---

## 2. Basic Page Object Structure

### Minimal Page Object

```typescript
// pages/login.page.ts
import { type Page, type Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

### Using the Page Object

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';

test('successful login', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');

  await expect(page).toHaveURL('/dashboard');
});

test('invalid credentials', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('wrong@email.com', 'wrongpassword');

  await expect(loginPage.errorMessage).toContainText('Invalid credentials');
});
```

### Complete Page Object Template

```typescript
// pages/base.page.ts
import { type Page, type Locator, expect } from '@playwright/test';

export abstract class BasePage {
  readonly page: Page;
  abstract readonly url: string;

  constructor(page: Page) {
    this.page = page;
  }

  async goto() {
    await this.page.goto(this.url);
  }

  async getTitle() {
    return await this.page.title();
  }

  async waitForPageLoad() {
    await this.page.waitForLoadState('networkidle');
  }
}
```

---

## 3. Locators in Page Objects

### Defining Locators

```typescript
export class ProductPage {
  readonly page: Page;

  // Role-based (recommended)
  readonly addToCartButton: Locator;
  readonly productTitle: Locator;
  readonly priceDisplay: Locator;

  // Text-based
  readonly descriptionText: Locator;

  // Test ID-based
  readonly quantityInput: Locator;

  // Composite locators
  readonly reviewSection: Locator;
  readonly reviewItems: Locator;

  constructor(page: Page) {
    this.page = page;

    // Prefer semantic locators
    this.addToCartButton = page.getByRole('button', { name: 'Add to Cart' });
    this.productTitle = page.getByRole('heading', { level: 1 });
    this.priceDisplay = page.getByTestId('product-price');

    // Text content
    this.descriptionText = page.getByText('Product Description');

    // Form inputs
    this.quantityInput = page.getByLabel('Quantity');

    // Composite
    this.reviewSection = page.locator('[data-testid="reviews"]');
    this.reviewItems = this.reviewSection.getByRole('article');
  }
}
```

### Dynamic Locators

```typescript
export class ProductListPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Method returning dynamic locator
  getProductCard(productName: string): Locator {
    return this.page.getByRole('article').filter({
      has: this.page.getByRole('heading', { name: productName })
    });
  }

  getProductByIndex(index: number): Locator {
    return this.page.getByRole('article').nth(index);
  }

  getProductAddButton(productName: string): Locator {
    return this.getProductCard(productName)
      .getByRole('button', { name: 'Add to Cart' });
  }
}
```

### Lazy Locators with Getters

```typescript
export class SearchResultsPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Lazy evaluation using getters
  get resultCount(): Locator {
    return this.page.getByTestId('result-count');
  }

  get results(): Locator {
    return this.page.getByRole('listitem');
  }

  get noResultsMessage(): Locator {
    return this.page.getByText('No results found');
  }

  get sortDropdown(): Locator {
    return this.page.getByRole('combobox', { name: 'Sort by' });
  }
}
```

---

## 4. Actions in Page Objects

### Basic Actions

```typescript
export class CheckoutPage {
  readonly page: Page;
  private readonly shippingForm: Locator;

  constructor(page: Page) {
    this.page = page;
    this.shippingForm = page.locator('[data-testid="shipping-form"]');
  }

  async fillShippingAddress(address: ShippingAddress) {
    await this.shippingForm.getByLabel('First Name').fill(address.firstName);
    await this.shippingForm.getByLabel('Last Name').fill(address.lastName);
    await this.shippingForm.getByLabel('Address').fill(address.street);
    await this.shippingForm.getByLabel('City').fill(address.city);
    await this.shippingForm.getByLabel('ZIP Code').fill(address.zipCode);
    await this.shippingForm.getByLabel('Country').selectOption(address.country);
  }

  async selectShippingMethod(method: 'standard' | 'express' | 'overnight') {
    await this.page.getByRole('radio', { name: method }).check();
  }

  async proceedToPayment() {
    await this.page.getByRole('button', { name: 'Continue to Payment' }).click();
  }
}
```

### Chainable Actions

```typescript
export class CartPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Return this for chaining
  async updateQuantity(productName: string, quantity: number): Promise<this> {
    const productRow = this.page.getByRole('row').filter({
      has: this.page.getByText(productName)
    });
    await productRow.getByRole('spinbutton').fill(quantity.toString());
    return this;
  }

  async removeItem(productName: string): Promise<this> {
    const productRow = this.page.getByRole('row').filter({
      has: this.page.getByText(productName)
    });
    await productRow.getByRole('button', { name: 'Remove' }).click();
    return this;
  }

  async applyCoupon(code: string): Promise<this> {
    await this.page.getByLabel('Coupon Code').fill(code);
    await this.page.getByRole('button', { name: 'Apply' }).click();
    return this;
  }

  async proceedToCheckout(): Promise<CheckoutPage> {
    await this.page.getByRole('button', { name: 'Checkout' }).click();
    return new CheckoutPage(this.page);
  }
}

// Usage with chaining
await cartPage
  .updateQuantity('Product A', 2)
  .removeItem('Product B')
  .applyCoupon('SAVE10')
  .proceedToCheckout();
```

### Async Action Helpers

```typescript
export class EditorPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async typeWithDelay(text: string, delayMs = 50) {
    const editor = this.page.getByRole('textbox', { name: 'Editor' });
    await editor.pressSequentially(text, { delay: delayMs });
  }

  async uploadFile(filePath: string) {
    const fileChooserPromise = this.page.waitForEvent('filechooser');
    await this.page.getByRole('button', { name: 'Upload' }).click();
    const fileChooser = await fileChooserPromise;
    await fileChooser.setFiles(filePath);
  }

  async waitForAutoSave() {
    await this.page.getByText('Saved').waitFor({ state: 'visible' });
  }

  async saveAndWait() {
    const responsePromise = this.page.waitForResponse(
      response => response.url().includes('/api/save') && response.ok()
    );
    await this.page.getByRole('button', { name: 'Save' }).click();
    await responsePromise;
  }
}
```

---

## 5. Assertions in Page Objects

### Page-Level Assertions

```typescript
import { expect, type Page, type Locator } from '@playwright/test';

export class DashboardPage {
  readonly page: Page;
  readonly welcomeMessage: Locator;
  readonly userStats: Locator;

  constructor(page: Page) {
    this.page = page;
    this.welcomeMessage = page.getByRole('heading', { name: /Welcome/ });
    this.userStats = page.getByTestId('user-stats');
  }

  // Assertion methods
  async expectToBeVisible() {
    await expect(this.page).toHaveURL(/\/dashboard/);
    await expect(this.welcomeMessage).toBeVisible();
  }

  async expectWelcomeMessage(userName: string) {
    await expect(this.welcomeMessage).toContainText(`Welcome, ${userName}`);
  }

  async expectStatValue(statName: string, value: string | number) {
    const stat = this.userStats.getByRole('listitem').filter({
      hasText: statName
    });
    await expect(stat).toContainText(value.toString());
  }

  async expectNoErrorsDisplayed() {
    await expect(this.page.getByRole('alert')).not.toBeVisible();
  }
}
```

### Soft Assertions in Page Objects

```typescript
export class ProfilePage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async verifyProfileData(expectedData: UserProfile) {
    // Use soft assertions to check all fields
    await expect.soft(
      this.page.getByLabel('Name')
    ).toHaveValue(expectedData.name);

    await expect.soft(
      this.page.getByLabel('Email')
    ).toHaveValue(expectedData.email);

    await expect.soft(
      this.page.getByLabel('Phone')
    ).toHaveValue(expectedData.phone || '');

    await expect.soft(
      this.page.getByLabel('Bio')
    ).toHaveValue(expectedData.bio || '');
  }
}
```

---

## 6. Component Page Objects

### Reusable Components

```typescript
// components/header.component.ts
export class HeaderComponent {
  readonly container: Locator;
  readonly logo: Locator;
  readonly searchInput: Locator;
  readonly cartIcon: Locator;
  readonly userMenu: Locator;

  constructor(page: Page) {
    this.container = page.locator('header');
    this.logo = this.container.getByRole('link', { name: 'Home' });
    this.searchInput = this.container.getByRole('searchbox');
    this.cartIcon = this.container.getByRole('link', { name: /Cart/ });
    this.userMenu = this.container.getByRole('button', { name: 'User menu' });
  }

  async search(query: string) {
    await this.searchInput.fill(query);
    await this.searchInput.press('Enter');
  }

  async openCart() {
    await this.cartIcon.click();
  }

  async logout() {
    await this.userMenu.click();
    await this.container.getByRole('menuitem', { name: 'Logout' }).click();
  }
}

// components/footer.component.ts
export class FooterComponent {
  readonly container: Locator;

  constructor(page: Page) {
    this.container = page.locator('footer');
  }

  async clickLink(linkName: string) {
    await this.container.getByRole('link', { name: linkName }).click();
  }

  async subscribToNewsletter(email: string) {
    await this.container.getByLabel('Email').fill(email);
    await this.container.getByRole('button', { name: 'Subscribe' }).click();
  }
}
```

### Composing Page Objects

```typescript
// pages/home.page.ts
export class HomePage {
  readonly page: Page;
  readonly header: HeaderComponent;
  readonly footer: FooterComponent;
  readonly heroSection: Locator;
  readonly featuredProducts: Locator;

  constructor(page: Page) {
    this.page = page;
    this.header = new HeaderComponent(page);
    this.footer = new FooterComponent(page);
    this.heroSection = page.getByTestId('hero-section');
    this.featuredProducts = page.getByTestId('featured-products');
  }

  async goto() {
    await this.page.goto('/');
  }

  async searchForProduct(query: string) {
    await this.header.search(query);
  }
}
```

---

## 7. Page Object Inheritance

### Base Page Class

```typescript
// pages/base.page.ts
export abstract class BasePage {
  readonly page: Page;
  readonly header: HeaderComponent;
  readonly footer: FooterComponent;
  abstract readonly url: string;

  constructor(page: Page) {
    this.page = page;
    this.header = new HeaderComponent(page);
    this.footer = new FooterComponent(page);
  }

  async goto() {
    await this.page.goto(this.url);
    await this.waitForPageLoad();
  }

  async waitForPageLoad() {
    await this.page.waitForLoadState('domcontentloaded');
  }

  async getPageTitle(): Promise<string> {
    return await this.page.title();
  }

  async takeScreenshot(name: string) {
    await this.page.screenshot({ path: `screenshots/${name}.png` });
  }
}
```

### Extending Base Page

```typescript
// pages/products.page.ts
export class ProductsPage extends BasePage {
  readonly url = '/products';
  readonly productGrid: Locator;
  readonly filterSidebar: Locator;
  readonly sortDropdown: Locator;

  constructor(page: Page) {
    super(page);
    this.productGrid = page.getByTestId('product-grid');
    this.filterSidebar = page.getByRole('complementary');
    this.sortDropdown = page.getByRole('combobox', { name: 'Sort' });
  }

  async filterByCategory(category: string) {
    await this.filterSidebar
      .getByRole('checkbox', { name: category })
      .check();
  }

  async sortBy(option: string) {
    await this.sortDropdown.selectOption(option);
  }

  getProductCard(name: string): ProductCardComponent {
    const card = this.productGrid.locator('article').filter({
      has: this.page.getByRole('heading', { name })
    });
    return new ProductCardComponent(card);
  }
}
```

---

## 8. Page Object Factory Pattern

```typescript
// pages/page-factory.ts
export class PageFactory {
  private page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  get login(): LoginPage {
    return new LoginPage(this.page);
  }

  get dashboard(): DashboardPage {
    return new DashboardPage(this.page);
  }

  get products(): ProductsPage {
    return new ProductsPage(this.page);
  }

  get cart(): CartPage {
    return new CartPage(this.page);
  }

  get checkout(): CheckoutPage {
    return new CheckoutPage(this.page);
  }
}

// Usage in tests
test('complete purchase flow', async ({ page }) => {
  const pages = new PageFactory(page);

  await pages.login.goto();
  await pages.login.login('user@example.com', 'password');

  await pages.products.goto();
  await pages.products.getProductCard('Widget').addToCart();

  await pages.cart.proceedToCheckout();
  await pages.checkout.completeOrder();
});
```

---

## 9. Fixtures with Page Objects

### Custom Fixtures

```typescript
// fixtures/pages.fixture.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { DashboardPage } from '../pages/dashboard.page';
import { ProductsPage } from '../pages/products.page';

type PageFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  productsPage: ProductsPage;
};

export const test = base.extend<PageFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },

  productsPage: async ({ page }, use) => {
    await use(new ProductsPage(page));
  },
});

export { expect } from '@playwright/test';
```

### Using Fixtures in Tests

```typescript
// tests/shopping.spec.ts
import { test, expect } from '../fixtures/pages.fixture';

test('user can browse products', async ({ loginPage, dashboardPage, productsPage }) => {
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password');

  await dashboardPage.expectToBeVisible();
  await dashboardPage.header.clickLink('Products');

  await productsPage.filterByCategory('Electronics');
  await expect(productsPage.productGrid).toBeVisible();
});
```

### Authenticated Fixtures

```typescript
// fixtures/auth.fixture.ts
type AuthFixtures = {
  authenticatedPage: Page;
  userDashboard: DashboardPage;
};

export const test = base.extend<AuthFixtures>({
  authenticatedPage: async ({ page }, use) => {
    // Login before test
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login(
      process.env.TEST_USER_EMAIL!,
      process.env.TEST_USER_PASSWORD!
    );

    // Wait for dashboard to confirm login
    await page.waitForURL('/dashboard');

    await use(page);

    // Logout after test
    const header = new HeaderComponent(page);
    await header.logout();
  },

  userDashboard: async ({ authenticatedPage }, use) => {
    await use(new DashboardPage(authenticatedPage));
  },
});
```

---

## 10. Data-Driven Page Objects

### Test Data Types

```typescript
// types/test-data.ts
export interface UserCredentials {
  email: string;
  password: string;
}

export interface Product {
  name: string;
  price: number;
  quantity: number;
}

export interface ShippingAddress {
  firstName: string;
  lastName: string;
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

export interface OrderData {
  products: Product[];
  shipping: ShippingAddress;
  paymentMethod: 'credit_card' | 'paypal' | 'bank_transfer';
}
```

### Data-Driven Page Object

```typescript
// pages/checkout.page.ts
export class CheckoutPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async fillShippingForm(address: ShippingAddress) {
    const form = this.page.getByTestId('shipping-form');

    for (const [field, value] of Object.entries(address)) {
      const label = this.fieldToLabel(field);
      const input = form.getByLabel(label);

      if (field === 'country' || field === 'state') {
        await input.selectOption(value);
      } else {
        await input.fill(value);
      }
    }
  }

  private fieldToLabel(field: string): string {
    const labels: Record<string, string> = {
      firstName: 'First Name',
      lastName: 'Last Name',
      street: 'Street Address',
      city: 'City',
      state: 'State',
      zipCode: 'ZIP Code',
      country: 'Country',
    };
    return labels[field] || field;
  }

  async completeOrder(orderData: OrderData) {
    await this.fillShippingForm(orderData.shipping);
    await this.selectPaymentMethod(orderData.paymentMethod);
    await this.submitOrder();
  }
}
```

---

## 11. Navigation Methods

```typescript
export class NavigationHelper {
  readonly page: Page;
  readonly header: HeaderComponent;

  constructor(page: Page) {
    this.page = page;
    this.header = new HeaderComponent(page);
  }

  async goToHome(): Promise<HomePage> {
    await this.header.logo.click();
    return new HomePage(this.page);
  }

  async goToProducts(): Promise<ProductsPage> {
    await this.page.getByRole('link', { name: 'Products' }).click();
    return new ProductsPage(this.page);
  }

  async goToCart(): Promise<CartPage> {
    await this.header.cartIcon.click();
    return new CartPage(this.page);
  }

  async goBack(): Promise<void> {
    await this.page.goBack();
  }

  async waitForNavigation(urlPattern: RegExp): Promise<void> {
    await this.page.waitForURL(urlPattern);
  }
}
```

---

## 12. Form Page Objects

```typescript
export class RegistrationFormPage {
  readonly page: Page;
  private readonly form: Locator;

  constructor(page: Page) {
    this.page = page;
    this.form = page.getByRole('form', { name: 'Registration' });
  }

  // Field accessors
  get emailField() { return this.form.getByLabel('Email'); }
  get passwordField() { return this.form.getByLabel('Password'); }
  get confirmPasswordField() { return this.form.getByLabel('Confirm Password'); }
  get termsCheckbox() { return this.form.getByRole('checkbox', { name: /terms/i }); }
  get submitButton() { return this.form.getByRole('button', { name: 'Register' }); }

  async fillForm(data: RegistrationData) {
    await this.emailField.fill(data.email);
    await this.passwordField.fill(data.password);
    await this.confirmPasswordField.fill(data.confirmPassword);

    if (data.acceptTerms) {
      await this.termsCheckbox.check();
    }
  }

  async submit() {
    await this.submitButton.click();
  }

  async getFieldError(fieldName: string): Promise<string | null> {
    const error = this.form.locator(`[data-error-for="${fieldName}"]`);
    if (await error.isVisible()) {
      return await error.textContent();
    }
    return null;
  }

  async expectFieldError(fieldName: string, expectedError: string) {
    const error = this.form.locator(`[data-error-for="${fieldName}"]`);
    await expect(error).toContainText(expectedError);
  }
}
```

---

## 13. Table/List Page Objects

```typescript
export class DataTablePage {
  readonly page: Page;
  private readonly table: Locator;

  constructor(page: Page) {
    this.page = page;
    this.table = page.getByRole('table');
  }

  get headers(): Locator {
    return this.table.getByRole('columnheader');
  }

  get rows(): Locator {
    return this.table.getByRole('row').filter({ hasNot: this.page.locator('th') });
  }

  async getRowCount(): Promise<number> {
    return await this.rows.count();
  }

  getRowByIndex(index: number): Locator {
    return this.rows.nth(index);
  }

  getRowByText(text: string): Locator {
    return this.rows.filter({ hasText: text });
  }

  async getCellValue(rowIndex: number, columnIndex: number): Promise<string> {
    const cell = this.getRowByIndex(rowIndex).getByRole('cell').nth(columnIndex);
    return (await cell.textContent()) || '';
  }

  async sortByColumn(columnName: string) {
    await this.table.getByRole('columnheader', { name: columnName }).click();
  }

  async selectRow(rowIndex: number) {
    await this.getRowByIndex(rowIndex).getByRole('checkbox').check();
  }

  async deleteSelectedRows() {
    await this.page.getByRole('button', { name: 'Delete Selected' }).click();
    await this.page.getByRole('button', { name: 'Confirm' }).click();
  }
}
```

---

## 14. Modal/Dialog Page Objects

```typescript
export class ConfirmationModal {
  readonly modal: Locator;

  constructor(page: Page) {
    this.modal = page.getByRole('dialog');
  }

  get title(): Locator {
    return this.modal.getByRole('heading');
  }

  get message(): Locator {
    return this.modal.locator('[data-testid="modal-message"]');
  }

  get confirmButton(): Locator {
    return this.modal.getByRole('button', { name: /confirm|yes|ok/i });
  }

  get cancelButton(): Locator {
    return this.modal.getByRole('button', { name: /cancel|no/i });
  }

  async waitForOpen() {
    await this.modal.waitFor({ state: 'visible' });
  }

  async confirm() {
    await this.confirmButton.click();
    await this.modal.waitFor({ state: 'hidden' });
  }

  async cancel() {
    await this.cancelButton.click();
    await this.modal.waitFor({ state: 'hidden' });
  }

  async close() {
    await this.modal.getByRole('button', { name: 'Close' }).click();
    await this.modal.waitFor({ state: 'hidden' });
  }
}
```

---

## 15. Authentication Page Objects

```typescript
export class AuthPages {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async loginViaUI(email: string, password: string) {
    await this.page.goto('/login');
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Sign in' }).click();
    await this.page.waitForURL('/dashboard');
  }

  async loginViaAPI(email: string, password: string) {
    const response = await this.page.request.post('/api/auth/login', {
      data: { email, password }
    });

    const { token } = await response.json();

    // Set token in localStorage or cookies
    await this.page.evaluate((t) => {
      localStorage.setItem('authToken', t);
    }, token);
  }

  async loginWithStoredState(statePath: string) {
    // Use stored authentication state
    await this.page.context().addCookies(
      JSON.parse(require('fs').readFileSync(statePath, 'utf8')).cookies
    );
  }

  async logout() {
    await this.page.getByRole('button', { name: 'User menu' }).click();
    await this.page.getByRole('menuitem', { name: 'Logout' }).click();
    await this.page.waitForURL('/login');
  }

  async saveAuthState(path: string) {
    await this.page.context().storageState({ path });
  }
}
```

---

## 16. API Integration in Page Objects

```typescript
export class ProductsPageWithAPI {
  readonly page: Page;
  readonly request: APIRequestContext;

  constructor(page: Page) {
    this.page = page;
    this.request = page.request;
  }

  // Create test data via API
  async createProduct(product: ProductData): Promise<string> {
    const response = await this.request.post('/api/products', {
      data: product
    });
    const { id } = await response.json();
    return id;
  }

  // Clean up via API
  async deleteProduct(productId: string) {
    await this.request.delete(`/api/products/${productId}`);
  }

  // Setup test state
  async setupTestProducts(products: ProductData[]): Promise<string[]> {
    const ids: string[] = [];
    for (const product of products) {
      const id = await this.createProduct(product);
      ids.push(id);
    }
    return ids;
  }

  // Verify via API
  async verifyProductExists(productId: string): Promise<boolean> {
    const response = await this.request.get(`/api/products/${productId}`);
    return response.ok();
  }
}
```

---

## 17. Best Practices

### 1. Single Responsibility

```typescript
// ✅ Good: Focused page object
export class LoginPage {
  // Only login-related functionality
}

// ❌ Bad: Page object doing too much
export class LoginPage {
  // Login + Registration + Password Reset + OAuth
}
```

### 2. Meaningful Method Names

```typescript
// ✅ Good: Describes the action
await cartPage.addProductToCart('Widget');
await checkoutPage.completeOrder();

// ❌ Bad: Generic names
await cartPage.clickButton();
await checkoutPage.submit();
```

### 3. Encapsulate Locators

```typescript
// ✅ Good: Locators are private
export class LoginPage {
  private readonly emailInput: Locator;

  async enterEmail(email: string) {
    await this.emailInput.fill(email);
  }
}

// ❌ Bad: Exposing locators directly
export class LoginPage {
  public emailInput: Locator; // Tests can access directly
}
```

### 4. Return Page Objects for Navigation

```typescript
// ✅ Good: Return type indicates page transition
async login(): Promise<DashboardPage> {
  await this.submitButton.click();
  return new DashboardPage(this.page);
}

// Usage
const dashboard = await loginPage.login();
await dashboard.expectWelcomeMessage();
```

### 5. Keep Tests Readable

```typescript
// ✅ Good: Test reads like a user story
test('user can complete purchase', async ({ loginPage, productsPage, cartPage, checkoutPage }) => {
  await loginPage.loginAs('customer@example.com', 'password');
  await productsPage.addToCart('Premium Widget');
  await cartPage.proceedToCheckout();
  await checkoutPage.fillShippingDetails(testAddress);
  await checkoutPage.submitOrder();
  await checkoutPage.expectOrderConfirmation();
});
```

### 6. Use Fixtures for Setup

```typescript
// ✅ Good: Use fixtures for common setup
export const test = base.extend({
  authenticatedPage: async ({ page }, use) => {
    await new LoginPage(page).login(testUser);
    await use(page);
  },
});
```

### 7. Handle Waiting Properly

```typescript
// ✅ Good: Let Playwright handle waiting
async addToCart() {
  await this.addButton.click();
  await this.successMessage.waitFor();
}

// ❌ Bad: Arbitrary waits
async addToCart() {
  await this.addButton.click();
  await this.page.waitForTimeout(2000);
}
```

---

## Quick Reference

| Pattern | Use Case |
|---------|----------|
| Base Page | Shared functionality across pages |
| Component PO | Reusable UI components |
| Fixtures | Test setup and teardown |
| Factory | Creating page objects on demand |
| Data-Driven | Parameterized test data |

---

## Additional Resources

- [Playwright Page Object Model](https://playwright.dev/docs/pom)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [Playwright Fixtures](https://playwright.dev/docs/test-fixtures)
