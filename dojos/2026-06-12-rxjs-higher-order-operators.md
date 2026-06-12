# Angular Dojo: RxJS Higher-Order Mapping Operators
**Datum:** 2026-06-12
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Verstehe den Unterschied zwischen `switchMap`, `mergeMap`, `concatMap` und `exhaustMap` und wähle in konkreten Angular-Szenarien den richtigen Operator.

## Hintergrund & Theorie

Alle vier Operatoren projizieren jeden emittierten Wert eines Quell-Observables auf ein neues (inneres) Observable – daher "Higher-Order". Der Unterschied liegt darin, wie sie mit **gleichzeitig aktiven inneren Observables** umgehen:

| Operator | Strategie | Typisches Szenario |
|---|---|---|
| `switchMap` | Bricht das vorherige innere Observable ab, sobald ein neuer Wert eintrifft | Live-Suche (Typeahead) |
| `mergeMap` | Alle inneren Observables laufen gleichzeitig, keine Reihenfolgengarantie | Parallele HTTP-Requests |
| `concatMap` | Wartet, bis das aktuelle innere Observable abgeschlossen ist, bevor es das nächste startet | Sequentielle Uploads |
| `exhaustMap` | Ignoriert neue Quellwerte, solange ein inneres Observable noch aktiv ist | Login-Button (kein Doppelklick) |

Ein häufiger Bug in Angular: `mergeMap` für eine Suche verwenden, sodass alte HTTP-Responses neuere Ergebnisse überschreiben können. `switchMap` löst das, indem es laufende Requests automatisch cancelt.

## Aufgabe

Baue eine **Produkt-Suchkomponente** mit einem `<input>`-Feld. Bei jeder Eingabe soll nach 300 ms ein HTTP-Request (simuliert) abgesetzt werden. Implementiere die Komponente mit dem *richtigen* Operator und erkläre dabei, warum die anderen drei Operatoren hier ungeeignet wären.

### Schritte

1. Erstelle eine neue Standalone-Komponente `ProductSearchComponent`.
2. Erstelle einen `ProductService` mit einer Methode `search(query: string): Observable<string[]>`, die einen `HttpClient.get`-Aufruf simuliert (du kannst `of([...]).pipe(delay(500))` verwenden).
3. Verbinde das `<input>`-Element via `fromEvent` oder einem `FormControl` mit dem Service.
4. Wende `debounceTime(300)` und `distinctUntilChanged()` an, bevor du auf den Service mappst.
5. Wähle den passenden Higher-Order-Operator und begründe die Wahl in einem Kommentar.
6. Zeige die Ergebnisse in der Template mit `@for` an.
7. Bonus: Implementiere einen zweiten Use-Case mit `exhaustMap` – z. B. einen „In den Warenkorb"-Button, der während des laufenden Requests weitere Klicks ignoriert.

## Hints

<details>
<summary>Hint 1 – Welcher Operator für die Suche?</summary>

`switchMap` ist korrekt: Sobald der Nutzer weiter tippt, wird der vorherige HTTP-Request gecancelt. So landen veraltete Antworten nie in der UI.

```typescript
this.searchResults$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.productService.search(query))
);
```

</details>

<details>
<summary>Hint 2 – exhaustMap für den Warenkorb-Button</summary>

```typescript
@Component({ ... })
export class AddToCartComponent {
  private clicks$ = new Subject<void>();

  readonly addToCart$ = this.clicks$.pipe(
    exhaustMap(() => this.cartService.add(this.productId))
  );

  onClick() {
    this.clicks$.next();
  }
}
```

`exhaustMap` stellt sicher, dass ein zweiter Klick ignoriert wird, solange `cartService.add()` noch läuft.

</details>

## Beispiellösung

```typescript
// product.service.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private products = ['Apple', 'Apricot', 'Banana', 'Blueberry', 'Cherry', 'Coconut'];

  search(query: string): Observable<string[]> {
    const results = this.products.filter(p =>
      p.toLowerCase().includes(query.toLowerCase())
    );
    // Simuliert Netzwerkverzögerung
    return of(results).pipe(delay(500));
  }
}
```

```typescript
// product-search.component.ts
import { Component, OnInit } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { Observable } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';
import { AsyncPipe } from '@angular/common';
import { ProductService } from './product.service';

@Component({
  selector: 'app-product-search',
  standalone: true,
  imports: [ReactiveFormsModule, AsyncPipe],
  template: `
    <input [formControl]="searchControl" placeholder="Produkt suchen..." />

    @if (searchResults$ | async; as results) {
      <ul>
        @for (result of results; track result) {
          <li>{{ result }}</li>
        }
      </ul>
    }
  `
})
export class ProductSearchComponent implements OnInit {
  searchControl = new FormControl('');
  searchResults$!: Observable<string[]>;

  constructor(private productService: ProductService) {}

  ngOnInit() {
    this.searchResults$ = this.searchControl.valueChanges.pipe(
      debounceTime(300),          // Warte 300ms nach letztem Tastendruck
      distinctUntilChanged(),     // Nur bei tatsächlicher Änderung weiterleiten
      // switchMap cancelt laufende Requests – verhindert "Race Conditions"
      switchMap(query => this.productService.search(query ?? ''))
    );
  }
}
```

```typescript
// Warenkorb-Beispiel mit exhaustMap
import { Subject } from 'rxjs';
import { exhaustMap } from 'rxjs/operators';

@Component({
  selector: 'app-add-to-cart',
  standalone: true,
  template: `<button (click)="onClick()">In den Warenkorb</button>`
})
export class AddToCartComponent {
  private clicks$ = new Subject<void>();

  constructor(private cartService: CartService) {
    // exhaustMap ignoriert Klicks, solange ein Request läuft
    this.clicks$.pipe(
      exhaustMap(() => this.cartService.add(this.productId))
    ).subscribe(() => console.log('Produkt hinzugefügt'));
  }

  onClick() {
    this.clicks$.next();
  }
}
```

## Weiterführendes

- **RxJS Marble Testing**: Schreibe Unit-Tests für deine Operatoren mit `TestScheduler` und Marble-Diagrammen – ideal um Race Conditions zu reproduzieren und zu verhindern.
- Interaktive Visualisierung der Operatoren: [rxmarbles.com](https://rxmarbles.com) – dort siehst du den Unterschied der vier Operatoren auf einen Blick.
- Tipp: In Angular 17+ lassen sich viele `switchMap`-Patterns durch `toSignal()` + `linkedSignal` ersetzen – vergleiche beide Ansätze hinsichtlich Lesbarkeit und Testbarkeit.
