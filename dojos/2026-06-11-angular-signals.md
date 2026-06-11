# Angular Dojo: Angular Signals
**Datum:** 2026-06-11
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst das neue Reaktivitätsmodell von Angular Signals und kannst `signal()`, `computed()` und `effect()` gezielt einsetzen, um feingranulare, performante Zustandsverwaltung ohne Zone.js-Abhängigkeit zu implementieren.

## Hintergrund & Theorie
Angular Signals (ab Angular 16, stabil ab 17) sind ein synchrones, push-basiertes Reaktivitätssystem. Im Gegensatz zu RxJS-Observables sind Signals immer synchron lesbar und tragen ihren aktuellen Wert. Angular kann damit Change Detection feingranular auslösen – nur die Komponenten, die ein Signal tatsächlich lesen, werden neu gerendert.

**Kernkonzepte:**
- `signal(initialValue)` – erstellt ein beschreibbares Signal
- `.set(value)` / `.update(fn)` / `.mutate(fn)` – Wert ändern
- `computed(() => ...)` – abgeleiteter, nur-lesbarer Wert; wird lazy und memoized berechnet
- `effect(() => ...)` – Seiteneffekt, der automatisch ausgeführt wird, wenn abhängige Signals sich ändern
- `toSignal(observable$)` / `toObservable(signal)` – Interop mit RxJS

Signals arbeiten ohne `Zone.js` und sind der Baustein für die kommende *zoneless* Angular-Architektur. Mit `OnPush`-Komponenten und Signals erreicht man maximale Render-Effizienz.

## Aufgabe
Baue eine kleine **Warenkorb-Komponente** mit Angular Signals:

- Eine Liste von Produkten (`Product[]`) wird als Signal gehalten
- Der Gesamtpreis wird als `computed()` Signal abgeleitet
- Ein `effect()` speichert den Warenkorb automatisch in `localStorage`, sobald er sich ändert
- Produkte können hinzugefügt und entfernt werden

### Schritte
1. Erstelle ein `CartComponent` als Standalone Component mit `signal<Product[]>([])`
2. Implementiere `addProduct(product: Product)` mit `.update()`
3. Implementiere `removeProduct(id: number)` mit `.update()` und `filter()`
4. Erstelle ein `computed()` Signal `totalPrice` das die Summe aller `price * quantity` berechnet
5. Füge einen `effect()` hinzu, der bei jeder Änderung `localStorage.setItem('cart', JSON.stringify(...))` aufruft
6. Lade im Konstruktor vorhandene Daten aus `localStorage` als Initialwert
7. Zeige im Template die Produktliste und den `totalPrice()` an

## Hints

<details>
<summary>Hint 1 – Signal mit Initialwert aus localStorage</summary>

```typescript
const saved = localStorage.getItem('cart');
const initial: Product[] = saved ? JSON.parse(saved) : [];
products = signal<Product[]>(initial);
```

</details>

<details>
<summary>Hint 2 – computed() und effect() im Zusammenspiel</summary>

```typescript
totalPrice = computed(() =>
  this.products().reduce((sum, p) => sum + p.price * p.quantity, 0)
);

constructor() {
  effect(() => {
    // Wird automatisch neu ausgeführt, wenn products() sich ändert
    localStorage.setItem('cart', JSON.stringify(this.products()));
  });
}
```

</details>

<details>
<summary>Hint 3 – update() für unveränderliche Zustandsänderungen</summary>

```typescript
addProduct(product: Product) {
  this.products.update(current => {
    const existing = current.find(p => p.id === product.id);
    if (existing) {
      return current.map(p =>
        p.id === product.id ? { ...p, quantity: p.quantity + 1 } : p
      );
    }
    return [...current, { ...product, quantity: 1 }];
  });
}
```

</details>

## Beispiellösung

```typescript
import { Component, computed, effect, signal } from '@angular/core';
import { CommonModule } from '@angular/common';

interface Product {
  id: number;
  name: string;
  price: number;
  quantity: number;
}

const SAMPLE_PRODUCTS: Omit<Product, 'quantity'>[] = [
  { id: 1, name: 'Angular-Buch', price: 39.99 },
  { id: 2, name: 'RxJS-Kurs', price: 29.99 },
  { id: 3, name: 'TypeScript-Workshop', price: 49.99 },
];

@Component({
  selector: 'app-cart',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>Warenkorb</h2>

    <div class="available-products">
      <h3>Verfügbare Produkte</h3>
      @for (product of availableProducts; track product.id) {
        <button (click)="addProduct(product)">
          {{ product.name }} – {{ product.price | currency:'EUR' }}
        </button>
      }
    </div>

    <div class="cart-items">
      <h3>Im Warenkorb ({{ products().length }} Artikel)</h3>
      @for (item of products(); track item.id) {
        <div class="cart-item">
          <span>{{ item.name }} × {{ item.quantity }}</span>
          <span>{{ item.price * item.quantity | currency:'EUR' }}</span>
          <button (click)="removeProduct(item.id)">Entfernen</button>
        </div>
      } @empty {
        <p>Der Warenkorb ist leer.</p>
      }
    </div>

    <div class="total">
      <strong>Gesamt: {{ totalPrice() | currency:'EUR' }}</strong>
    </div>

    <button (click)="clearCart()">Warenkorb leeren</button>
  `,
})
export class CartComponent {
  availableProducts = SAMPLE_PRODUCTS;

  products = signal<Product[]>(this.loadFromStorage());

  totalPrice = computed(() =>
    this.products().reduce((sum, p) => sum + p.price * p.quantity, 0)
  );

  constructor() {
    effect(() => {
      localStorage.setItem('cart', JSON.stringify(this.products()));
    });
  }

  addProduct(product: Omit<Product, 'quantity'>) {
    this.products.update(current => {
      const existing = current.find(p => p.id === product.id);
      if (existing) {
        return current.map(p =>
          p.id === product.id ? { ...p, quantity: p.quantity + 1 } : p
        );
      }
      return [...current, { ...product, quantity: 1 }];
    });
  }

  removeProduct(id: number) {
    this.products.update(current => current.filter(p => p.id !== id));
  }

  clearCart() {
    this.products.set([]);
  }

  private loadFromStorage(): Product[] {
    try {
      const saved = localStorage.getItem('cart');
      return saved ? JSON.parse(saved) : [];
    } catch {
      return [];
    }
  }
}
```

## Weiterführendes
- **Signal-basierte Inputs/Outputs (Angular 17.1+):** `input()`, `output()`, `model()` als Signal-Varianten von `@Input()`/`@Output()` – ermöglichen vollständig reaktive Komponenten ohne Decorators
- **Ressource:** [Angular Signals Guide (angular.dev)](https://angular.dev/guide/signals)
- **Nächster Schritt:** Kombiniere Signals mit `toSignal(httpClient.get(...))` für asynchrone Datenladelogik ohne manuelle Subscriptions
- **Zoneless Angular:** Experimentiere mit `provideExperimentalZonelessChangeDetection()` in `app.config.ts` – Signals machen dies möglich
