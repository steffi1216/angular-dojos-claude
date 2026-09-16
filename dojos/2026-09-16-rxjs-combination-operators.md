# Angular Dojo: RxJS Combination Operators – Parallele Datenströme kombinieren
**Datum:** 2026-09-16
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wann und wie du `combineLatest`, `zip` und `forkJoin` einsetzt, um mehrere Observables parallel zu kombinieren – und verstehst die entscheidenden Unterschiede zwischen den drei Operatoren.

## Hintergrund & Theorie

In realen Angular-Anwendungen musst du häufig mehrere HTTP-Anfragen gleichzeitig abschicken und deren Ergebnisse zusammenführen. Dafür bietet RxJS drei Kombinations-Operatoren:

**`forkJoin([obs1, obs2])`**
Wartet, bis *alle* übergebenen Observables abgeschlossen sind, und gibt einmalig ein Array mit den letzten Werten aus. Ideal für HTTP-Requests, die abgeschlossen werden. Bricht ab, wenn eines der Observables einen Fehler wirft.

**`zip([obs1, obs2])`**
Paart Werte nach Index: gibt `[val1_1, val2_1]` aus, dann `[val1_2, val2_2]` usw. Nützlich, wenn die Reihenfolge der Emissionen semantisch bedeutsam ist. Schließt ab, wenn das kürzeste Observable fertig ist.

**`combineLatest([obs1, obs2])`**
Emittiert ein neues Array, sobald *irgendein* Observable einen neuen Wert liefert – immer mit dem jeweils aktuellsten Wert der anderen Quellen. Perfekt für reaktive Dashboards, die auf mehrere sich ändernde Datenquellen reagieren (z. B. Filterkriterien + Suchergebnis + Nutzerprofil). Startet erst, wenn *alle* Quellen mindestens einmal geemittet haben.

| Operator | Abschluss nötig? | Reagiert auf Updates? | Typischer Einsatz |
|---|---|---|---|
| `forkJoin` | Ja | Nein | HTTP-Requests parallel laden |
| `zip` | Ja | Nein | Werte paarweise kombinieren |
| `combineLatest` | Nein | Ja | Reaktive UI (Filter, Streams) |

## Aufgabe

Erstelle eine `DashboardComponent`, die ein Dashboard für einen Online-Shop simuliert. Das Dashboard soll:

1. Gleichzeitig Nutzer-Daten, Produkt-Daten und Bestellungen vom (simulierten) Backend laden (`forkJoin`).
2. Live auf Suchterm *und* Kategorie-Filter reagieren und die gefilterten Produkte anzeigen (`combineLatest`).
3. Einen Ladezustand und Fehlerbehandlung einbauen.

### Schritte

1. **Services erstellen** – Erstelle einen `ShopService` mit drei Methoden, die Observables zurückgeben (simuliere mit `of(...)` + `delay()`):
   - `getUser(): Observable<User>`
   - `getProducts(): Observable<Product[]>`
   - `getOrders(): Observable<Order[]>`

2. **Initialdaten mit `forkJoin` laden** – Kombiniere alle drei Requests in `ngOnInit` mit `forkJoin` und weise die Ergebnisse Signals zu.

3. **Reaktiven Filter mit `combineLatest` bauen** – Erstelle zwei `BehaviorSubject`s (Suchterm + Kategorie). Kombiniere sie mit `combineLatest` und der Produkt-Liste und berechne die gefilterten Produkte als Signal (`toSignal`).

4. **Template bauen** – Zeige Ladezustand, Fehler, Nutzername, Bestellanzahl und die gefilterte Produktliste an.

## Hints

<details>
<summary>Hint 1 – forkJoin Struktur</summary>

```typescript
forkJoin({
  user: this.shopService.getUser(),
  products: this.shopService.getProducts(),
  orders: this.shopService.getOrders(),
}).subscribe({
  next: ({ user, products, orders }) => { /* ... */ },
  error: (err) => { /* Fehlerbehandlung */ },
});
```

`forkJoin` akzeptiert sowohl ein Array als auch ein Objekt – mit einem Objekt sind die Keys im Ergebnis direkt benannt.

</details>

<details>
<summary>Hint 2 – combineLatest mit toSignal</summary>

```typescript
private searchTerm$ = new BehaviorSubject<string>('');
private category$ = new BehaviorSubject<string>('all');
private products$ = new BehaviorSubject<Product[]>([]);

filteredProducts = toSignal(
  combineLatest([this.products$, this.searchTerm$, this.category$]).pipe(
    map(([products, term, cat]) =>
      products
        .filter(p => cat === 'all' || p.category === cat)
        .filter(p => p.name.toLowerCase().includes(term.toLowerCase()))
    )
  ),
  { initialValue: [] }
);
```

</details>

<details>
<summary>Hint 3 – Ladezustand und Fehler mit Signals</summary>

```typescript
isLoading = signal(true);
error = signal<string | null>(null);

ngOnInit() {
  forkJoin({ ... }).subscribe({
    next: (data) => {
      this.isLoading.set(false);
      // Daten setzen
    },
    error: (err) => {
      this.isLoading.set(false);
      this.error.set('Daten konnten nicht geladen werden.');
    },
  });
}
```

</details>

## Beispiellösung

```typescript
// shop.service.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay } from 'rxjs/operators';

export interface User { id: number; name: string; }
export interface Product { id: number; name: string; category: string; price: number; }
export interface Order { id: number; total: number; }

@Injectable({ providedIn: 'root' })
export class ShopService {
  getUser(): Observable<User> {
    return of({ id: 1, name: 'Maria Muster' }).pipe(delay(300));
  }

  getProducts(): Observable<Product[]> {
    return of([
      { id: 1, name: 'Laptop', category: 'electronics', price: 999 },
      { id: 2, name: 'T-Shirt', category: 'clothing', price: 25 },
      { id: 3, name: 'Kopfhörer', category: 'electronics', price: 199 },
      { id: 4, name: 'Jeans', category: 'clothing', price: 79 },
    ]).pipe(delay(500));
  }

  getOrders(): Observable<Order[]> {
    return of([{ id: 1, total: 1024 }, { id: 2, total: 104 }]).pipe(delay(400));
  }
}

// dashboard.component.ts
import { Component, inject, OnInit, signal } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { BehaviorSubject, combineLatest, forkJoin } from 'rxjs';
import { map } from 'rxjs/operators';
import { ShopService, Product, User, Order } from './shop.service';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [/* FormsModule, CommonModule, ... */],
  template: `
    @if (isLoading()) {
      <p>Lade Dashboard...</p>
    } @else if (error()) {
      <p class="error">{{ error() }}</p>
    } @else {
      <h2>Willkommen, {{ user()?.name }}</h2>
      <p>Bestellungen: {{ orders()?.length }}</p>

      <input
        placeholder="Suche..."
        (input)="searchTerm$.next($any($event.target).value)"
      />
      <select (change)="category$.next($any($event.target).value)">
        <option value="all">Alle</option>
        <option value="electronics">Elektronik</option>
        <option value="clothing">Kleidung</option>
      </select>

      <ul>
        @for (p of filteredProducts(); track p.id) {
          <li>{{ p.name }} – {{ p.price | currency:'EUR' }}</li>
        }
      </ul>
    }
  `,
})
export class DashboardComponent implements OnInit {
  private shopService = inject(ShopService);

  isLoading = signal(true);
  error = signal<string | null>(null);
  user = signal<User | null>(null);
  orders = signal<Order[] | null>(null);

  private products$ = new BehaviorSubject<Product[]>([]);
  searchTerm$ = new BehaviorSubject<string>('');
  category$ = new BehaviorSubject<string>('all');

  filteredProducts = toSignal(
    combineLatest([this.products$, this.searchTerm$, this.category$]).pipe(
      map(([products, term, cat]) =>
        products
          .filter(p => cat === 'all' || p.category === cat)
          .filter(p => p.name.toLowerCase().includes(term.toLowerCase()))
      )
    ),
    { initialValue: [] as Product[] }
  );

  ngOnInit() {
    forkJoin({
      user: this.shopService.getUser(),
      products: this.shopService.getProducts(),
      orders: this.shopService.getOrders(),
    }).subscribe({
      next: ({ user, products, orders }) => {
        this.user.set(user);
        this.orders.set(orders);
        this.products$.next(products);
        this.isLoading.set(false);
      },
      error: () => {
        this.error.set('Daten konnten nicht geladen werden.');
        this.isLoading.set(false);
      },
    });
  }
}
```

## Weiterführendes

- **`combineLatestWith` (pipeable):** Statt `combineLatest([a, b])` kannst du auch `a.pipe(combineLatestWith(b))` schreiben – besser lesbar in langen Pipe-Ketten.
- **`race`:** Ein weiterer Kombinationsoperator – gibt den Wert des *ersten* Observables aus, das emittiert. Nützlich für Timeout-Logik.
- **Fehlerbehandlung in `forkJoin`:** Wenn ein einzelner Request fehlschlägt, bricht `forkJoin` komplett ab. Mit `catchError` am einzelnen Observable + Fallback-Wert (`of(null)`) kannst du partielle Fehler tolerieren.
- [RxJS-Doku: combineLatest](https://rxjs.dev/api/index/function/combineLatest)
- [RxJS Marbles (Visualisierung)](https://rxmarbles.com/)
