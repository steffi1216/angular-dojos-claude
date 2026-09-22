# Angular Dojo: Signal Equality Functions
**Datum:** 2026-09-22
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit benutzerdefinierten Gleichheitsfunktionen (`equal`) für `signal()` und `computed()` steuern kannst, wann ein Signal eine reaktive Änderung auslöst – und damit unnötige Re-Renders und `effect()`-Ausführungen vermeidest.

## Hintergrund & Theorie
Standardmäßig vergleicht Angular Signals Werte mit `Object.is()`. Das bedeutet: Jedes Mal, wenn du ein Signal mit einem neuen Objekt- oder Array-Referenz setzt, gilt der Wert als geändert – selbst wenn die Inhalte identisch sind. Dieses Verhalten kann zu überflüssigen Updates führen.

Mit der `equal`-Option kannst du eine eigene Vergleichsfunktion übergeben:

```typescript
const user = signal<User>({ id: 1, name: 'Alice' }, {
  equal: (a, b) => a.id === b.id && a.name === b.name
});
```

Angular ruft `equal(previousValue, nextValue)` auf, bevor es eine Änderung propagiert. Gibt die Funktion `true` zurück, wird die Änderung **unterdrückt** – alle abhängigen `computed()`-Signale und `effect()`-Callbacks bleiben unberührt.

Das Gleiche funktioniert bei `computed()`:

```typescript
const sortedIds = computed(
  () => items().map(i => i.id).sort(),
  { equal: (a, b) => a.length === b.length && a.every((v, i) => v === b[i]) }
);
```

Damit wird eine Downstream-Reaktion nur ausgelöst, wenn sich die tatsächlichen IDs ändern – nicht bei jedem Neuaufbau des Arrays.

**Wichtig:** Die `equal`-Funktion muss **pure** und **seiteneffektfrei** sein, da Angular sie mehrfach und in beliebiger Reihenfolge aufrufen kann.

## Aufgabe
Erstelle eine `ProductListComponent`, die eine Liste von Produkten als Signal verwaltet. Optimiere die Reaktivität so, dass:

1. Ein `effect()` für das Speichern ("saving to backend") **nur** feuert, wenn sich der Inhalt einer Produktliste inhaltlich ändert – nicht bei referentiell neuen Arrays mit denselben Produkten.
2. Ein `computed()`-Signal `totalPrice` nur dann als geändert gilt, wenn sich der berechnete Wert tatsächlich unterscheidet (auf 2 Dezimalstellen).
3. Eine Methode `refreshList()` demonstriert, dass das Setzen einer neuen Array-Referenz mit identischem Inhalt **keine** reaktive Aktualisierung auslöst.

### Schritte
1. Definiere ein `Product`-Interface mit `id: number`, `name: string`, `price: number`.
2. Erstelle ein `signal<Product[]>([...], { equal: shallowProductListEqual })`, wobei `shallowProductListEqual` prüft, ob Länge und alle `id`/`name`/`price`-Werte übereinstimmen.
3. Erstelle ein `computed(() => ..., { equal: (a, b) => Math.round(a * 100) === Math.round(b * 100) })` für `totalPrice`.
4. Registriere einen `effect()`, der bei Änderung der Produktliste eine Konsolenausgabe macht (`'[Backend] Saving products...'`).
5. Implementiere `refreshList()`, das das Signal mit einer strukturell identischen (aber neuen Array-Referenz) Liste setzt – und zeige in der Template-Ansicht, dass der Effect **nicht** erneut feuert.
6. Implementiere `addProduct(product: Product)`, das tatsächlich ein neues Produkt hinzufügt – und verifiziere, dass der Effect **diesmal** feuert.

## Hints
<details>
<summary>Hint 1 – Aufbau der Equality-Funktion</summary>

Eine `shallowProductListEqual`-Funktion könnte so aussehen:

```typescript
function shallowProductListEqual(a: Product[], b: Product[]): boolean {
  if (a.length !== b.length) return false;
  return a.every((p, i) =>
    p.id === b[i].id && p.name === b[i].name && p.price === b[i].price
  );
}
```

Alternativ kannst du `JSON.stringify` nutzen – aber das ist langsamer und für Produktionscode nicht empfohlen.
</details>

<details>
<summary>Hint 2 – Effect und Signal-Schreibrechte</summary>

`effect()` funktioniert seit Angular 19 ohne `allowSignalWrites`-Flag. Für eine Konsolenausgabe benötigst du es ohnehin nicht. Achte darauf, dass du `effect()` im Injection-Kontext (Konstruktor oder `inject()`-Funktion) erstellst:

```typescript
constructor() {
  effect(() => {
    const products = this.products();
    console.log('[Backend] Saving products...', products.length);
  });
}
```

Der Effect liest `this.products()` – damit registriert er es als Abhängigkeit. Nur wenn `equal` `false` zurückgibt, wird der Effect neu ausgeführt.
</details>

## Beispiellösung

```typescript
import { Component, computed, effect, signal } from '@angular/core';
import { JsonPipe } from '@angular/common';

interface Product {
  id: number;
  name: string;
  price: number;
}

function shallowProductListEqual(a: Product[], b: Product[]): boolean {
  if (a.length !== b.length) return false;
  return a.every(
    (p, i) => p.id === b[i].id && p.name === b[i].name && p.price === b[i].price
  );
}

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [JsonPipe],
  template: `
    <h2>Produkte</h2>
    <p>Gesamtpreis: {{ totalPrice() | number: '1.2-2' }} €</p>
    <pre>{{ products() | json }}</pre>
    <button (click)="refreshList()">Refresh (kein Effect)</button>
    <button (click)="addProduct({ id: 4, name: 'Monitor', price: 399.99 })">
      Produkt hinzufügen (Effect feuert)
    </button>
  `,
})
export class ProductListComponent {
  products = signal<Product[]>(
    [
      { id: 1, name: 'Laptop', price: 1299.0 },
      { id: 2, name: 'Maus', price: 29.99 },
      { id: 3, name: 'Tastatur', price: 79.99 },
    ],
    { equal: shallowProductListEqual }
  );

  // Nur geändert, wenn totalPrice auf 2 Dezimalstellen abweicht
  totalPrice = computed(
    () => this.products().reduce((sum, p) => sum + p.price, 0),
    { equal: (a, b) => Math.round(a * 100) === Math.round(b * 100) }
  );

  constructor() {
    effect(() => {
      const list = this.products();
      console.log('[Backend] Saving products...', list.length, 'items');
    });
  }

  // Neue Referenz, gleicher Inhalt → Equal gibt true zurück → kein Effect
  refreshList(): void {
    this.products.set([...this.products()].map(p => ({ ...p })));
  }

  addProduct(product: Product): void {
    this.products.update(list => [...list, product]);
  }
}
```

**Erwartetes Verhalten in der Konsole:**
- Initial: `[Backend] Saving products... 3 items`
- `refreshList()`: _keine_ Ausgabe (Effect feuert nicht)
- `addProduct(...)`: `[Backend] Saving products... 4 items`

## Weiterführendes
- **`signal()` mit `equal: () => false`** erzwingt immer eine Änderung – nützlich für Trigger-Signale oder Reload-Pattern.
- Für tiefe Objektvergleiche bietet sich [`structuredClone`](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone) + Vergleich an, aber beachte die Performance-Kosten.
- Offizielle Docs: [Angular Signals – Equality](https://angular.dev/guide/signals#equality-functions)
- Kombiniere Equality-Funktionen mit `computed()` für selektive Memoization – ähnlich wie `distinctUntilChanged` in RxJS.
