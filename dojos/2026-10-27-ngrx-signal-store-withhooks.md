# Angular Dojo: withHooks – Store-Lifecycle im NgRx Signal Store
**Datum:** 2026-10-27
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `withHooks` Lifecycle-Logik (Initialisierung, Aufräumen) direkt in einen NgRx Signal Store integrierst – ohne separate Services oder manuelle `ngOnInit`/`ngOnDestroy`-Hooks in Komponenten.

## Hintergrund & Theorie

NgRx Signal Store bietet mit `withHooks` eine elegante Möglichkeit, Initialisierungs- und Cleanup-Logik direkt im Store zu verankern. `withHooks` nimmt ein Objekt mit zwei optionalen Funktionen entgegen:

- **`onInit`**: Wird einmalig ausgeführt, wenn der Store instanziiert wird. Ideal für das initiale Laden von Daten, das Starten von Subscriptions oder das Registrieren von Effects.
- **`onDestroy`**: Wird beim Zerstören des Stores ausgeführt. Perfekt für das Abmelden von Subscriptions oder das Freigeben von Ressourcen.

Innerhalb von `withHooks` hat man über Dependency Injection Zugriff auf alle anderen Store-Features (z.B. `withMethods`, `withComputed`), indem man sie als Parameter der Funktion angibt:

```typescript
withHooks({
  onInit(store, httpClient = inject(HttpClient)) {
    // Zugriff auf store-Methoden und DI-Tokens
    store.loadData();
  }
})
```

Ein entscheidender Vorteil: Da der Store `providedIn: 'root'` oder component-level (`providers: [MyStore]`) bereitgestellt werden kann, folgt sein Lifecycle automatisch dem Lifecycle des Injectors – bei Component-level Provision wird `onDestroy` beim Zerstören der Komponente aufgerufen.

`withHooks` ist kombinierbar mit `tapResponse` aus `@ngrx/operators`, um Fehler in Effects elegant zu behandeln.

## Aufgabe

Erstelle einen `ProductStore` mit NgRx Signal Store, der:

1. Produkte beim Initialisieren automatisch aus einem Service lädt (`onInit`)
2. Einen laufenden Poll (Intervall) stoppt, wenn der Store zerstört wird (`onDestroy`)
3. Den Ladezustand korrekt verwaltet (loading / error / data)

### Schritte

1. Erstelle ein Interface `Product` und einen `ProductService` mit einer `getProducts()`-Methode, die `Observable<Product[]>` zurückgibt.

2. Erstelle einen `ProductStore` mit folgenden Features:
   - `withState<ProductState>({ products: [], loading: false, error: null })`
   - `withMethods(...)` für `setLoading`, `setProducts`, `setError`
   - `withHooks(...)` mit `onInit` und `onDestroy`

3. Im `onInit`:
   - Starte einen `interval(10_000)` und lade bei jedem Tick die Produkte neu
   - Speichere die Subscription in einer lokalen Variable
   - Verwende `tapResponse` für die HTTP-Anfrage

4. Im `onDestroy`:
   - Rufe `subscription.unsubscribe()` auf

5. Registriere den `ProductStore` auf Component-Level (`providers: [ProductStore]`) und zeige die Daten im Template an.

## Hints

<details>
<summary>Hint 1 – Store-Grundstruktur</summary>

```typescript
import { signalStore, withState, withMethods, withHooks, patchState } from '@ngrx/signals';

interface ProductState {
  products: Product[];
  loading: boolean;
  error: string | null;
}

const initialState: ProductState = {
  products: [],
  loading: false,
  error: null,
};

export const ProductStore = signalStore(
  withState(initialState),
  withMethods((store) => ({
    setLoading(loading: boolean) {
      patchState(store, { loading });
    },
    setProducts(products: Product[]) {
      patchState(store, { products, loading: false, error: null });
    },
    setError(error: string) {
      patchState(store, { error, loading: false });
    },
  })),
  withHooks({ onInit, onDestroy })
);
```

</details>

<details>
<summary>Hint 2 – onInit und onDestroy mit Interval</summary>

```typescript
import { inject } from '@angular/core';
import { interval, startWith, switchMap } from 'rxjs';
import { tapResponse } from '@ngrx/operators';

function onInit(store: InstanceType<typeof ProductStore>) {
  const productService = inject(ProductService);

  store.setLoading(true);

  const subscription = interval(10_000)
    .pipe(
      startWith(0),
      switchMap(() => productService.getProducts())
    )
    .subscribe(
      tapResponse({
        next: (products) => store.setProducts(products),
        error: (err: Error) => store.setError(err.message),
      })
    );

  return subscription; // wird als Rückgabewert für onDestroy verfügbar
}

function onDestroy(_store: unknown, subscription = /* über Closure */ ...) {
  subscription.unsubscribe();
}
```

**Tipp:** In der Praxis nutzt man oft `DestroyRef` oder speichert die Subscription in einer Closure:

```typescript
withHooks((store) => {
  let subscription: Subscription;
  return {
    onInit() {
      const svc = inject(ProductService);
      subscription = interval(10_000).pipe(
        startWith(0),
        switchMap(() => svc.getProducts())
      ).subscribe(tapResponse({
        next: (products) => store.setProducts(products),
        error: (err: Error) => store.setError(err.message),
      }));
    },
    onDestroy() {
      subscription?.unsubscribe();
    }
  };
}),
```

</details>

## Beispiellösung

```typescript
// product.model.ts
export interface Product {
  id: number;
  name: string;
  price: number;
}

// product.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Product } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }
}

// product.store.ts
import { signalStore, withState, withMethods, withHooks, patchState } from '@ngrx/signals';
import { inject } from '@angular/core';
import { interval, Subscription, startWith, switchMap } from 'rxjs';
import { tapResponse } from '@ngrx/operators';
import { ProductService } from './product.service';
import { Product } from './product.model';

interface ProductState {
  products: Product[];
  loading: boolean;
  error: string | null;
}

export const ProductStore = signalStore(
  withState<ProductState>({ products: [], loading: false, error: null }),
  withMethods((store) => ({
    setLoading: (loading: boolean) => patchState(store, { loading }),
    setProducts: (products: Product[]) =>
      patchState(store, { products, loading: false, error: null }),
    setError: (error: string) =>
      patchState(store, { error, loading: false }),
  })),
  withHooks((store) => {
    let subscription: Subscription;

    return {
      onInit() {
        const productService = inject(ProductService);
        store.setLoading(true);

        subscription = interval(10_000)
          .pipe(
            startWith(0),
            switchMap(() => productService.getProducts())
          )
          .subscribe(
            tapResponse({
              next: (products) => store.setProducts(products),
              error: (err: Error) => store.setError(err.message),
            })
          );
      },
      onDestroy() {
        subscription?.unsubscribe();
      },
    };
  })
);

// product-list.component.ts
import { Component, inject } from '@angular/core';
import { ProductStore } from './product.store';

@Component({
  selector: 'app-product-list',
  standalone: true,
  providers: [ProductStore], // Component-level: Store lebt so lange wie die Komponente
  template: `
    @if (store.loading()) {
      <p>Lade Produkte...</p>
    } @else if (store.error()) {
      <p>Fehler: {{ store.error() }}</p>
    } @else {
      <ul>
        @for (product of store.products(); track product.id) {
          <li>{{ product.name }} – {{ product.price | currency }}</li>
        }
      </ul>
    }
  `,
})
export class ProductListComponent {
  protected store = inject(ProductStore);
}
```

## Weiterführendes

- `withHooks` lässt sich hervorragend mit `withCallState` aus `@ngrx/signals/entities` kombinieren, um Ladezustände noch deklarativer zu verwalten – schau dir `EntityCallState` im NgRx-Docs an.
- Für komplexere Initialisierungslogik mit sequentiellen HTTP-Aufrufen eignet sich `concatMap` statt `switchMap`, um sicherzustellen, dass vorige Requests abgeschlossen werden.
- [NgRx Signal Store Docs – withHooks](https://ngrx.io/guide/signals/signal-store/lifecycle-hooks)
