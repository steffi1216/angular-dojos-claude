# Angular Dojo: Performance Optimierung
**Datum:** 2026-06-25
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular-Anwendungen mit `trackBy`, reinen Pipes und Memoization gezielt optimierst, sodass unnötige Re-Renders und teure Berechnungen vermieden werden.

## Hintergrund & Theorie

Angular rendert standardmäßig bei jeder Change-Detection-Runde alle Templates neu. Drei Techniken helfen, die Arbeit pro Runde drastisch zu reduzieren:

**trackBy in `*ngFor`**
Ohne `trackBy` ersetzt Angular bei Array-Änderungen alle DOM-Knoten, auch wenn sich nur ein Element verändert hat. Eine `trackBy`-Funktion gibt jedem Element eine stabile Identität (z. B. die `id`), sodass nur geänderte Knoten neu gerendert werden.

**Pure Pipes**
Eine Pure Pipe wird nur neu ausgeführt, wenn sich ihre Eingabe (Referenz) ändert. Damit lassen sich teure Transformationen (Filtern, Sortieren, Formatieren) aus dem Template auslagern, ohne sie bei jedem Tick neu zu berechnen. Angular-Standardpipes wie `date`, `currency`, `async` sind alle pure.

**Memoization**
Memoization cached das Ergebnis einer Funktion anhand ihrer Argumente. In Angular bietet sich das an für Selektoren (NgRx `createSelector` memoized automatisch), aber auch für eigene Service-Methoden oder Computed-Werte mit Signals (`computed()`).

Zusammen mit der `OnPush`-Change-Detection-Strategie bilden diese drei Techniken das Fundament einer performanten Angular-Applikation.

## Aufgabe

Du hast eine Produktliste, die serverseitig sehr groß sein kann (1 000+ Einträge). Die Anforderungen:

1. Die Liste rendert Produkte mit `*ngFor` und soll DOM-Recycling nutzen.
2. Ein Suchfeld filtert die Produkte; die Filterlogik soll **nicht** bei jedem Tastendruck im Template-Code erneut ausgeführt werden.
3. Ein "teures" Label (Rabatt in Prozent) wird pro Produkt berechnet – die Berechnung soll nur bei echter Eingabeänderung wiederholt werden.

### Schritte

1. Erstelle ein neues Standalone-Component `ProductListComponent` mit `OnPush` Change Detection.
2. Definiere ein Array `products` mit mindestens 5 Mock-Produkten (`{ id: number; name: string; price: number; originalPrice: number }`).
3. Füge ein Reactive-Forms-`FormControl` für die Suche hinzu und exponiere die gefilterte Liste als `Signal` (via `toSignal` + `computed`).
4. Erstelle eine **Pure Pipe** `DiscountPipe`, die den Rabatt in Prozent berechnet: `((originalPrice - price) / originalPrice * 100).toFixed(0) + '%'`.
5. Nutze in deinem Template `*ngFor` mit einer `trackBy`-Funktion auf `product.id`.
6. Zeige den Discount pro Zeile mithilfe der `DiscountPipe` an.
7. Öffne die Angular DevTools (Chrome Extension) → Profiler und prüfe, dass bei einer Suche nur betroffene Komponenten markiert werden.

## Hints

<details>
<summary>Hint 1 – Signal-basierte Filterung</summary>

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { computed, signal } from '@angular/core';

// Suchfeld als Signal
const searchValue = toSignal(this.searchCtrl.valueChanges, { initialValue: '' });

// Gefilterte Liste – wird nur neu berechnet wenn searchValue oder products sich ändern
filteredProducts = computed(() =>
  this.products.filter(p =>
    p.name.toLowerCase().includes((searchValue() ?? '').toLowerCase())
  )
);
```

</details>

<details>
<summary>Hint 2 – trackBy-Funktion</summary>

```typescript
trackByProductId(_index: number, product: Product): number {
  return product.id;
}
```

Im Template:
```html
<li *ngFor="let product of filteredProducts(); trackBy: trackByProductId">
```

</details>

<details>
<summary>Hint 3 – Pure Pipe Grundgerüst</summary>

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'discount', standalone: true, pure: true })
export class DiscountPipe implements PipeTransform {
  transform(price: number, originalPrice: number): string {
    if (originalPrice <= 0) return '0%';
    return ((originalPrice - price) / originalPrice * 100).toFixed(0) + '%';
  }
}
```

</details>

## Beispiellösung

```typescript
// product-list.component.ts
import { ChangeDetectionStrategy, Component, inject } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { NgFor } from '@angular/common';
import { computed } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { DiscountPipe } from './discount.pipe';

interface Product {
  id: number;
  name: string;
  price: number;
  originalPrice: number;
}

@Component({
  selector: 'app-product-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule, NgFor, DiscountPipe],
  template: `
    <input [formControl]="searchCtrl" placeholder="Suche..." />

    <ul>
      <li *ngFor="let p of filteredProducts(); trackBy: trackById">
        {{ p.name }} – {{ p.price | currency:'EUR' }}
        <span class="badge">-{{ p.price | discount:p.originalPrice }}</span>
      </li>
    </ul>
  `,
})
export class ProductListComponent {
  searchCtrl = new FormControl('');

  readonly products: Product[] = [
    { id: 1, name: 'Laptop Pro',    price: 899,  originalPrice: 1199 },
    { id: 2, name: 'Mechanical Keyboard', price: 79, originalPrice: 99 },
    { id: 3, name: 'USB-C Hub',     price: 29,   originalPrice: 45  },
    { id: 4, name: 'Monitor 4K',    price: 349,  originalPrice: 499 },
    { id: 5, name: 'Webcam HD',     price: 59,   originalPrice: 79  },
  ];

  private searchValue = toSignal(this.searchCtrl.valueChanges, { initialValue: '' });

  filteredProducts = computed(() =>
    this.products.filter(p =>
      p.name.toLowerCase().includes((this.searchValue() ?? '').toLowerCase())
    )
  );

  trackById(_index: number, product: Product): number {
    return product.id;
  }
}
```

```typescript
// discount.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'discount', standalone: true, pure: true })
export class DiscountPipe implements PipeTransform {
  transform(price: number, originalPrice: number): string {
    if (originalPrice <= 0) return '0%';
    return ((originalPrice - price) / originalPrice * 100).toFixed(0) + '%';
  }
}
```

## Weiterführendes

- **Angular DevTools Profiler** – Installiere die Chrome-Extension und nutze den "Record"-Modus, um Change-Detection-Zyklen zu visualisieren. Ziel: möglichst wenige "checked" Komponenten pro Interaktion.
- **`NgRx createSelector`** – Memoization out-of-the-box für Store-Selektoren: Mehrfachaufrufe mit gleichen Inputs geben das gecachte Ergebnis zurück ohne die Projektionsfunktion erneut auszuführen.
- **`@angular/core/rxjs-interop` `takeUntilDestroyed`** – Kombiniere Performance-Techniken mit sauberen Subscriptions, um Memory Leaks zu vermeiden.
- Offizielle Docs: [angular.dev/guide/performance](https://angular.dev/guide/performance)
