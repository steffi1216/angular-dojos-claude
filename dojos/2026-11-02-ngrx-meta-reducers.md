# Angular Dojo: NgRx Meta-Reducers
**Datum:** 2026-11-02
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Verstehe, was NgRx Meta-Reducers sind und wie du damit querschnittliche Belange (Logging, State-Persistence, Undo/Redo) auf alle Reducers einer Applikation anwenden kannst, ohne den Business-Code zu verändern.

## Hintergrund & Theorie

Ein **Meta-Reducer** ist eine Funktion höherer Ordnung, die einen Reducer empfängt und einen neuen Reducer zurückgibt. Er „umhüllt" alle Reducers der Applikation – ähnlich wie ein HTTP-Interceptor alle HTTP-Requests abfängt.

```
MetaReducer: (reducer: ActionReducer<State>) => ActionReducer<State>
```

Dadurch kannst du Logik ausführen **bevor** und/oder **nachher** jeder Action verarbeitet wird – für alle Actions gleichzeitig, ohne jeden Reducer anzufassen.

Typische Anwendungsfälle:
- **Logging**: Jede Action und den daraus resultierenden State loggen
- **Undo/Redo**: State-Snapshots sammeln und navigieren
- **State Hydration**: State beim App-Start aus dem `localStorage` wiederherstellen
- **Reset**: Einen speziellen "Reset"-Action-Typ global behandeln
- **Encryption/Serialization**: State vor dem Speichern transformieren

Meta-Reducers werden in der Reihenfolge des Arrays ausgeführt (outer-to-inner). Beim Konfigurieren mit `provideStore()` übergibt man ein `metaReducers`-Array im Config-Objekt.

**Wichtig:** Meta-Reducers laufen bei *jeder* Action. Performance-intensive Logik (z. B. deep cloning) sollte nur im Dev-Mode aktiv sein.

## Aufgabe

Erstelle zwei Meta-Reducers für eine Shopping-Cart-Applikation:

1. **`loggerMetaReducer`** (nur Dev-Mode): Loggt jede Action mit ihrem Typ sowie den State davor und danach in die Konsole.
2. **`localStorageMetaReducer`**: Persistiert den gesamten State im `localStorage` und lädt ihn beim App-Start als initialen State (Hydration).

Verdrahte beide Meta-Reducers in der App-Konfiguration so, dass `loggerMetaReducer` nur im Entwicklungsmodus aktiv ist.

### Schritte

1. Erstelle das grundlegende Store-Setup mit `provideStore()` für einen minimalen `cartReducer` (Items hinzufügen/entfernen).
2. Implementiere `loggerMetaReducer`: Empfange den `reducer`, logge vor und nach dem Reducer-Aufruf und gib den neuen State zurück.
3. Implementiere `localStorageMetaReducer`: Lade beim ersten Aufruf (wenn `action.type === '@ngrx/store/init'`) den gespeicherten State aus `localStorage` und merge ihn mit dem initialen State. Speichere nach jeder Action den aktuellen State.
4. Registriere beide Meta-Reducers in `app.config.ts`: Logger nur wenn `!environment.production`, Storage immer.
5. Teste das Verhalten: Füge Artikel hinzu, lade die Seite neu – der State sollte wiederhergestellt sein.

## Hints

<details>
<summary>Hint 1 – Grundstruktur eines Meta-Reducers</summary>

```typescript
import { ActionReducer, MetaReducer } from '@ngrx/store';

export function loggerMetaReducer(
  reducer: ActionReducer<AppState>
): ActionReducer<AppState> {
  return function (state, action) {
    console.log('action:', action.type);
    console.log('state before:', state);
    const nextState = reducer(state, action);
    console.log('state after:', nextState);
    return nextState;
  };
}
```

</details>

<details>
<summary>Hint 2 – State aus localStorage hydratisieren</summary>

```typescript
const STORAGE_KEY = 'app_state';

export function localStorageMetaReducer(
  reducer: ActionReducer<AppState>
): ActionReducer<AppState> {
  return function (state, action) {
    // Beim Init: gespeicherten State laden
    if (action.type === '@ngrx/store/init') {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) {
        try {
          const savedState = JSON.parse(saved);
          // merge: gespeicherter State überschreibt initialen State
          state = { ...state, ...savedState };
        } catch {
          // Korrupte Daten ignorieren
        }
      }
    }
    const nextState = reducer(state, action);
    localStorage.setItem(STORAGE_KEY, JSON.stringify(nextState));
    return nextState;
  };
}
```

</details>

<details>
<summary>Hint 3 – Registrierung in app.config.ts</summary>

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideStore, MetaReducer } from '@ngrx/store';
import { environment } from '../environments/environment';
import { cartReducer } from './store/cart.reducer';
import { loggerMetaReducer, localStorageMetaReducer } from './store/meta-reducers';

const metaReducers: MetaReducer[] = [
  ...(!environment.production ? [loggerMetaReducer] : []),
  localStorageMetaReducer,
];

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({ cart: cartReducer }, { metaReducers }),
  ],
};
```

</details>

## Beispiellösung

```typescript
// store/cart.reducer.ts
import { createReducer, on, createAction, props } from '@ngrx/store';

export interface CartItem { id: string; name: string; quantity: number; }
export interface CartState { items: CartItem[]; }
const initialState: CartState = { items: [] };

export const addItem = createAction('[Cart] Add Item', props<{ item: CartItem }>());
export const removeItem = createAction('[Cart] Remove Item', props<{ id: string }>());
export const clearCart = createAction('[Cart] Clear');

export const cartReducer = createReducer(
  initialState,
  on(addItem, (state, { item }) => ({
    items: [...state.items, item],
  })),
  on(removeItem, (state, { id }) => ({
    items: state.items.filter(i => i.id !== id),
  })),
  on(clearCart, () => initialState)
);

// store/meta-reducers.ts
import { ActionReducer, MetaReducer } from '@ngrx/store';

export interface AppState { cart: import('./cart.reducer').CartState; }

const STORAGE_KEY = 'ngrx_app_state';

export function loggerMetaReducer(
  reducer: ActionReducer<AppState>
): ActionReducer<AppState> {
  return function (state, action) {
    console.group(`%c[NgRx] ${action.type}`, 'color: #6c5ce7; font-weight: bold');
    console.log('%cprev state', 'color: #999', state);
    const next = reducer(state, action);
    console.log('%cnext state', 'color: #4caf50', next);
    console.groupEnd();
    return next;
  };
}

export function localStorageMetaReducer(
  reducer: ActionReducer<AppState>
): ActionReducer<AppState> {
  return function (state, action) {
    if (action.type === '@ngrx/store/init') {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) {
        try {
          const persisted = JSON.parse(raw) as Partial<AppState>;
          state = { ...state, ...persisted };
        } catch { /* ignorieren */ }
      }
    }
    const nextState = reducer(state, action);
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(nextState));
    } catch { /* QuotaExceededError ignorieren */ }
    return nextState;
  };
}

export const metaReducers: MetaReducer<AppState>[] = [localStorageMetaReducer];

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideStore } from '@ngrx/store';
import { isDevMode } from '@angular/core';
import { cartReducer } from './store/cart.reducer';
import { loggerMetaReducer, localStorageMetaReducer, metaReducers } from './store/meta-reducers';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(
      { cart: cartReducer },
      {
        metaReducers: [
          ...(isDevMode() ? [loggerMetaReducer] : []),
          localStorageMetaReducer,
        ],
      }
    ),
  ],
};

// cart.component.ts (Beispiel-Komponente)
import { Component, inject } from '@angular/core';
import { Store } from '@ngrx/store';
import { selectCartItems } from './store/cart.selectors';
import { addItem, removeItem, clearCart } from './store/cart.reducer';

@Component({
  selector: 'app-cart',
  template: `
    <h2>Warenkorb ({{ items().length }} Artikel)</h2>
    <ul>
      @for (item of items(); track item.id) {
        <li>
          {{ item.name }} × {{ item.quantity }}
          <button (click)="remove(item.id)">✕</button>
        </li>
      }
    </ul>
    <button (click)="addTestItem()">Artikel hinzufügen</button>
    <button (click)="clear()">Leeren</button>
    <p><em>State überlebt Page-Reload dank localStorage Meta-Reducer!</em></p>
  `,
  standalone: true,
})
export class CartComponent {
  private store = inject(Store);
  items = this.store.selectSignal(selectCartItems);

  addTestItem(): void {
    this.store.dispatch(addItem({
      item: { id: crypto.randomUUID(), name: 'Produkt', quantity: 1 }
    }));
  }

  remove(id: string): void {
    this.store.dispatch(removeItem({ id }));
  }

  clear(): void {
    this.store.dispatch(clearCart());
  }
}
```

## Weiterführendes

- **Undo/Redo mit Meta-Reducers**: Halte ein Array von State-Snapshots im Meta-Reducer vor und dispatche `undo`/`redo`-Actions – der Meta-Reducer navigiert durch die History, ohne dass ein Reducer das wissen muss.
- **Serialization Check (Dev-only)**: NgRx bietet `@ngrx/store/src/meta-reducers/serialization_checker.js` als Vorbild für einen Meta-Reducer, der nicht-serialisierbare Werte im State/Actions aufspürt.
- **Partielle Persistence**: Speichere nicht den gesamten State, sondern nur bestimmte Feature-Slices (z. B. User-Preferences), indem du im Meta-Reducer selektiv serialisierst.
- Offizielle Doku: [ngrx.io/guide/store/metareducers](https://ngrx.io/guide/store/metareducers)
