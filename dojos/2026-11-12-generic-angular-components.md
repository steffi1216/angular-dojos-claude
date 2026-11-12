# Angular Dojo: Generic Angular Components mit TypeScript Generics
**Datum:** 2026-11-12
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular-Komponenten und Direktiven mit TypeScript-Generics typsicher und wiederverwendbar gestaltest – für Listen, Tabellen, Select-Felder und andere Muster, die über verschiedene Datentypen hinweg funktionieren.

## Hintergrund & Theorie

TypeScript Generics ermöglichen es, Komponenten zu bauen, die mit beliebigen Datentypen arbeiten, ohne dabei die Typsicherheit zu verlieren. In Angular gibt es mehrere Möglichkeiten, generische Komponenten umzusetzen:

**1. Generische Klassen mit `@Component`:**
Angular-Komponenten können generische Klassen sein – der Compiler löst den Typ im Template auf, wenn du die Komponente mit einem konkreten Typ verwendest.

**2. Generic Inputs mit `input<T>()`:**
Mit der neuen Signal-basierten `input()`-API kannst du typsichere generische Inputs definieren.

**3. `NgTemplateOutlet` mit typisierten Kontext-Variablen:**
Durch Template-Slots (`ng-template`) übergibst du den konkreten Rendervorgang an den Aufrufer – zusammen mit `$implicit` und benannten Kontextvariablen bleibt dabei alles typsicher.

**Typischer Anwendungsfall:** Eine `SelectionListComponent<T>`, die beliebige Items anzeigt, eine `displayFn` akzeptiert und das selektierte Item als `T` zurückgibt – ohne dass jede Seite eine eigene Listenkomponente braucht.

> **Wichtig:** TypeScript leitet den Generic-Typ aus dem ersten `input()`-Signal ab. Sobald `items` einen konkreten Typ bekommt, kennt der Compiler auch den Rückgabetyp von `selectedItem`.

## Aufgabe

Baue eine generische `SelectionListComponent<T>`, die:
1. Eine beliebige Array-Liste `items: T[]` rendert
2. Eine `displayFn: (item: T) => string` für das Label akzeptiert
3. Bei Klick ein `selected` Output-Event mit dem Typ `T` emittiert
4. Optional: einen `itemTemplate`-Input (TemplateRef) für Custom-Rendering unterstützt

Teste die Komponente mit zwei verschiedenen Datentypen (`User` und `Product`).

### Schritte

1. Erstelle das Interface `User { id: number; name: string; email: string }` und `Product { sku: string; label: string; price: number }`.
2. Erstelle die generische Komponente `SelectionListComponent<T>` mit Signal-Inputs (`items`, `displayFn`, `itemTemplate`) und einem `output<T>()`.
3. Baue das Template: iteriere über `items`, zeige entweder das Custom-Template (`itemTemplate` via `NgTemplateOutlet`) oder das Standard-Label per `displayFn`.
4. Erstelle eine Host-Komponente, die `SelectionListComponent` einmal mit `User[]` und einmal mit `Product[]` verwendet.
5. Stelle sicher, dass TypeScript beim falschen Typ einen Fehler wirft (z.B. wenn du `displayFn` mit `(item: Product) => string` an die User-Liste übergibst).

## Hints

<details>
<summary>Hint 1 – Generische Klasse deklarieren</summary>

```typescript
@Component({ ... })
export class SelectionListComponent<T> {
  items = input.required<T[]>();
  displayFn = input.required<(item: T) => string>();
  itemTemplate = input<TemplateRef<{ $implicit: T }>>();
  selected = output<T>();

  onSelect(item: T) {
    this.selected.emit(item);
  }
}
```

TypeScript leitet `T` automatisch aus dem `items`-Input ab, sobald du die Komponente im Template verwendest.

</details>

<details>
<summary>Hint 2 – NgTemplateOutlet mit typisiertem Kontext</summary>

```html
@for (item of items(); track $index) {
  <li (click)="onSelect(item)">
    @if (itemTemplate()) {
      <ng-container
        [ngTemplateOutlet]="itemTemplate()!"
        [ngTemplateOutletContext]="{ $implicit: item }">
      </ng-container>
    } @else {
      {{ displayFn()(item) }}
    }
  </li>
}
```

Im aufrufenden Template nimmst du den Typ per `let-item`:
```html
<ng-template #customTpl let-product>
  <strong>{{ product.label }}</strong> – {{ product.price | currency }}
</ng-template>
```

</details>

<details>
<summary>Hint 3 – Typ-Inferenz im Host-Template erzwingen</summary>

Angular leitet `T` aus dem `[items]`-Binding ab. Damit `displayFn` ebenfalls den richtigen Typ bekommt, definiere sie im Component-Code mit explizitem Typ:

```typescript
// host.component.ts
userDisplayFn = (user: User): string => user.name;
productDisplayFn = (product: Product): string => `${product.label} (${product.sku})`;
```

Wenn du `userDisplayFn` an `[displayFn]` einer `Product`-Liste bindest, wirft der Compiler einen Typfehler – genau das ist das Ziel.

</details>

## Beispiellösung

```typescript
// selection-list.component.ts
import { Component, input, output, TemplateRef } from '@angular/core';
import { NgTemplateOutlet } from '@angular/common';

@Component({
  selector: 'app-selection-list',
  standalone: true,
  imports: [NgTemplateOutlet],
  template: `
    <ul>
      @for (item of items(); track $index) {
        <li (click)="onSelect(item)" style="cursor:pointer; padding: 4px 0">
          @if (itemTemplate()) {
            <ng-container
              [ngTemplateOutlet]="itemTemplate()!"
              [ngTemplateOutletContext]="{ $implicit: item }">
            </ng-container>
          } @else {
            {{ displayFn()(item) }}
          }
        </li>
      }
    </ul>
  `,
})
export class SelectionListComponent<T> {
  items = input.required<T[]>();
  displayFn = input.required<(item: T) => string>();
  itemTemplate = input<TemplateRef<{ $implicit: T }>>();

  selected = output<T>();

  onSelect(item: T): void {
    this.selected.emit(item);
  }
}

// models.ts
export interface User {
  id: number;
  name: string;
  email: string;
}

export interface Product {
  sku: string;
  label: string;
  price: number;
}

// host.component.ts
import { Component, signal } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { SelectionListComponent } from './selection-list.component';
import { User, Product } from './models';

@Component({
  selector: 'app-host',
  standalone: true,
  imports: [SelectionListComponent, CurrencyPipe],
  template: `
    <h2>Benutzer</h2>
    <app-selection-list
      [items]="users()"
      [displayFn]="userDisplayFn"
      (selected)="selectedUser.set($event)">
    </app-selection-list>
    @if (selectedUser()) {
      <p>Ausgewählt: {{ selectedUser()!.name }} ({{ selectedUser()!.email }})</p>
    }

    <h2>Produkte (Custom Template)</h2>
    <app-selection-list
      [items]="products()"
      [displayFn]="productDisplayFn"
      [itemTemplate]="productTpl"
      (selected)="selectedProduct.set($event)">
    </app-selection-list>
    @if (selectedProduct()) {
      <p>Ausgewählt: {{ selectedProduct()!.label }} – {{ selectedProduct()!.price | currency }}</p>
    }

    <ng-template #productTpl let-product>
      <strong>{{ product.label }}</strong>
      <small style="margin-left:8px; color: gray">{{ product.sku }}</small>
    </ng-template>
  `,
})
export class HostComponent {
  users = signal<User[]>([
    { id: 1, name: 'Alice', email: 'alice@example.com' },
    { id: 2, name: 'Bob', email: 'bob@example.com' },
    { id: 3, name: 'Carol', email: 'carol@example.com' },
  ]);

  products = signal<Product[]>([
    { sku: 'P-001', label: 'Mechanische Tastatur', price: 149.99 },
    { sku: 'P-002', label: 'USB-C Hub', price: 49.99 },
    { sku: 'P-003', label: 'Webcam 4K', price: 229.99 },
  ]);

  selectedUser = signal<User | null>(null);
  selectedProduct = signal<Product | null>(null);

  userDisplayFn = (user: User): string => `${user.name} <${user.email}>`;
  productDisplayFn = (product: Product): string => product.label;
}
```

**Typfehler testen:**
```typescript
// Dieser Code erzeugt einen Compile-Fehler:
// Argument of type '(user: User) => string' is not assignable
// to parameter of type '(item: Product) => string'.
<app-selection-list
  [items]="products()"
  [displayFn]="userDisplayFn"  // ← TypeScript-Fehler!
  (selected)="selectedProduct.set($event)">
```

## Weiterführendes

- **Generische Services:** Dasselbe Muster funktioniert für Services – z.B. ein `CrudService<T>` mit Methoden wie `getAll(): Observable<T[]>` und `save(item: T): Observable<T>`.
- **Strikte Template-Typprüfung aktivieren:** Mit `strictTemplates: true` in `tsconfig.json` erzwingt Angular die vollständige Typ-Inferenz auch im Template-Compiler – ein Muss für generische Komponenten.
- **Offizielle Dokumentation:** [Angular – Generic Components](https://angular.dev/guide/components/component-host)
