# Angular Dojo: NgRx ComponentStore
**Datum:** 2026-11-22
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Lerne, wie du mit NgRx ComponentStore lokalen, reaktiven Komponentenstate verwaltest – als leichtgewichtige Alternative zum globalen NgRx Store.

## Hintergrund & Theorie
NgRx ComponentStore ist eine eigenständige State-Management-Bibliothek für **lokalen Komponentenstate**. Im Gegensatz zum globalen NgRx Store lebt der ComponentStore nur so lange, wie die Komponente, der er zugeordnet ist – kein globales ActionDispatch, keine Reducer-Registrierung.

Die drei Kernkonzepte:

**`state`** – Der initiale State wird im Konstruktor gesetzt (`this.setState(initialState)`).

**`updater`** – Eine pure Funktion, die den State aktualisiert. Vergleichbar mit einem Reducer, aber direkt in der Store-Klasse:
```typescript
readonly setLoading = this.updater((state, loading: boolean) => ({ ...state, loading }));
```

**`effect`** – Für asynchrone Operationen (HTTP-Calls, Timer). Erhält ein Observable als Input und kann den State über `updater` aktualisieren.

**`select`** – Erstellt einen abgeleiteten Slice des States als Observable:
```typescript
readonly items$ = this.select(state => state.items);
```

ComponentStore eignet sich ideal für Seiten mit komplexem lokalen State (z. B. Suchseiten, Wizards, Tabellen mit Filtern), bei denen globaler State Overhead wäre.

## Aufgabe
Erstelle eine Such-Komponente für eine Produktliste mit lokalem State über NgRx ComponentStore. Die Komponente soll Suchbegriff, Ladezustand und Ergebnisse verwalten.

### Schritte

1. **ComponentStore erstellen** – Erstelle `ProductSearchStore` mit dem Interface `ProductSearchState`:
   ```typescript
   interface ProductSearchState {
     query: string;
     loading: boolean;
     products: Product[];
     error: string | null;
   }
   ```

2. **Selectors definieren** – Erstelle `select`-Selectors für `query$`, `loading$`, `products$` und `error$`. Füge einen kombinierten Selector `vm$` (ViewModel) hinzu, der alle vier zusammenfasst.

3. **Updaters implementieren** – Schreibe Updater-Methoden für `setQuery`, `setLoading` und `setError`.

4. **Effect hinzufügen** – Erstelle einen `searchProducts`-Effect, der:
   - Den Ladezustand auf `true` setzt
   - Den HTTP-Call (simuliert mit `timer(500)`) ausführt
   - `switchMap` nutzt, um parallele Anfragen zu verwerfen
   - Bei Erfolg die Produkte setzt, bei Fehler `setError` aufruft

5. **Komponente verdrahten** – Injiziere den Store in eine Standalone-Komponente, binde `vm$` per `async`-Pipe ans Template und rufe `searchProducts` beim Tippen auf.

## Hints

<details>
<summary>Hint 1 – ComponentStore-Klasse aufsetzen</summary>

```typescript
import { ComponentStore } from '@ngrx/component-store';
import { Injectable } from '@angular/core';

@Injectable()
export class ProductSearchStore extends ComponentStore<ProductSearchState> {
  constructor() {
    super({ query: '', loading: false, products: [], error: null });
  }
}
```

Vergiss `@Injectable()` nicht – der Store wird als Provider in der Komponente registriert, nicht global.
</details>

<details>
<summary>Hint 2 – Effect mit switchMap und Fehlerbehandlung</summary>

```typescript
readonly searchProducts = this.effect<string>(query$ =>
  query$.pipe(
    tap(() => this.setLoading(true)),
    switchMap(query =>
      this.productService.search(query).pipe(
        tapResponse(
          products => {
            this.patchState({ products, loading: false, error: null });
          },
          (err: Error) => {
            this.patchState({ error: err.message, loading: false });
          }
        )
      )
    )
  )
);
```

`tapResponse` aus `@ngrx/component-store` ist wie `tap` + `catchError`, beendet den Effect-Stream aber nicht bei Fehlern.
</details>

## Beispiellösung

```typescript
// product-search.store.ts
import { Injectable } from '@angular/core';
import { ComponentStore, tapResponse } from '@ngrx/component-store';
import { switchMap, tap } from 'rxjs/operators';
import { Observable, timer, of } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Product {
  id: number;
  name: string;
}

interface ProductSearchState {
  query: string;
  loading: boolean;
  products: Product[];
  error: string | null;
}

@Injectable()
export class ProductSearchStore extends ComponentStore<ProductSearchState> {
  constructor() {
    super({ query: '', loading: false, products: [], error: null });
  }

  // Selectors
  readonly query$ = this.select(s => s.query);
  readonly loading$ = this.select(s => s.loading);
  readonly products$ = this.select(s => s.products);
  readonly error$ = this.select(s => s.error);

  readonly vm$ = this.select({
    query: this.query$,
    loading: this.loading$,
    products: this.products$,
    error: this.error$,
  });

  // Updaters
  readonly setQuery = this.updater((state, query: string) => ({ ...state, query }));
  readonly setLoading = this.updater((state, loading: boolean) => ({ ...state, loading }));
  readonly setError = this.updater((state, error: string) => ({ ...state, error, loading: false }));

  // Effect
  readonly searchProducts = this.effect<string>(query$ =>
    query$.pipe(
      tap(query => this.setQuery(query)),
      tap(() => this.setLoading(true)),
      switchMap(query =>
        // Simulierter HTTP-Call
        timer(500).pipe(
          map(() => [
            { id: 1, name: `${query} Produkt A` },
            { id: 2, name: `${query} Produkt B` },
          ] as Product[])
        ).pipe(
          tapResponse(
            products => this.patchState({ products, loading: false, error: null }),
            (err: Error) => this.setError(err.message)
          )
        )
      )
    )
  );
}
```

```typescript
// product-search.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { AsyncPipe, NgFor, NgIf } from '@angular/common';
import { ProductSearchStore } from './product-search.store';

@Component({
  selector: 'app-product-search',
  standalone: true,
  imports: [AsyncPipe, NgFor, NgIf],
  providers: [ProductSearchStore],  // lokaler Provider – Store lebt mit der Komponente
  template: `
    <ng-container *ngIf="vm$ | async as vm">
      <input
        [value]="vm.query"
        (input)="onSearch($event)"
        placeholder="Produkt suchen..."
      />

      <p *ngIf="vm.loading">Lädt...</p>
      <p *ngIf="vm.error" style="color: red">{{ vm.error }}</p>

      <ul>
        <li *ngFor="let p of vm.products">{{ p.name }}</li>
      </ul>
    </ng-container>
  `,
})
export class ProductSearchComponent implements OnInit {
  private readonly store = inject(ProductSearchStore);
  readonly vm$ = this.store.vm$;

  ngOnInit() {
    this.store.searchProducts('');
  }

  onSearch(event: Event) {
    const query = (event.target as HTMLInputElement).value;
    this.store.searchProducts(query);
  }
}
```

## Weiterführendes
- **`patchState` vs. `setState` vs. `updater`**: `patchState` ist der einfachste Weg für partielle Updates; `updater` erlaubt parameterisierte Transformationen; `setState` ersetzt den gesamten State.
- Für Signal-basierte Alternative: Seit Angular 17+ kannst du `signalStore` aus `@ngrx/signals` nutzen, der ComponentStore bei neuen Projekten zunehmend ablöst.
- Offizielle Doku: [ngrx.io/guide/component-store](https://ngrx.io/guide/component-store)
