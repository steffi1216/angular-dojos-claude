# Angular Dojo: RxJS `scan` für State-Management
**Datum:** 2026-10-17
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem RxJS-Operator `scan` zustandsbehaftete Streams aufbaust und damit lokales State-Management direkt in einem Angular-Service implementierst – ohne NgRx oder externe Bibliotheken.

## Hintergrund & Theorie

`scan` ist das reaktive Gegenstück zu `Array.reduce`: Er akkumuliert Werte über die Zeit und gibt nach jedem emittierten Event den aktuellen Akkumulator zurück. Das Muster `Subject + scan + shareReplay` ist die Grundlage für schlankes, reaktives State-Management in Angular-Services.

```
Action-Stream (Subject)
    │
    ▼
  scan((state, action) => newState, initialState)
    │
    ▼
  shareReplay(1)  ← aktuellen State für späte Subscriber cachen
    │
    ▼
  state$ (Observable)
```

**Typische Anwendungsfälle:**
- Einkaufswagen (Artikel hinzufügen/entfernen, Mengen ändern)
- Infinite-Scroll-Listen (neue Seiten akkumulieren)
- Undo/Redo-Stacks
- Optimistische UI-Updates (lokal vorab anwenden, bei Fehler zurückrollen)
- Ereigniszähler (Klicks, Benachrichtigungen)

`scan` unterscheidet sich von `reduce` darin, dass es **nicht** auf den Abschluss des Streams wartet – jede Emission löst eine neue Ausgabe aus.

## Aufgabe

Baue einen `CartService` für einen Webshop, der den Warenkorb-Zustand rein reaktiv mit `scan` verwaltet. Die Komponente soll Artikel hinzufügen, entfernen und die Menge ändern können, ohne direkten Zugriff auf den inneren State.

### Schritte

1. **State & Action-Typen definieren**
   Definiere ein `CartItem`-Interface (`id`, `name`, `price`, `quantity`) und einen discriminated-union-Typ `CartAction` mit den Varianten `add`, `remove` und `updateQuantity`.

2. **`CartService` mit `scan` implementieren**
   Erstelle einen `Subject<CartAction>` als Aktions-Stream. Leite daraus mit `scan` das `state$`-Observable ab, das ein `CartItem[]`-Array akkumuliert. Gib den State über `shareReplay(1)` weiter und leite `total$` als `computed` (d.h. per `map`) daraus ab.

3. **Komponente verdrahten**
   Erstelle eine `CartComponent`, die den `CartService` injiziert, mit der `async`-Pipe den aktuellen State anzeigt und Buttons zum Hinzufügen/Entfernen von Testartikeln bereitstellt.

4. **Gesamt-Preis reaktiv berechnen**
   Leite `total$` als separates Observable ab, das automatisch aktualisiert wird, wenn sich der Warenkorb ändert.

## Hints

<details>
<summary>Hint 1 – Grundstruktur des Services</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class CartService {
  private actions$ = new Subject<CartAction>();

  readonly state$: Observable<CartItem[]> = this.actions$.pipe(
    scan((items, action) => this.reduce(items, action), [] as CartItem[]),
    shareReplay(1)
  );

  readonly total$ = this.state$.pipe(
    map(items => items.reduce((sum, i) => sum + i.price * i.quantity, 0))
  );

  dispatch(action: CartAction) {
    this.actions$.next(action);
  }

  private reduce(items: CartItem[], action: CartAction): CartItem[] {
    switch (action.type) {
      case 'add': { /* ... */ }
      case 'remove': { /* ... */ }
      case 'updateQuantity': { /* ... */ }
    }
  }
}
```

</details>

<details>
<summary>Hint 2 – Reducer-Logik für die drei Actions</summary>

```typescript
case 'add': {
  const existing = items.find(i => i.id === action.item.id);
  if (existing) {
    return items.map(i =>
      i.id === action.item.id ? { ...i, quantity: i.quantity + 1 } : i
    );
  }
  return [...items, { ...action.item, quantity: 1 }];
}
case 'remove':
  return items.filter(i => i.id !== action.id);
case 'updateQuantity':
  return items.map(i =>
    i.id === action.id ? { ...i, quantity: Math.max(0, action.quantity) } : i
  ).filter(i => i.quantity > 0);
```

</details>

## Beispiellösung

```typescript
// models/cart.model.ts
export interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

export type CartAction =
  | { type: 'add'; item: Omit<CartItem, 'quantity'> }
  | { type: 'remove'; id: string }
  | { type: 'updateQuantity'; id: string; quantity: number };

// services/cart.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable } from 'rxjs';
import { scan, shareReplay, map } from 'rxjs/operators';
import { CartItem, CartAction } from '../models/cart.model';

@Injectable({ providedIn: 'root' })
export class CartService {
  private actions$ = new Subject<CartAction>();

  readonly state$: Observable<CartItem[]> = this.actions$.pipe(
    scan((items, action) => this.reduce(items, action), [] as CartItem[]),
    shareReplay(1)
  );

  readonly total$: Observable<number> = this.state$.pipe(
    map(items => items.reduce((sum, i) => sum + i.price * i.quantity, 0))
  );

  readonly count$: Observable<number> = this.state$.pipe(
    map(items => items.reduce((sum, i) => sum + i.quantity, 0))
  );

  dispatch(action: CartAction): void {
    this.actions$.next(action);
  }

  private reduce(items: CartItem[], action: CartAction): CartItem[] {
    switch (action.type) {
      case 'add': {
        const existing = items.find(i => i.id === action.item.id);
        if (existing) {
          return items.map(i =>
            i.id === action.item.id ? { ...i, quantity: i.quantity + 1 } : i
          );
        }
        return [...items, { ...action.item, quantity: 1 }];
      }
      case 'remove':
        return items.filter(i => i.id !== action.id);
      case 'updateQuantity':
        return items
          .map(i =>
            i.id === action.id ? { ...i, quantity: Math.max(0, action.quantity) } : i
          )
          .filter(i => i.quantity > 0);
    }
  }
}

// components/cart.component.ts
import { Component, inject } from '@angular/core';
import { AsyncPipe, CurrencyPipe } from '@angular/common';
import { CartService } from '../services/cart.service';

@Component({
  selector: 'app-cart',
  standalone: true,
  imports: [AsyncPipe, CurrencyPipe],
  template: `
    <h2>Warenkorb ({{ cart.count$ | async }} Artikel)</h2>

    @for (item of cart.state$ | async; track item.id) {
      <div>
        <span>{{ item.name }} – {{ item.price | currency:'EUR' }}</span>
        <button (click)="decrement(item.id)">−</button>
        <span>{{ item.quantity }}</span>
        <button (click)="increment(item.id, item)">+</button>
        <button (click)="remove(item.id)">✕</button>
      </div>
    }

    <p><strong>Gesamt: {{ cart.total$ | async | currency:'EUR' }}</strong></p>

    <button (click)="addSample()">Testartikel hinzufügen</button>
  `
})
export class CartComponent {
  cart = inject(CartService);

  addSample(): void {
    const items = [
      { id: 'p1', name: 'Tastatur', price: 89.99 },
      { id: 'p2', name: 'Maus', price: 49.99 },
      { id: 'p3', name: 'Monitor', price: 299.00 },
    ];
    const item = items[Math.floor(Math.random() * items.length)];
    this.cart.dispatch({ type: 'add', item });
  }

  increment(id: string, item: { quantity: number }): void {
    this.cart.dispatch({ type: 'updateQuantity', id, quantity: item.quantity + 1 });
  }

  decrement(id: string, quantity: number): void {
    this.cart.dispatch({ type: 'updateQuantity', id, quantity: quantity - 1 });
  }

  remove(id: string): void {
    this.cart.dispatch({ type: 'remove', id });
  }
}
```

## Weiterführendes

- **Undo/Redo**: Statt nur den aktuellen State zu speichern, akkumuliere mit `scan` eine History (`{ past, present, future }`) und implementiere `undo`/`redo`-Actions.
- **Persistenz**: Kombiniere `state$` mit `tap(state => localStorage.setItem('cart', JSON.stringify(state)))` und lade den initialen State aus `localStorage` als zweiten Parameter von `scan`.
- **Mit Signals kombinieren**: Übergib `state$` an `toSignal(this.cart.state$, { initialValue: [] })` im Component, um den State als Signal zu nutzen und `async`-Pipe zu vermeiden.
- [RxJS scan – offizielle Dokumentation](https://rxjs.dev/api/operators/scan)
