# Angular Dojo: HTTP-Testing mit HttpTestingController
**Datum:** 2026-08-08
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular-Services und -Komponenten, die `HttpClient` verwenden, mit dem `HttpTestingController` isoliert und deterministisch testest – inklusive Fehlerszenarien und Signal-basiertem State.

## Hintergrund & Theorie
Angular stellt mit `provideHttpClientTesting()` eine Test-Infrastruktur bereit, die echte HTTP-Aufrufe durch einen synchron steuerbaren Mock-Mechanismus ersetzt. Das Herzstück ist der `HttpTestingController`: Er fängt ausgehende Requests ab, erlaubt dir deren Parameter zu inspizieren und gibt dann eine kontrollierte Response (oder einen Fehler) zurück.

Seit Angular 15+ wird `HttpClient` über `provideHttpClient()` konfiguriert. Für Tests kombinierst du das mit `provideHttpClientTesting()`. In der Testdatei injizierst du `HttpTestingController` und rufst nach jedem Test `verify()` auf – das stellt sicher, dass keine unerwarteten Requests offen geblieben sind.

Wichtige Methoden des Controllers:
- `expectOne(url | MatchFn)` – erwartet genau einen passenden Request
- `expectNone(url)` – stellt sicher, dass kein Request abgeschickt wurde
- `match(url | MatchFn)` – gibt alle passenden Requests zurück
- `req.flush(body, options?)` – simuliert eine erfolgreiche Response
- `req.error(ErrorEvent, options?)` – simuliert einen Netzwerkfehler
- `controller.verify()` – prüft auf offene, unbehandelte Requests

Durch diesen Ansatz sind Tests vollständig synchron und deterministisch – kein echtes Netzwerk, keine Timing-Probleme.

## Aufgabe
Erstelle einen `ProductService`, der Produkte von einer REST-API lädt und in einem Signal speichert. Schreibe anschließend vollständige Unit-Tests mit `HttpTestingController`, die Erfolgs- und Fehlerszenarien abdecken.

### Schritte

1. **Service erstellen:** Implementiere einen `ProductService` mit einer `loadProducts()`-Methode, die `GET /api/products` aufruft und das Ergebnis in einem `signal<Product[]>` hält. Füge ein `error`-Signal für Fehlermeldungen hinzu.

2. **TestBed konfigurieren:** Richte in der Spec-Datei `TestBed` mit `provideHttpClient()` und `provideHttpClientTesting()` ein. Injiziere den Service und den `HttpTestingController`.

3. **Erfolgsfall testen:** Rufe `loadProducts()` auf, erwarte den Request mit `expectOne()`, simuliere eine Response mit `flush()` und prüfe, dass das `products`-Signal die korrekten Daten enthält.

4. **Fehlerfall testen:** Simuliere einen HTTP-500-Fehler mit `req.flush('Server error', { status: 500, statusText: 'Internal Server Error' })` und verifiziere, dass das `error`-Signal gesetzt wird.

5. **Request-Parameter prüfen:** Teste eine gefilterte Suche `GET /api/products?category=electronics` und verifiziere Header und Query-Params des abgefangenen Requests.

6. **`verify()` in `afterEach`:** Stelle sicher, dass nach jedem Test keine offenen Requests übrig bleiben.

## Hints

<details>
<summary>Hint 1 – Service-Grundstruktur</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);

  products = signal<Product[]>([]);
  error = signal<string | null>(null);

  loadProducts(category?: string): void {
    const params = category ? { category } : {};
    this.http.get<Product[]>('/api/products', { params }).subscribe({
      next: (data) => {
        this.products.set(data);
        this.error.set(null);
      },
      error: (err: HttpErrorResponse) => {
        this.error.set(err.message);
      },
    });
  }
}
```
</details>

<details>
<summary>Hint 2 – TestBed-Setup und flush</summary>

```typescript
beforeEach(() => {
  TestBed.configureTestingModule({
    providers: [
      provideHttpClient(),
      provideHttpClientTesting(),
    ],
  });
  service = TestBed.inject(ProductService);
  controller = TestBed.inject(HttpTestingController);
});

afterEach(() => controller.verify());

it('loads products successfully', () => {
  const mockProducts: Product[] = [{ id: 1, name: 'Laptop' }];
  service.loadProducts();

  const req = controller.expectOne('/api/products');
  expect(req.request.method).toBe('GET');
  req.flush(mockProducts);

  expect(service.products()).toEqual(mockProducts);
  expect(service.error()).toBeNull();
});
```
</details>

<details>
<summary>Hint 3 – Fehlerfall und Query-Params</summary>

```typescript
it('handles HTTP 500 error', () => {
  service.loadProducts();
  const req = controller.expectOne('/api/products');
  req.flush('Server error', { status: 500, statusText: 'Internal Server Error' });

  expect(service.products()).toEqual([]);
  expect(service.error()).toBeTruthy();
});

it('sends category query param', () => {
  service.loadProducts('electronics');
  const req = controller.expectOne(r =>
    r.url === '/api/products' && r.params.get('category') === 'electronics'
  );
  req.flush([]);
});
```
</details>

## Beispiellösung

```typescript
// product.service.ts
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';

export interface Product {
  id: number;
  name: string;
  category: string;
  price: number;
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);

  products = signal<Product[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);

  loadProducts(category?: string): void {
    this.loading.set(true);
    this.error.set(null);

    const params: Record<string, string> = {};
    if (category) params['category'] = category;

    this.http.get<Product[]>('/api/products', { params }).subscribe({
      next: (data) => {
        this.products.set(data);
        this.loading.set(false);
      },
      error: (err: HttpErrorResponse) => {
        this.error.set(`Fehler ${err.status}: ${err.statusText}`);
        this.loading.set(false);
      },
    });
  }
}
```

```typescript
// product.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting } from '@angular/common/http/testing';
import { HttpTestingController } from '@angular/common/http/testing';
import { Product, ProductService } from './product.service';

describe('ProductService', () => {
  let service: ProductService;
  let controller: HttpTestingController;

  const mockProducts: Product[] = [
    { id: 1, name: 'Laptop',    category: 'electronics', price: 999 },
    { id: 2, name: 'Headphones', category: 'electronics', price: 199 },
  ];

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });
    service    = TestBed.inject(ProductService);
    controller = TestBed.inject(HttpTestingController);
  });

  afterEach(() => controller.verify());

  it('setzt loading=true während des Requests', () => {
    service.loadProducts();
    expect(service.loading()).toBeTrue();
    controller.expectOne('/api/products').flush([]);
    expect(service.loading()).toBeFalse();
  });

  it('befüllt das products-Signal bei Erfolg', () => {
    service.loadProducts();
    controller.expectOne('/api/products').flush(mockProducts);
    expect(service.products()).toEqual(mockProducts);
    expect(service.error()).toBeNull();
  });

  it('setzt das error-Signal bei HTTP-500', () => {
    service.loadProducts();
    controller.expectOne('/api/products').flush(
      'Server Error',
      { status: 500, statusText: 'Internal Server Error' }
    );
    expect(service.error()).toContain('500');
    expect(service.products()).toEqual([]);
  });

  it('sendet den category-Query-Parameter korrekt', () => {
    service.loadProducts('electronics');

    const req = controller.expectOne(r =>
      r.url === '/api/products' && r.params.get('category') === 'electronics'
    );
    expect(req.request.method).toBe('GET');
    req.flush(mockProducts);

    expect(service.products().length).toBe(2);
  });

  it('sendet keinen Request, wenn loadProducts nicht aufgerufen wurde', () => {
    controller.expectNone('/api/products');
  });

  it('verarbeitet eine leere Response korrekt', () => {
    service.loadProducts();
    controller.expectOne('/api/products').flush([]);
    expect(service.products()).toEqual([]);
    expect(service.error()).toBeNull();
  });
});
```

## Weiterführendes
- **Interceptor-Tests:** Kombiniere `HttpTestingController` mit `withInterceptors([authInterceptor])`, um zu prüfen, ob dein Interceptor den `Authorization`-Header korrekt setzt – `req.request.headers.get('Authorization')` steht dir im abgefangenen Request zur Verfügung.
- **Offizielle Docs:** [Angular – HTTP client testing](https://angular.dev/guide/http/testing)
- **Marble Testing für RxJS** (als Ergänzung): Wenn du komplexe RxJS-Ketten wie `switchMap` testen willst, schau dir `TestScheduler` aus `rxjs/testing` an.
