# Angular Dojo: OnPush + Signals – Automatische Reaktivität
**Datum:** 2026-09-23
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Verstehen, wie Signal-basierte Inputs und `computed()` OnPush-Komponenten automatisch aktualisieren – ohne `ChangeDetectorRef.markForCheck()` oder `async`-Pipe – und wie `toSignal` mit reaktivem `switchMap` bei Input-Wechsel richtig kombiniert wird.

## Hintergrund & Theorie
Mit `ChangeDetectionStrategy.OnPush` prüft Angular eine Komponente nur in diesen Fällen auf Änderungen: eine `@Input()`-Referenz ändert sich, ein DOM-Event tritt auf, die `async`-Pipe empfängt einen neuen Wert, oder `ChangeDetectorRef.markForCheck()` wird manuell aufgerufen.

Mit dem Signal-Modell (Angular 17+) ändert sich das grundlegend. Signal-Inputs (`input()`) registrieren eine reaktive Abhängigkeit direkt in der Component-View. Jede Änderung eines Signals – ob `input()`, `computed()` oder `toSignal()` – löst automatisch eine View-Aktualisierung aus, unabhängig von Zone.js.

Ein wichtiges Gotcha: In einem Feld-Initialisierer ist der Wert eines `input.required()` noch nicht verfügbar (der Konstruktor läuft vor der Input-Zuweisung). Um ein Observable reaktiv auf Input-Änderungen zu subscriben, muss `toObservable(this.input)` zusammen mit `switchMap` verwendet werden. Dieses Muster ersetzt vollständig das klassische `ngOnChanges` + `subscription.unsubscribe()` Boilerplate.

## Aufgabe
Erstelle eine `ProductCardComponent` mit `ChangeDetectionStrategy.OnPush`, die:
1. Einen `product` Signal-Input (`input.required<Product>()`) empfängt
2. Den rabattierten Preis via `computed()` berechnet
3. Lagerbestand-Updates aus einem Observable bezieht, das sich bei Product-Wechsel automatisch neu subscribt
4. Alles ohne `ChangeDetectorRef`, `ngOnChanges` oder `async`-Pipe rendert

### Schritte
1. Erstelle `product.model.ts` mit einem `Product`-Interface (`id`, `name`, `price`)
2. Erstelle `stock.service.ts` mit `getStock$(id: string): Observable<number>` (simuliere mit `interval` + `map`)
3. Erstelle `product-card.component.ts` als Standalone-Komponente mit `OnPush`
4. Definiere `product = input.required<Product>()` und `discountedPrice = computed(() => ...)`
5. Kombiniere `toObservable(this.product)` + `switchMap` + `toSignal` für den reaktiven Lagerbestand
6. Erstelle eine Parent-Komponente mit einem `signal<Product>` und einem Button, der den Preis ändert
7. Verifiziere im Browser-DevTools, dass keine Zone.js-basierten Change-Detection-Zyklen ausgelöst werden (optional: mit Angular DevTools Profiler)

## Hints
<details>
<summary>Hint 1: computed() auf Basis eines Signal-Inputs</summary>

```typescript
import { ChangeDetectionStrategy, Component, computed, input } from '@angular/core';

@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
export class ProductCardComponent {
  product = input.required<Product>();

  // computed() liest den Signal-Wert – keine manuelle Subscription nötig
  discountedPrice = computed(() => this.product().price * 0.9);
}
```
</details>

<details>
<summary>Hint 2: toSignal + toObservable für reaktives Re-subscribe</summary>

```typescript
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { inject } from '@angular/core';
import { switchMap } from 'rxjs';

export class ProductCardComponent {
  product = input.required<Product>();
  private stockService = inject(StockService);

  // toObservable(signal) emittiert bei jeder Signal-Änderung einen neuen Wert.
  // switchMap kündigt die alte Subscription und öffnet eine neue.
  stock = toSignal(
    toObservable(this.product).pipe(
      switchMap(p => this.stockService.getStock$(p.id))
    ),
    { initialValue: null }
  );
}
```
</details>

## Beispiellösung

```typescript
// product.model.ts
export interface Product {
  id: string;
  name: string;
  price: number;
}

// stock.service.ts
import { Injectable } from '@angular/core';
import { interval, map } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class StockService {
  getStock$(productId: string) {
    return interval(2000).pipe(
      map(() => Math.floor(Math.random() * 100))
    );
  }
}

// product-card.component.ts
import {
  ChangeDetectionStrategy,
  Component,
  computed,
  inject,
  input,
} from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { switchMap } from 'rxjs';
import { StockService } from './stock.service';
import { Product } from './product.model';

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [CurrencyPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div style="border: 1px solid #ccc; padding: 1rem; border-radius: 4px;">
      <h3>{{ product().name }}</h3>
      <p>Originalpreis: <strong>{{ product().price | currency:'EUR' }}</strong></p>
      <p>Rabattpreis (-10%): <strong>{{ discountedPrice() | currency:'EUR' }}</strong></p>
      <p>Lagerbestand: <strong>{{ stock() ?? '...' }}</strong> Stück</p>
    </div>
  `,
})
export class ProductCardComponent {
  product = input.required<Product>();

  private stockService = inject(StockService);

  discountedPrice = computed(() => this.product().price * 0.9);

  // toObservable reaktiviert switchMap bei jedem neuen product()-Wert.
  // toSignal verwaltet die Subscription automatisch (wird bei destroy bereinigt).
  stock = toSignal(
    toObservable(this.product).pipe(
      switchMap(p => this.stockService.getStock$(p.id))
    ),
    { initialValue: null }
  );
}

// app.component.ts
import { Component, signal } from '@angular/core';
import { ProductCardComponent } from './product-card.component';
import { Product } from './product.model';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ProductCardComponent],
  template: `
    <h2>Produkt-Demo</h2>
    <app-product-card [product]="currentProduct()" />
    <br>
    <button (click)="increasePrice()">Preis +5€</button>
    <button (click)="switchProduct()">Produkt wechseln</button>
  `,
})
export class AppComponent {
  currentProduct = signal<Product>({ id: '1', name: 'Angular Buch', price: 39.99 });

  increasePrice() {
    this.currentProduct.update(p => ({ ...p, price: +(p.price + 5).toFixed(2) }));
  }

  switchProduct() {
    this.currentProduct.set({ id: '2', name: 'TypeScript Kurs', price: 59.99 });
  }
}
```

### Wichtige Beobachtungen
- Klick auf "Preis +5€": `discountedPrice()` aktualisiert sofort – kein `markForCheck()`, kein `ngOnChanges`
- Klick auf "Produkt wechseln": `stock` subscribt automatisch den neuen Observable-Stream (`switchMap` kündigt den alten)
- Die Komponente mit `OnPush` rendert exakt dann, wenn sich ein Signal ändert – nicht häufiger

## Weiterführendes
- **Vorsicht bei Feld-Initialisierern**: `input.required()` hat erst nach dem Konstruktor seinen Wert – deshalb niemals direkt `this.product()` in einem anderen Feld-Initialisierer verwenden, ohne `toObservable` als Brücke zu nutzen
- **Zoneless mit `provideExperimentalZonelessChangeDetection()`**: Signal-basierte Komponenten laufen auch vollständig ohne Zone.js; `OnPush` ist dann implizit für alle Komponenten
- Angular-Doku: [Signal Inputs](https://angular.dev/guide/signals/inputs) | [RxJS Interop](https://angular.dev/guide/signals/rxjs-interop)
