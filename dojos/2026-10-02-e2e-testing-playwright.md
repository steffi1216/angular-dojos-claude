# Angular Dojo: E2E Testing mit Playwright
**Datum:** 2026-10-02
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular-Applikationen mit Playwright End-to-End testest – inklusive Page Object Model (POM), Fixtures und dem Testen von Angular-spezifischen Mustern wie Navigation, Formularen und HTTP-Mocking.

## Hintergrund & Theorie

Playwright ist ein modernes E2E-Testing-Framework von Microsoft, das sich hervorragend mit Angular verträgt. Im Gegensatz zu Unit- und Komponenten-Tests prüfen E2E-Tests die gesamte Applikation aus der Nutzerperspektive – inklusive Routing, HTTP-Kommunikation und Browser-APIs.

**Wichtige Konzepte:**

- **Page Object Model (POM):** Kapselt Selektor-Logik in wiederverwendbare Klassen, damit Änderungen an der UI nur an einer Stelle angepasst werden müssen.
- **Fixtures:** Playwright-Fixtures erlauben es, POM-Instanzen automatisch in Tests zu injizieren – analog zu Angulars Dependency Injection.
- **`page.route()`:** Interceptet HTTP-Anfragen im Browser und ermöglicht das Mocken von API-Antworten, ohne den Angular-Server zu verändern.
- **`waitForResponse` / `waitForRequest`:** Synchronisiert Tests mit asynchronen Netzwerkanfragen – entscheidend für SPA-Testing.
- **Auto-waiting:** Playwright wartet automatisch, bis Elemente sichtbar und interaktiv sind – kein manuelles `sleep` nötig.

Angular CLI ≥ 17 erzeugt mit `ng add @playwright/test` oder `ng e2e` eine vorkonfigurierte Playwright-Umgebung direkt im Projekt.

## Aufgabe

Du hast eine Angular-Applikation mit einer Produktliste, die Daten von `/api/products` lädt, und einer Detailansicht mit einem Formular zum Bearbeiten. Schreibe E2E-Tests mit dem Page Object Model, die:

1. Die Produktliste mit einem gemockten HTTP-Response testen
2. Navigation zur Detailansicht und zurück prüfen
3. Das Edit-Formular befüllen und absenden – dabei den PUT-Request verifizieren

### Schritte

1. **Playwright setup** – Installiere Playwright und erstelle die Konfiguration
2. **Page Objects** – Erstelle `ProductListPage` und `ProductDetailPage` als Klassen
3. **Fixture erstellen** – Definiere ein Playwright-Fixture, das die Page Objects bereitstellt
4. **HTTP-Mocking** – Intercepte den `/api/products`-Endpunkt mit `page.route()`
5. **Tests schreiben** – Schreibe drei Tests mit `test.use(fixture)` und verifiziere Navigations- und Formularverhalten
6. **Request-Assertion** – Nutze `page.waitForRequest()`, um den gesendeten PUT-Body zu prüfen

## Hints

<details>
<summary>Hint 1 – Grundstruktur Page Object</summary>

```typescript
// e2e/pages/product-list.page.ts
import { Page, Locator } from '@playwright/test';

export class ProductListPage {
  readonly productItems: Locator;
  readonly searchInput: Locator;

  constructor(private page: Page) {
    this.productItems = page.locator('[data-testid="product-item"]');
    this.searchInput = page.getByRole('searchbox');
  }

  async goto() {
    await this.page.goto('/products');
  }

  async clickProduct(index: number) {
    await this.productItems.nth(index).click();
  }

  getProductName(index: number): Locator {
    return this.productItems.nth(index).locator('[data-testid="product-name"]');
  }
}
```

</details>

<details>
<summary>Hint 2 – HTTP-Mocking mit page.route()</summary>

```typescript
// Im Test oder beforeEach:
await page.route('/api/products', async route => {
  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify([
      { id: 1, name: 'Widget A', price: 9.99 },
      { id: 2, name: 'Widget B', price: 19.99 },
    ]),
  });
});

// PUT-Request abfangen und Body prüfen:
const requestPromise = page.waitForRequest(
  req => req.url().includes('/api/products/1') && req.method() === 'PUT'
);
await submitButton.click();
const request = await requestPromise;
const body = request.postDataJSON();
expect(body.name).toBe('Updated Widget A');
```

</details>

<details>
<summary>Hint 3 – Playwright Fixture für Page Objects</summary>

```typescript
// e2e/fixtures.ts
import { test as base } from '@playwright/test';
import { ProductListPage } from './pages/product-list.page';
import { ProductDetailPage } from './pages/product-detail.page';

type Fixtures = {
  productListPage: ProductListPage;
  productDetailPage: ProductDetailPage;
};

export const test = base.extend<Fixtures>({
  productListPage: async ({ page }, use) => {
    await use(new ProductListPage(page));
  },
  productDetailPage: async ({ page }, use) => {
    await use(new ProductDetailPage(page));
  },
});

export { expect } from '@playwright/test';
```

</details>

## Beispiellösung

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:4200',
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
  },
  webServer: {
    command: 'ng serve',
    url: 'http://localhost:4200',
    reuseExistingServer: !process.env['CI'],
  },
});

// e2e/pages/product-list.page.ts
import { Page, Locator } from '@playwright/test';

export class ProductListPage {
  readonly productItems: Locator;

  constructor(private page: Page) {
    this.productItems = page.locator('[data-testid="product-item"]');
  }

  async goto() {
    await this.page.goto('/products');
  }

  async clickProduct(name: string) {
    await this.page.getByText(name).click();
  }
}

// e2e/pages/product-detail.page.ts
import { Page, Locator } from '@playwright/test';

export class ProductDetailPage {
  readonly nameInput: Locator;
  readonly priceInput: Locator;
  readonly submitButton: Locator;
  readonly backButton: Locator;

  constructor(private page: Page) {
    this.nameInput = page.getByLabel('Name');
    this.priceInput = page.getByLabel('Preis');
    this.submitButton = page.getByRole('button', { name: 'Speichern' });
    this.backButton = page.getByRole('link', { name: 'Zurück' });
  }
}

// e2e/fixtures.ts
import { test as base } from '@playwright/test';
import { ProductListPage } from './pages/product-list.page';
import { ProductDetailPage } from './pages/product-detail.page';

export const test = base.extend<{
  productListPage: ProductListPage;
  productDetailPage: ProductDetailPage;
}>({
  productListPage: async ({ page }, use) => use(new ProductListPage(page)),
  productDetailPage: async ({ page }, use) => use(new ProductDetailPage(page)),
});

export { expect } from '@playwright/test';

// e2e/products.spec.ts
import { test, expect } from './fixtures';

const MOCK_PRODUCTS = [
  { id: 1, name: 'Widget A', price: 9.99 },
  { id: 2, name: 'Widget B', price: 19.99 },
];

test.beforeEach(async ({ page }) => {
  await page.route('/api/products', route =>
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify(MOCK_PRODUCTS),
    })
  );
  await page.route('/api/products/*', route =>
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify(MOCK_PRODUCTS[0]),
    })
  );
});

test('zeigt gemockte Produktliste an', async ({ productListPage }) => {
  await productListPage.goto();
  await expect(productListPage.productItems).toHaveCount(2);
  await expect(productListPage.productItems.first()).toContainText('Widget A');
});

test('navigiert zur Detailansicht und zurück', async ({
  page,
  productListPage,
  productDetailPage,
}) => {
  await productListPage.goto();
  await productListPage.clickProduct('Widget A');

  await expect(page).toHaveURL(/\/products\/1/);
  await expect(productDetailPage.nameInput).toHaveValue('Widget A');

  await productDetailPage.backButton.click();
  await expect(page).toHaveURL('/products');
});

test('bearbeitet Produkt und sendet PUT-Request', async ({
  page,
  productListPage,
  productDetailPage,
}) => {
  // PUT-Response mocken
  await page.route('**/api/products/1', route => {
    if (route.request().method() === 'PUT') {
      route.fulfill({ status: 200, body: '{}' });
    } else {
      route.continue();
    }
  });

  await productListPage.goto();
  await productListPage.clickProduct('Widget A');

  // Request abfangen BEVOR Submit
  const putRequest = page.waitForRequest(
    req => req.url().includes('/api/products/1') && req.method() === 'PUT'
  );

  await productDetailPage.nameInput.fill('Widget A (aktualisiert)');
  await productDetailPage.priceInput.fill('12.99');
  await productDetailPage.submitButton.click();

  const request = await putRequest;
  const body = request.postDataJSON();

  expect(body.name).toBe('Widget A (aktualisiert)');
  expect(body.price).toBe(12.99);
});
```

## Weiterführendes

- **Angular Component Harnesses + Playwright**: Mit `@angular/cdk/testing` lassen sich Playwright-Tests auf CDK-Harnesses aufbauen – kombiniere `HarnessLoader` mit dem `PlaywrightHarnessEnvironment`-Adapter für typsichere E2E-Tests.
- **Visual Testing**: `await expect(page).toHaveScreenshot()` erstellt Pixel-Snapshots – ideal für Regressions-Tests bei UI-Änderungen.
- **Playwright Codegen**: `npx playwright codegen http://localhost:4200` zeichnet Interaktionen auf und generiert automatisch Testcode als Startpunkt.
- Offizielle Doku: [playwright.dev/docs/test-fixtures](https://playwright.dev/docs/test-fixtures)
