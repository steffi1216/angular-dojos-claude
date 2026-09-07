# Angular Dojo: NgRx Router Store
**Datum:** 2026-09-07
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du den Angular Router mit NgRx über `@ngrx/router-store` synchronisierst, damit Router-State (aktuelle URL, Route-Parameter, Query-Params) als Teil des globalen Store-Zustands zugänglich ist und in Selektoren und Effects genutzt werden kann.

## Hintergrund & Theorie
In NgRx-Applikationen lebt der gesamte Anwendungszustand im Store. Der Angular Router hat jedoch seinen eigenen, separaten State. `@ngrx/router-store` schlägt eine Brücke: Es serialisiert bei jedem Navigationsereignis den Router-State und schreibt ihn in den Store. Dadurch können:

- **Selektoren** direkt auf URL-Parameter, Query-Parameter und URL zugreifen.
- **Effects** auf Navigationsevents reagieren (z. B. Daten nachladen, wenn eine Route aktiviert wird).
- **Time-Travel-Debugging** mit Redux DevTools auch Navigationsvorgänge zeigen.

Standardmäßig speichert `@ngrx/router-store` den kompletten `RouterStateSnapshot`, was teuer ist. Mit einem **Custom Router State Serializer** kontrollierst du genau, welche Informationen serialisiert werden — nur das, was du wirklich brauchst.

Der Flow:
```
Navigation → RouterStore-Middleware → ROUTER_NAVIGATION Action → Reducer → Store → Selektoren
```

## Aufgabe
Baue eine kleine Angular-Applikation, die Produkte nach Kategorie filtert. Die aktive Kategorie steht als Route-Parameter in der URL (`/products/:categoryId`). Die Produktliste soll den `categoryId`-Parameter direkt aus dem NgRx Store lesen — ohne `ActivatedRoute` im Component zu injecten.

### Schritte

1. **Abhängigkeiten installieren** und `@ngrx/router-store` in der App registrieren (`provideRouterStore()`).

2. **Custom Serializer** implementieren: Erstelle einen `CustomRouterStateSerializer`, der aus dem `RouterStateSnapshot` nur `{ url, params, queryParams, fragment }` extrahiert und serialisiert. Registriere ihn via `routerReducer` und dem Token `ROUTER_STATE_SERIALIZER`.

3. **Feature-State und Selektoren** erstellen:
   - Nutze `getRouterSelectors()` aus `@ngrx/router-store`, um vorgefertigte Selektoren zu erhalten.
   - Erstelle einen eigenen Selektor `selectCategoryId`, der `selectRouteParam('categoryId')` verwendet.

4. **Effect** implementieren: Schreibe einen Effect `loadProductsByCategory$`, der auf `routerNavigatedAction` reagiert, den `selectCategoryId`-Selektor auswertet und (simuliert) Produkte für die Kategorie lädt.

5. **Component** verdrahten: Lies `categoryId` und die geladenen Produkte ausschließlich über Store-Selektoren. Zeige beides im Template an.

## Hints

<details>
<summary>Hint 1 – provideRouterStore und routerReducer</summary>

```typescript
// app.config.ts
import { provideStore } from '@ngrx/store';
import { provideRouterStore, routerReducer } from '@ngrx/router-store';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideStore({ router: routerReducer }),
    provideRouterStore(),
  ],
};
```

</details>

<details>
<summary>Hint 2 – Custom Serializer und getRouterSelectors</summary>

```typescript
// router-serializer.ts
import { RouterStateSnapshot } from '@angular/router';
import { RouterStateSerializer } from '@ngrx/router-store';

export interface MinimalRouterState {
  url: string;
  params: Record<string, string>;
  queryParams: Record<string, string>;
}

export class CustomRouterStateSerializer
  implements RouterStateSerializer<MinimalRouterState> {
  serialize(routerState: RouterStateSnapshot): MinimalRouterState {
    let route = routerState.root;
    while (route.firstChild) route = route.firstChild;
    return {
      url: routerState.url,
      params: route.params as Record<string, string>,
      queryParams: routerState.root.queryParams as Record<string, string>,
    };
  }
}

// Registrierung:
provideRouterStore({ serializer: CustomRouterStateSerializer })

// router.selectors.ts
import { getRouterSelectors } from '@ngrx/router-store';
export const {
  selectCurrentRoute,
  selectRouteParam,
  selectQueryParam,
  selectUrl,
} = getRouterSelectors();

export const selectCategoryId = selectRouteParam('categoryId');
```

</details>

<details>
<summary>Hint 3 – Effect mit routerNavigatedAction</summary>

```typescript
// products.effects.ts
import { inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { routerNavigatedAction } from '@ngrx/router-store';
import { Store } from '@ngrx/store';
import { switchMap, withLatestFrom, filter, map } from 'rxjs/operators';
import { selectCategoryId } from './router.selectors';
import { loadProductsSuccess } from './products.actions';

export const loadProductsByCategory$ = createEffect(
  (actions$ = inject(Actions), store = inject(Store)) =>
    actions$.pipe(
      ofType(routerNavigatedAction),
      withLatestFrom(store.select(selectCategoryId)),
      filter(([, categoryId]) => !!categoryId),
      switchMap(([, categoryId]) => {
        // Simulation: echte HTTP-Anfrage hier
        const products = [
          { id: 1, name: `Product A (${categoryId})` },
          { id: 2, name: `Product B (${categoryId})` },
        ];
        return [loadProductsSuccess({ products })];
      })
    ),
  { functional: true }
);
```

</details>

## Beispiellösung

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { provideRouterStore, routerReducer } from '@ngrx/router-store';
import { routes } from './app.routes';
import { CustomRouterStateSerializer } from './router-serializer';
import { productsReducer } from './products.reducer';
import * as productEffects from './products.effects';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideStore({ router: routerReducer, products: productsReducer }),
    provideEffects(productEffects),
    provideRouterStore({ serializer: CustomRouterStateSerializer }),
  ],
};

// router.selectors.ts
import { createFeatureSelector } from '@ngrx/store';
import { getRouterSelectors } from '@ngrx/router-store';
import { MinimalRouterState } from './router-serializer';

export const selectRouter = createFeatureSelector<MinimalRouterState>('router');

export const {
  selectUrl,
  selectRouteParam,
  selectQueryParam,
} = getRouterSelectors(selectRouter);

export const selectCategoryId = selectRouteParam('categoryId');

// products.component.ts
import { Component, inject } from '@angular/core';
import { Store } from '@ngrx/store';
import { AsyncPipe, NgFor } from '@angular/common';
import { selectCategoryId } from './router.selectors';
import { selectAllProducts } from './products.selectors';

@Component({
  selector: 'app-products',
  standalone: true,
  imports: [AsyncPipe, NgFor],
  template: `
    <h2>Kategorie: {{ categoryId$ | async }}</h2>
    <ul>
      <li *ngFor="let p of products$ | async">{{ p.name }}</li>
    </ul>
  `,
})
export class ProductsComponent {
  private store = inject(Store);
  categoryId$ = this.store.select(selectCategoryId);
  products$ = this.store.select(selectAllProducts);
}

// app.routes.ts
import { Routes } from '@angular/router';
import { ProductsComponent } from './products.component';

export const routes: Routes = [
  { path: 'products/:categoryId', component: ProductsComponent },
  { path: '', redirectTo: 'products/electronics', pathMatch: 'full' },
];
```

## Weiterführendes
- **NgRx DevTools + Router Store**: In den Redux DevTools siehst du jeden `ROUTER_NAVIGATION`-Dispatch mit dem serialisierten State — ideal für Time-Travel-Debugging von Navigationen.
- **Tipp**: Kombiniere `routerNavigatedAction` mit `EMPTY` (aus RxJS) in Effects, wenn du nur auf bestimmte Routes reagieren willst — prüfe `payload.event.url` und gib `EMPTY` zurück, wenn die Route nicht passt.
- **Offiziell**: [NgRx Router Store Docs](https://ngrx.io/guide/router-store)
