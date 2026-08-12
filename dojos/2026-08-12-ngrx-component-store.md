# Angular Dojo: NgRx Component Store
**Datum:** 2026-08-12
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Lerne, wie du mit `@ngrx/component-store` lokalen, komponentengebundenen State verwaltest – ohne den globalen NgRx Store zu belasten. Du übst das Muster aus `State`, `Updater`, `Effect` und `Selector` auf Komponentenebene.

## Hintergrund & Theorie
Der globale NgRx Store eignet sich für App-weiten State. Für State, der nur in einer Komponente (oder einem Feature-Baum) lebt, ist `@ngrx/component-store` die elegantere Alternative: Er ist leichtgewichtiger, direkt an den Komponentenlebenszyklus gekoppelt und vermeidet Boilerplate wie Actions/Reducers.

Kernkonzepte:

- **`ComponentStore<T>`** – Basisklasse, von der dein Service erbt. Initialisiert den State mit `setState`.
- **`select()`** – Leitet Observables aus dem State ab (ähnlich wie NgRx Selectors).
- **`updater()`** – Reine Funktion, die den State immutabel aktualisiert (ähnlich einem Reducer).
- **`effect()`** – Behandelt Seiteneffekte (z. B. HTTP-Calls); greift auf den State zu und aktualisiert ihn.
- **Lifecycle**: Der Store wird als `providers`-Eintrag in der Komponente bereitgestellt und lebt genau so lange wie sie.

Der Component Store eignet sich besonders für Dialoge, komplexe Formulare, Listen mit Filterzustand oder wiederverwendbare Feature-Komponenten.

## Aufgabe
Erstelle eine Produktlisten-Komponente mit eigenem Component Store. Der Store soll eine Liste von Produkten laden (simulierter HTTP-Call), einen Ladezustand verwalten und eine Filterfunktion nach Kategorie bereitstellen.

### Schritte
1. Installiere `@ngrx/component-store` (falls nicht vorhanden) und erstelle einen `ProductsStore`-Service, der von `ComponentStore<ProductsState>` erbt.
2. Definiere den State-Typ `ProductsState` mit den Feldern `products`, `loading` und `selectedCategory`.
3. Implementiere einen `updater` (`setCategory`) und einen `effect` (`loadProducts`), der nach einer kurzen Verzögerung Dummy-Produkte in den State schreibt.
4. Erstelle die Selektoren `products$` (gefiltert nach `selectedCategory`) und `loading$`.
5. Binde den Store als `providers`-Eintrag in einer Standalone-Komponente ein und zeige die gefilterte Liste sowie einen Kategorie-Filter im Template an.

## Hints
<details>
<summary>Hint 1 – State-Typ und Initialisierung</summary>

```typescript
interface ProductsState {
  products: Product[];
  loading: boolean;
  selectedCategory: string | null;
}

@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  constructor() {
    super({ products: [], loading: false, selectedCategory: null });
  }
}
```
</details>

<details>
<summary>Hint 2 – Selector mit Filterlogik</summary>

```typescript
readonly filteredProducts$ = this.select(({ products, selectedCategory }) =>
  selectedCategory
    ? products.filter(p => p.category === selectedCategory)
    : products
);
```
</details>

<details>
<summary>Hint 3 – Effect für asynchrones Laden</summary>

```typescript
readonly loadProducts = this.effect<void>(trigger$ =>
  trigger$.pipe(
    tap(() => this.patchState({ loading: true })),
    switchMap(() =>
      of(DUMMY_PRODUCTS).pipe(
        delay(800),
        tapResponse(
          products => this.patchState({ products, loading: false }),
          () => this.patchState({ loading: false })
        )
      )
    )
  )
);
```
</details>

## Beispiellösung

```typescript
// products.store.ts
import { Injectable } from '@angular/core';
import { ComponentStore, tapResponse } from '@ngrx/component-store';
import { delay, of, switchMap, tap } from 'rxjs';

interface Product {
  id: number;
  name: string;
  category: string;
}

interface ProductsState {
  products: Product[];
  loading: boolean;
  selectedCategory: string | null;
}

const DUMMY_PRODUCTS: Product[] = [
  { id: 1, name: 'Laptop', category: 'Electronics' },
  { id: 2, name: 'Headphones', category: 'Electronics' },
  { id: 3, name: 'Desk Chair', category: 'Furniture' },
  { id: 4, name: 'Bookshelf', category: 'Furniture' },
  { id: 5, name: 'Running Shoes', category: 'Sports' },
];

@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  constructor() {
    super({ products: [], loading: false, selectedCategory: null });
  }

  // Selectors
  readonly loading$ = this.select(s => s.loading);
  readonly selectedCategory$ = this.select(s => s.selectedCategory);
  readonly filteredProducts$ = this.select(({ products, selectedCategory }) =>
    selectedCategory
      ? products.filter(p => p.category === selectedCategory)
      : products
  );
  readonly categories$ = this.select(({ products }) =>
    [...new Set(products.map(p => p.category))]
  );

  // Updaters
  readonly setCategory = this.updater((state, category: string | null) => ({
    ...state,
    selectedCategory: category,
  }));

  // Effects
  readonly loadProducts = this.effect<void>(trigger$ =>
    trigger$.pipe(
      tap(() => this.patchState({ loading: true })),
      switchMap(() =>
        of(DUMMY_PRODUCTS).pipe(
          delay(800),
          tapResponse(
            products => this.patchState({ products, loading: false }),
            () => this.patchState({ loading: false })
          )
        )
      )
    )
  );
}
```

```typescript
// products.component.ts
import { Component, OnInit } from '@angular/core';
import { AsyncPipe, NgFor, NgIf } from '@angular/common';
import { ProductsStore } from './products.store';

@Component({
  selector: 'app-products',
  standalone: true,
  imports: [AsyncPipe, NgFor, NgIf],
  providers: [ProductsStore],
  template: `
    <div *ngIf="store.loading$ | async">Laden...</div>

    <div>
      <button (click)="store.setCategory(null)">Alle</button>
      <button
        *ngFor="let cat of store.categories$ | async"
        (click)="store.setCategory(cat)"
      >
        {{ cat }}
      </button>
    </div>

    <ul>
      <li *ngFor="let product of store.filteredProducts$ | async">
        {{ product.name }} <em>({{ product.category }})</em>
      </li>
    </ul>
  `,
})
export class ProductsComponent implements OnInit {
  constructor(readonly store: ProductsStore) {}

  ngOnInit() {
    this.store.loadProducts();
  }
}
```

## Weiterführendes
- **Kombination mit Signals**: Ab Angular 17+ kannst du `toSignal(this.select(...))` verwenden, um Observables aus dem Component Store in Signals umzuwandeln und moderne Template-Syntax (`@if`, `@for`) zu nutzen.
- **Offizielle Doku**: [ngrx.io/guide/component-store](https://ngrx.io/guide/component-store) – besonders die Abschnitte zu *Initialization*, *Lifecycle* und *tapResponse*.
- **Wann Store, wann Component Store?** Als Faustregel: Wird der State in mehr als einer Routen-Ebene benötigt → globaler Store. Ist er auf eine Feature-Komponente beschränkt → Component Store.
