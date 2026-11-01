# Angular Dojo: `rxResource()` – Reaktive Datenladen mit Observable-Loader
**Datum:** 2026-11-01
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du `rxResource()` aus `@angular/core/rxjs-interop` einsetzt, um RxJS-Observables nahtlos in das Signal-basierte Resource-API zu integrieren – inklusive Request-Parameter, Ladezeichen und Fehlerbehandlung.

## Hintergrund & Theorie

Angular 19 führte neben `resource()` (Promise-basiert) und `httpResource()` (HttpClient-Integration) auch **`rxResource()`** ein – eine Observable-basierte Variante des Resource APIs.

`rxResource()` kombiniert das Beste aus beiden Welten:
- Die **deklarative Reaktivität** von Signals (automatische Neuladung bei Parameteränderung)
- Die **Mächtigkeit von RxJS** als Loader (switchMap, retry, delay, etc.)

Sobald sich der Request-Signal ändert, wird das vorherige Observable automatisch abgebrochen (`unsubscribe`) und ein neues gestartet – analog zu `switchMap`. Das Ergebnis ist ein `ResourceRef`-Objekt mit denselben Status-Signals wie `resource()`:

```typescript
resource.value()    // T | undefined – der aktuelle Wert
resource.status()   // ResourceStatus – idle, loading, resolved, error, ...
resource.isLoading() // boolean
resource.error()    // unknown | undefined
resource.reload()   // Manuelles Neuladen
```

Der entscheidende Unterschied zu `resource()`: Der `loader` gibt ein **Observable** zurück (nicht ein Promise) – du kannst also `pipe()`, `retry()`, `delay()` und alle RxJS-Operatoren nutzen.

## Aufgabe

Implementiere eine `ProductSearchComponent`, die:

1. Eine Suche (Signal) als Input bekommt
2. Mit `rxResource()` eine Produkt-API abfragt (Debouncing via RxJS)
3. Ladezeichen und Fehlerzustände im Template anzeigt
4. Einen „Neu laden"-Button anbietet

### Schritte

1. Erstelle einen `ProductService` mit einer Methode `search(query: string): Observable<Product[]>`, die einen HTTP-Request simuliert (nutze `of([...]).pipe(delay(500))` oder echtes `HttpClient`)

2. Erstelle `ProductSearchComponent` als Standalone Component mit einem Input-Signal `searchQuery = input<string>('')`

3. Definiere eine `rxResource()` mit:
   - `request: () => this.searchQuery()` – reagiert automatisch auf Änderungen
   - `loader: ({ request }) => this.productService.search(request).pipe(retry(2))`

4. Zeige im Template:
   - Ladeindikator wenn `products.isLoading()`
   - Fehlermeldung wenn `products.error()` gesetzt ist
   - Die Produktliste aus `products.value()`
   - Einen Button, der `products.reload()` aufruft

5. Teste: Was passiert, wenn du `searchQuery` schnell mehrfach änderst – werden parallele Requests abgebrochen?

## Hints

<details>
<summary>Hint 1 – Grundstruktur von rxResource()</summary>

```typescript
import { rxResource } from '@angular/core/rxjs-interop';
import { Component, input } from '@angular/core';
import { inject } from '@angular/core';

@Component({ ... })
export class ProductSearchComponent {
  private productService = inject(ProductService);
  searchQuery = input<string>('');

  products = rxResource({
    request: () => this.searchQuery(),
    loader: ({ request: query }) =>
      this.productService.search(query).pipe(
        retry(2)
      )
  });
}
```

`request` ist eine Computed-Funktion – sie wird im reaktiven Kontext ausgeführt und triggert ein Neuladen, sobald sich ihr Rückgabewert ändert.

</details>

<details>
<summary>Hint 2 – Template mit Status-Handling</summary>

```html
@if (products.isLoading()) {
  <p>Lädt...</p>
}

@if (products.error()) {
  <p class="error">Fehler: {{ products.error() }}</p>
}

@if (products.value(); as list) {
  @for (product of list; track product.id) {
    <div>{{ product.name }} – {{ product.price | currency }}</div>
  }
}

<button (click)="products.reload()">Neu laden</button>
```

Beachte: `products.value()` liefert `undefined` während des ersten Ladens – prüfe daher immer auf Existenz des Wertes.

</details>

<details>
<summary>Hint 3 – Leerer Request überspringen</summary>

Wenn `searchQuery()` leer ist, möchtest du keinen API-Call auslösen. Gib dafür `undefined` aus dem `request`-Callback zurück – `rxResource()` überspringt dann den Loader:

```typescript
products = rxResource({
  request: () => this.searchQuery() || undefined,
  loader: ({ request: query }) =>
    this.productService.search(query)
});
```

Der Status bleibt dann `idle` statt `loading`.

</details>

## Beispiellösung

```typescript
// product.service.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay, map } from 'rxjs/operators';

export interface Product {
  id: number;
  name: string;
  price: number;
}

const MOCK_PRODUCTS: Product[] = [
  { id: 1, name: 'Angular Buch', price: 39.99 },
  { id: 2, name: 'RxJS Kurs', price: 79.0 },
  { id: 3, name: 'Signal Workshop', price: 129.0 },
];

@Injectable({ providedIn: 'root' })
export class ProductService {
  search(query: string): Observable<Product[]> {
    return of(MOCK_PRODUCTS).pipe(
      delay(600),
      map(products =>
        products.filter(p =>
          p.name.toLowerCase().includes(query.toLowerCase())
        )
      )
    );
  }
}

// product-search.component.ts
import { Component, input, inject } from '@angular/core';
import { rxResource } from '@angular/core/rxjs-interop';
import { retry } from 'rxjs/operators';
import { CurrencyPipe } from '@angular/common';
import { ProductService } from './product.service';

@Component({
  selector: 'app-product-search',
  standalone: true,
  imports: [CurrencyPipe],
  template: `
    <div class="search-results">
      @if (products.isLoading()) {
        <p class="loading">⏳ Suche läuft...</p>
      }

      @if (products.error()) {
        <p class="error">
          Fehler beim Laden. 
          <button (click)="products.reload()">Erneut versuchen</button>
        </p>
      }

      @if (products.value(); as list) {
        @if (list.length === 0) {
          <p>Keine Produkte gefunden.</p>
        } @else {
          <ul>
            @for (product of list; track product.id) {
              <li>
                <strong>{{ product.name }}</strong>
                – {{ product.price | currency:'EUR' }}
              </li>
            }
          </ul>
        }
      }

      @if (!products.isLoading() && products.status() === 'resolved') {
        <button (click)="products.reload()">🔄 Neu laden</button>
      }
    </div>
  `
})
export class ProductSearchComponent {
  private productService = inject(ProductService);

  // Input von der Parent-Komponente (z.B. ein Suchfeld)
  searchQuery = input<string>('');

  products = rxResource({
    // Gibt undefined zurück, wenn Suche leer → kein API-Call
    request: () => this.searchQuery().trim() || undefined,
    loader: ({ request: query }) =>
      this.productService.search(query).pipe(
        retry({ count: 2, delay: 1000 })
      )
  });
}

// parent.component.ts – Verwendung
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ProductSearchComponent],
  template: `
    <input
      type="search"
      placeholder="Produkt suchen..."
      [value]="query()"
      (input)="query.set($event.target.value)"
    />
    <app-product-search [searchQuery]="query()" />
  `
})
export class AppComponent {
  query = signal('');
}
```

**Schlüsselpunkte:**
- `rxResource()` bricht automatisch das vorherige Observable ab (wie `switchMap`), wenn `request()` sich ändert
- `retry()` aus RxJS funktioniert direkt im Loader – kein manueller Retry-Code nötig
- `undefined` als Request-Wert hält den Status auf `idle` ohne HTTP-Call
- `products.reload()` forciert ein Neuladen mit demselben Request

## Weiterführendes

- Kombiniere `rxResource()` mit `debounceTime()` im Loader oder nutze ein debounced Signal im `request`-Callback für Echtzeit-Suche ohne übermäßige Requests
- Für HTTP-spezifische Ressourcen: vergleiche `rxResource()` mit `httpResource()` – letzteres integriert direkt den `HttpClient` inklusive TypeScript-Typsicherheit
- Mit `resource()` (Promise-basiert) und `rxResource()` (Observable-basiert) hast du nun beide Welten: wähle je nach vorhandener API-Schicht
- Dokumentation: [angular.dev/api/core/rxjs-interop/rxResource](https://angular.dev/api/core/rxjs-interop/rxResource)
