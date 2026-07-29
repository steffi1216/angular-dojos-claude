# Angular Dojo: Custom RxJS Operators
**Datum:** 2026-07-29
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, eigene RxJS-Operatoren zu erstellen – sowohl `MonoTypeOperatorFunction` für Typ-erhaltende als auch `OperatorFunction<T, R>` für typumwandelnde Operatoren – und setzt sie in Angular-Services wiederverwendbar ein.

## Hintergrund & Theorie

RxJS-Operatoren sind Funktionen, die ein Observable entgegennehmen und ein neues Observable zurückgeben. Eigene Operatoren zu schreiben bedeutet, diese Signatur zu implementieren:

```typescript
// Typ-erhaltend (T → T)
type MonoTypeOperatorFunction<T> = (source: Observable<T>) => Observable<T>

// Typumwandelnd (T → R)
type OperatorFunction<T, R> = (source: Observable<T>) => Observable<R>
```

Das Schöne daran: Ein eigener Operator ist **nichts anderes als eine Funktion**, die bestehende Operatoren mit `pipe()` kombiniert. So lassen sich wiederkehrende Muster kapseln:

```typescript
export function retryWithDelay<T>(count: number, delayMs: number): MonoTypeOperatorFunction<T> {
  return (source) => source.pipe(
    retry({ count, delay: delayMs }),
  );
}
```

Typische Anwendungsfälle in Angular:
- **HTTP-Fehlerbehandlung** (automatischer Retry, Logging, Error-Mapping)
- **Loading-State-Management** (Ergebnisse in `{ loading, data, error }` verpacken)
- **Debounce + distinctUntilChanged** für Suchfelder als wiederverwendbarer Operator
- **Caching** mit `shareReplay` und automatischem Reset

Der Vorteil: Der Code bleibt in Services und Komponenten schlank, Logik ist testbar und einmalig definiert.

## Aufgabe

Erstelle zwei Custom Operators für einen `ProductSearchService`:

1. **`filterEmpty()`** – ein `MonoTypeOperatorFunction<string>`, das leere Strings und Strings unter 2 Zeichen herausfiltert und zusätzlich `debounceTime(300)` + `distinctUntilChanged()` kombiniert.

2. **`mapToLoadingState<T>()`** – ein `OperatorFunction<T, LoadingState<T>>`, das jeden Wert in `{ status: 'loaded', data: T }` umwandelt, Fehler in `{ status: 'error', error: string }` und beim Start `{ status: 'loading' }` emittiert.

Verwende beide Operatoren in einem `ProductSearchService`, der eine (simulierte) HTTP-Anfrage macht, und binde das Ergebnis in einer Komponente mit `AsyncPipe` ans Template.

### Schritte

1. Erstelle `src/app/operators/filter-empty.operator.ts` mit der `filterEmpty()`-Funktion.
2. Erstelle `src/app/operators/loading-state.operator.ts` mit dem `LoadingState<T>`-Typ und `mapToLoadingState<T>()`.
3. Erstelle `src/app/services/product-search.service.ts`, der `HttpClient` nutzt (oder eine simulierte `of()`-Quelle) und beide Operatoren anwendet.
4. Erstelle eine `ProductSearchComponent` mit einem `<input>` (via `FormControl`) und einem Template, das die drei Zustände (`loading`, `loaded`, `error`) anzeigt.
5. Schreibe einen Unit-Test für `mapToLoadingState()` mit `TestScheduler` oder `firstValueFrom`.

## Hints

<details>
<summary>Hint 1 – Struktur von mapToLoadingState</summary>

```typescript
export type LoadingState<T> =
  | { status: 'loading' }
  | { status: 'loaded'; data: T }
  | { status: 'error'; error: string };

export function mapToLoadingState<T>(): OperatorFunction<T, LoadingState<T>> {
  return (source) =>
    source.pipe(
      map((data): LoadingState<T> => ({ status: 'loaded', data })),
      catchError((err): Observable<LoadingState<T>> =>
        of({ status: 'error', error: err.message ?? 'Unbekannter Fehler' })
      ),
      startWith<LoadingState<T>>({ status: 'loading' }),
    );
}
```

</details>

<details>
<summary>Hint 2 – filterEmpty und Service-Verkettung</summary>

```typescript
// filter-empty.operator.ts
export function filterEmpty(): MonoTypeOperatorFunction<string> {
  return (source) =>
    source.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter((v) => v.trim().length >= 2),
    );
}

// Im Service:
search(term$: Observable<string>): Observable<LoadingState<Product[]>> {
  return term$.pipe(
    filterEmpty(),
    switchMap((term) =>
      this.http.get<Product[]>(`/api/products?q=${term}`).pipe(
        mapToLoadingState(),
      )
    ),
  );
}
```

`switchMap` + `mapToLoadingState()` innerhalb des `switchMap` ist wichtig, damit jeder neue Suchbegriff seinen eigenen `loading`-State auslöst und abgebrochene Requests nicht das Ergebnis überschreiben.

</details>

## Beispiellösung

```typescript
// src/app/operators/loading-state.operator.ts
import { Observable, of, OperatorFunction } from 'rxjs';
import { catchError, map, startWith } from 'rxjs/operators';

export type LoadingState<T> =
  | { status: 'loading' }
  | { status: 'loaded'; data: T }
  | { status: 'error'; error: string };

export function mapToLoadingState<T>(): OperatorFunction<T, LoadingState<T>> {
  return (source) =>
    source.pipe(
      map((data): LoadingState<T> => ({ status: 'loaded', data })),
      catchError((err): Observable<LoadingState<T>> =>
        of({ status: 'error', error: err?.message ?? 'Fehler' })
      ),
      startWith<LoadingState<T>>({ status: 'loading' }),
    );
}

// src/app/operators/filter-empty.operator.ts
import { MonoTypeOperatorFunction } from 'rxjs';
import { debounceTime, distinctUntilChanged, filter } from 'rxjs/operators';

export function filterEmpty(): MonoTypeOperatorFunction<string> {
  return (source) =>
    source.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter((v) => v.trim().length >= 2),
    );
}

// src/app/services/product-search.service.ts
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, switchMap } from 'rxjs';
import { filterEmpty } from '../operators/filter-empty.operator';
import { LoadingState, mapToLoadingState } from '../operators/loading-state.operator';

export interface Product { id: number; name: string; }

@Injectable({ providedIn: 'root' })
export class ProductSearchService {
  private http = inject(HttpClient);

  search(term$: Observable<string>): Observable<LoadingState<Product[]>> {
    return term$.pipe(
      filterEmpty(),
      switchMap((term) =>
        this.http.get<Product[]>(`/api/products?q=${encodeURIComponent(term)}`).pipe(
          mapToLoadingState(),
        )
      ),
    );
  }
}

// src/app/product-search/product-search.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { AsyncPipe, NgIf, NgFor } from '@angular/common';
import { ProductSearchService, Product } from '../services/product-search.service';
import { LoadingState } from '../operators/loading-state.operator';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-product-search',
  standalone: true,
  imports: [ReactiveFormsModule, AsyncPipe, NgIf, NgFor],
  template: `
    <input [formControl]="searchControl" placeholder="Produkt suchen..." />

    @if (result$ | async; as result) {
      @switch (result.status) {
        @case ('loading') { <p>Laden...</p> }
        @case ('error')   { <p class="error">{{ result.error }}</p> }
        @case ('loaded')  {
          <ul>
            @for (product of result.data; track product.id) {
              <li>{{ product.name }}</li>
            }
          </ul>
        }
      }
    }
  `,
})
export class ProductSearchComponent implements OnInit {
  private searchService = inject(ProductSearchService);
  searchControl = new FormControl('', { nonNullable: true });
  result$!: Observable<LoadingState<Product[]>>;

  ngOnInit(): void {
    this.result$ = this.searchService.search(
      this.searchControl.valueChanges,
    );
  }
}

// src/app/operators/loading-state.operator.spec.ts
import { firstValueFrom, of, throwError } from 'rxjs';
import { mapToLoadingState } from './loading-state.operator';

describe('mapToLoadingState', () => {
  it('emittiert loading → loaded', async () => {
    const results: unknown[] = [];
    of(42).pipe(mapToLoadingState()).subscribe((v) => results.push(v));

    expect(results).toEqual([
      { status: 'loading' },
      { status: 'loaded', data: 42 },
    ]);
  });

  it('emittiert loading → error bei Fehler', async () => {
    const results: unknown[] = [];
    throwError(() => new Error('Netzwerkfehler'))
      .pipe(mapToLoadingState())
      .subscribe((v) => results.push(v));

    expect(results).toEqual([
      { status: 'loading' },
      { status: 'error', error: 'Netzwerkfehler' },
    ]);
  });
});
```

## Weiterführendes

- **Typsicherheit testen**: Übergib den Operator an `pipe()` in TypeScript und prüfe, ob der Rückgabetyp korrekt inferiert wird – das zeigt dir sofort, ob deine Generics stimmen.
- **`TestScheduler`** aus `rxjs/testing` ermöglicht marble-basierte Tests für zeitabhängige Operatoren wie `filterEmpty()` – ideal für präzises Testen von `debounceTime`.
- Lies die [RxJS-Doku zu operator creation](https://rxjs.dev/guide/operators#creating-new-operators-from-scratch) für Low-Level-Ansätze mit `new Observable()`.
- Kombiniere `mapToLoadingState()` mit `toSignal({ initialValue: { status: 'loading' } })` für einen vollständig Signal-basierten Ansatz ohne `AsyncPipe`.
