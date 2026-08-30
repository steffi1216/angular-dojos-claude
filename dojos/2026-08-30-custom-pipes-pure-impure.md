# Angular Dojo: Custom Pipes – Pure, Impure und Async
**Datum:** 2026-08-30
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst den Unterschied zwischen Pure und Impure Pipes, weißt wann du welche einsetzt, und kannst typsichere, performante Custom Pipes mit optionalen Parametern erstellen.

## Hintergrund & Theorie

Angular-Pipes transformieren Daten im Template, ohne die Quelldaten zu mutieren. Der entscheidende Schalter ist `pure`:

**Pure Pipes** (`pure: true`, der Default) werden von Angular **memoized**: Angular ruft `transform()` nur dann erneut auf, wenn sich der primitive Eingabewert oder die Referenz des Objekts ändert. Das macht sie sehr performant – identisch zu einer reinen Funktion. Werden Objekt-Properties mutiert (ohne neue Referenz), bemerkt Angular die Änderung nicht.

**Impure Pipes** (`pure: false`) werden bei **jedem Change-Detection-Zyklus** ausgeführt – unabhängig davon, ob sich der Eingabewert geändert hat. Das ist mächtig (z. B. bei sich ändernden Arrays, `Date.now()`, HTTP-Subscriptions), aber teuer. Einsatz mit Bedacht.

**Async Pipe** ist die bekannteste Impure Pipe: Sie subscribed automatisch auf ein Observable oder Promise, gibt den aktuellen Wert aus und unsubscribed beim Zerstören der Komponente.

Seit Angular 17+ können Pipes auch als **Standalone** markiert werden (`standalone: true` im `@Pipe`-Decorator) und direkt in `imports` einer Standalone-Komponente eingebunden werden.

Eine Best Practice: Pure Pipes als Ersatz für Methoden im Template verwenden – Methoden werden bei jedem CD-Zyklus aufgerufen, Pure Pipes nicht.

## Aufgabe

Erstelle eine Produktlisten-Komponente mit drei Custom Pipes:

1. **`CurrencyFormatPipe`** (pure) – Formatiert eine Zahl als Währung mit konfigurierbarem Symbol und Nachkommastellen.
2. **`FilterByPipe`** (impure) – Filtert ein Array von Produkten nach einem Suchstring (demonstriert, warum impure hier nötig ist, wenn das Array mutiert wird).
3. **`StockStatusPipe`** (pure) – Gibt einen Label-String (`'Verfügbar'`, `'Niedrig'`, `'Ausverkauft'`) basierend auf einem Zahlenwert zurück.

### Schritte

1. Erstelle ein neues Standalone-Projekt oder nutze eine bestehende Angular-App. Generiere die drei Pipes mit der CLI:
   ```bash
   ng generate pipe pipes/currency-format --standalone
   ng generate pipe pipes/filter-by --standalone
   ng generate pipe pipes/stock-status --standalone
   ```

2. Implementiere `CurrencyFormatPipe` als pure Pipe:
   - `transform(value: number, symbol = '€', decimals = 2): string`
   - Nutze `Intl.NumberFormat` für eine saubere Formatierung.

3. Implementiere `FilterByPipe` als impure Pipe:
   - `transform(products: Product[], query: string): Product[]`
   - Setze `pure: false` im `@Pipe`-Decorator.
   - Filtere nach `name` und `category` (case-insensitive).

4. Implementiere `StockStatusPipe` als pure Pipe:
   - `transform(stock: number): 'Verfügbar' | 'Niedrig' | 'Ausverkauft'`
   - `> 10` → Verfügbar, `1–10` → Niedrig, `0` → Ausverkauft.

5. Erstelle eine `ProductListComponent` (standalone), importiere alle drei Pipes und binde sie im Template ein:
   ```html
   <input [(ngModel)]="searchQuery" placeholder="Suchen..." />
   <div *ngFor="let p of products | filterBy: searchQuery">
     {{ p.name }} – {{ p.price | currencyFormat: '€' : 2 }}
     <span>{{ p.stock | stockStatus }}</span>
   </div>
   ```

6. Füge einen Button hinzu, der ein Produkt zum Array hinzufügt **ohne** das Array zu ersetzen (`push()`). Beobachte: Die `FilterByPipe` (impure) reagiert, eine hypothetische pure Pipe würde es nicht.

## Hints

<details>
<summary>Hint 1 – Intl.NumberFormat</summary>

```typescript
transform(value: number, symbol = '€', decimals = 2): string {
  return new Intl.NumberFormat('de-DE', {
    style: 'currency',
    currency: symbol === '€' ? 'EUR' : 'USD',
    minimumFractionDigits: decimals,
    maximumFractionDigits: decimals,
  }).format(value);
}
```

</details>

<details>
<summary>Hint 2 – Impure Pipe Decorator</summary>

```typescript
@Pipe({
  name: 'filterBy',
  standalone: true,
  pure: false,   // ← Schlüssel: wird bei jedem CD-Zyklus ausgeführt
})
export class FilterByPipe implements PipeTransform {
  transform(products: Product[], query: string): Product[] {
    if (!query) return products;
    const q = query.toLowerCase();
    return products.filter(
      p => p.name.toLowerCase().includes(q) || p.category.toLowerCase().includes(q)
    );
  }
}
```

</details>

## Beispiellösung

```typescript
// product.model.ts
export interface Product {
  id: number;
  name: string;
  category: string;
  price: number;
  stock: number;
}

// pipes/currency-format.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'currencyFormat', standalone: true })
export class CurrencyFormatPipe implements PipeTransform {
  transform(value: number, symbol = '€', decimals = 2): string {
    const locale = symbol === '€' ? 'de-DE' : 'en-US';
    const currency = symbol === '€' ? 'EUR' : 'USD';
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency,
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals,
    }).format(value);
  }
}

// pipes/stock-status.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'stockStatus', standalone: true })
export class StockStatusPipe implements PipeTransform {
  transform(stock: number): 'Verfügbar' | 'Niedrig' | 'Ausverkauft' {
    if (stock > 10) return 'Verfügbar';
    if (stock > 0) return 'Niedrig';
    return 'Ausverkauft';
  }
}

// pipes/filter-by.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { Product } from '../product.model';

@Pipe({ name: 'filterBy', standalone: true, pure: false })
export class FilterByPipe implements PipeTransform {
  transform(products: Product[], query: string): Product[] {
    if (!query?.trim()) return products;
    const q = query.toLowerCase();
    return products.filter(
      p => p.name.toLowerCase().includes(q) || p.category.toLowerCase().includes(q)
    );
  }
}

// product-list.component.ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { NgFor } from '@angular/common';
import { Product } from './product.model';
import { CurrencyFormatPipe } from './pipes/currency-format.pipe';
import { FilterByPipe } from './pipes/filter-by.pipe';
import { StockStatusPipe } from './pipes/stock-status.pipe';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [FormsModule, NgFor, CurrencyFormatPipe, FilterByPipe, StockStatusPipe],
  template: `
    <input [(ngModel)]="searchQuery" placeholder="Produkt suchen..." />
    <button (click)="addProduct()">+ Produkt hinzufügen</button>

    <ul>
      @for (p of products | filterBy: searchQuery; track p.id) {
        <li>
          <strong>{{ p.name }}</strong> ({{ p.category }})
          — {{ p.price | currencyFormat: '€' : 2 }}
          <em [class]="p.stock | stockStatus | lowercase">
            [{{ p.stock | stockStatus }}]
          </em>
        </li>
      }
    </ul>
  `,
})
export class ProductListComponent {
  searchQuery = '';

  products: Product[] = [
    { id: 1, name: 'Laptop Pro', category: 'Elektronik', price: 1299.99, stock: 15 },
    { id: 2, name: 'Wireless Mouse', category: 'Zubehör', price: 49.95, stock: 5 },
    { id: 3, name: 'USB-C Hub', category: 'Zubehör', price: 39.9, stock: 0 },
    { id: 4, name: 'Monitor 27"', category: 'Elektronik', price: 349.0, stock: 8 },
  ];

  private nextId = 5;

  addProduct(): void {
    // push() mutiert das Array – nur eine impure FilterByPipe erkennt das!
    this.products.push({
      id: this.nextId++,
      name: `Neues Produkt ${this.nextId}`,
      category: 'Zubehör',
      price: Math.random() * 100,
      stock: Math.floor(Math.random() * 20),
    });
  }
}
```

## Weiterführendes

- **Memoization selbst implementieren:** Wenn eine impure Pipe nötig ist, aber Performance kritisch bleibt, kannst du intern cachen (z. B. mit einem `Map<string, Product[]>`), der als Key den Suchstring verwendet – so kombinierst du beide Vorteile.
- **Pure Pipe als Template-Method-Ersatz:** Ersetzt Methoden im Template (`{{ getLabel(x) }}`) immer durch pure Pipes – Angular führt `getLabel()` bei jedem CD-Zyklus aus, die pure Pipe nur bei Eingabeänderung.
- **Offizielle Docs:** [angular.dev/guide/pipes](https://angular.dev/guide/pipes) – besonders der Abschnitt "Detecting pure changes" ist lesenswert.
