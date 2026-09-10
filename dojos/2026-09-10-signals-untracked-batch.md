# Angular Dojo: Signals – `untracked()`, `batch()` und erweiterte Reaktivitätskontrolle
**Datum:** 2026-09-10
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `untracked()` Abhängigkeiten bewusst aus reaktiven Kontexten ausschließt und mit `batch()` mehrere Signal-Updates zu einem einzigen Reaktionszyklus zusammenfasst, um unnötige Re-Renders und Effekt-Ausführungen zu vermeiden.

## Hintergrund & Theorie

Angular Signals verfolgen automatisch, welche Signale in einem reaktiven Kontext (z. B. `computed()` oder `effect()`) gelesen werden. Das ist mächtig, führt aber manchmal zu unerwünschten Abhängigkeiten:

**`untracked(fn)`** liest einen Signal-Wert, *ohne* eine Abhängigkeit zu registrieren. Das Signal wird also gelesen, aber Änderungen daran lösen keinen Re-Run des aktuellen Kontexts aus. Typische Anwendungsfälle: Logging, Audit-Trails, einmalige Initialisierungswerte.

```typescript
effect(() => {
  const user = currentUser(); // Abhängigkeit: re-runs bei Änderung
  const ts = untracked(() => timestamp()); // KEIN Re-Run bei timestamp-Änderung
  log(user, ts);
});
```

**`batch(fn)`** (ab Angular 19 als `flushEffects` / in Vorgängerversionen als Zone-Batching verfügbar, heute `batch()` aus `@angular/core`) fasst mehrere Signal-Schreiboperationen zusammen. Abhängige `computed()`-Signale und Effekte werden erst *nach* dem Batch neu berechnet, nicht nach jedem einzelnen `set()`-Aufruf.

```typescript
batch(() => {
  firstName.set('Anna');
  lastName.set('Müller');
  age.set(30);
});
// computed(() => fullName()) wird nur EINMAL neu berechnet
```

Ohne `batch()` würde jede `set()`-Operation sofort alle abhängigen Computed Signals und Effekte triggern — das kann bei komplexen Signalgraphen zu erheblichem Overhead führen.

## Aufgabe

Baue einen kleinen **Warenkorb-Service** als Standalone Component, der mehrere Signals für Artikel, Menge und Rabatt verwaltet. Demonstriere den Unterschied zwischen gebatchten und ungebatchten Updates sowie den Einsatz von `untracked()` in einem Logging-Effekt.

### Schritte

1. Erstelle eine `CartComponent` als Standalone Component mit folgenden Signals:
   - `items: WritableSignal<{ name: string; price: number; qty: number }[]>`
   - `discount: WritableSignal<number>` (0–100 %)
   - `updateCount: WritableSignal<number>` (Zähler für jede Einzeloperation)

2. Erstelle ein `computed()`-Signal `total`, das den Gesamtpreis (inkl. Rabatt) berechnet. Zähle außerdem in einer normalen Variablen mit, wie oft `total` neu berechnet wird.

3. Implementiere eine Methode `addItemUnbatched()`, die `items`, `discount` und `updateCount` *ohne* `batch()` aktualisiert. Beobachte, wie oft `total` recomputed wird.

4. Implementiere eine Methode `addItemBatched()`, die dieselben drei Signale innerhalb von `batch()` aktualisiert. Vergleiche die Recompute-Anzahl.

5. Füge einen `effect()` hinzu, der bei jeder `total`-Änderung einen Log-Eintrag schreibt. Nutze `untracked()`, um dabei den aktuellen Wert von `updateCount` zu lesen, *ohne* dass Änderungen an `updateCount` den Effekt separat triggern.

6. Zeige im Template: `total`, Recompute-Zähler, Log-Einträge und zwei Buttons für die beiden Methoden.

## Hints

<details>
<summary>Hint 1 – batch() Grundstruktur</summary>

```typescript
import { batch } from '@angular/core';

addItemBatched() {
  batch(() => {
    this.items.update(list => [...list, { name: 'Buch', price: 15, qty: 1 }]);
    this.discount.set(10);
    this.updateCount.update(n => n + 1);
  });
}
```
`total` wird erst *nach* dem Ende des `batch()`-Callbacks neu berechnet.
</details>

<details>
<summary>Hint 2 – untracked() im effect()</summary>

```typescript
import { effect, untracked } from '@angular/core';

constructor() {
  effect(() => {
    const t = this.total(); // Abhängigkeit registrieren → re-run bei total-Änderung
    const count = untracked(() => this.updateCount()); // NUR lesen, keine Abhängigkeit
    this.logs.push(`Total: ${t}€ (nach ${count} Updates)`);
  });
}
```
Ohne `untracked()` würde jede Erhöhung von `updateCount` den Effekt erneut triggern.
</details>

<details>
<summary>Hint 3 – Recompute-Zähler</summary>

Computed Signals lassen sich nicht direkt instrumentieren. Verwende stattdessen eine Klassenvariable:

```typescript
recomputeCount = 0;

total = computed(() => {
  this.recomputeCount++;
  const sum = this.items().reduce((acc, i) => acc + i.price * i.qty, 0);
  return sum * (1 - this.discount() / 100);
});
```
Da `recomputeCount` kein Signal ist, löst das Inkrementieren keine Reaktion aus — es ist nur ein Messartefakt.
</details>

## Beispiellösung

```typescript
import { Component, computed, effect, signal, untracked, batch } from '@angular/core';
import { CommonModule } from '@angular/common';

interface CartItem {
  name: string;
  price: number;
  qty: number;
}

@Component({
  selector: 'app-cart',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>Warenkorb</h2>
    <p>Gesamt: {{ total() | currency:'EUR' }}</p>
    <p>Rabatt: {{ discount() }}%</p>
    <p>Recompute-Zähler: {{ recomputeCount }}</p>
    <p>Update-Zähler: {{ updateCount() }}</p>
    <ul>
      <li *ngFor="let entry of logs">{{ entry }}</li>
    </ul>
    <button (click)="addItemUnbatched()">+ Artikel (unbatched)</button>
    <button (click)="addItemBatched()">+ Artikel (batched)</button>
  `,
})
export class CartComponent {
  items = signal<CartItem[]>([{ name: 'Stift', price: 2, qty: 3 }]);
  discount = signal(0);
  updateCount = signal(0);
  logs: string[] = [];
  recomputeCount = 0;

  total = computed(() => {
    this.recomputeCount++;
    const sum = this.items().reduce((acc, i) => acc + i.price * i.qty, 0);
    return sum * (1 - this.discount() / 100);
  });

  constructor() {
    effect(() => {
      const t = this.total();
      const count = untracked(() => this.updateCount());
      this.logs.push(`Total: ${t.toFixed(2)}€ — nach ${count} Update-Ops`);
    });
  }

  addItemUnbatched() {
    // Drei separate Sets → total wird bis zu 3× neu berechnet
    this.items.update(list => [...list, { name: 'Buch', price: 15, qty: 1 }]);
    this.discount.set(5);
    this.updateCount.update(n => n + 1);
  }

  addItemBatched() {
    // Alle Sets in einem Batch → total wird nur 1× neu berechnet
    batch(() => {
      this.items.update(list => [...list, { name: 'Heft', price: 3, qty: 2 }]);
      this.discount.set(10);
      this.updateCount.update(n => n + 1);
    });
  }
}
```

**Erwartetes Verhalten:**
- `addItemUnbatched()`: `recomputeCount` steigt um 2–3 (jedes `set()` triggert `total`).
- `addItemBatched()`: `recomputeCount` steigt um genau 1.
- `updateCount`-Änderungen triggern den Effekt *nicht* separat (dank `untracked()`).

## Weiterführendes

- **Angular Docs – Reactive Primitives:** Die offizielle Dokumentation erklärt `batch()` und `untracked()` im Kontext des Signals-Reaktivitätsmodells.
- **Tipp:** In zoneless Angular (ohne `zone.js`) ist `batch()` besonders wichtig, da es kein automatisches Change-Detection-Batching durch die Zone gibt.
- **Vertiefung:** Schreibe einen Unit-Test mit `TestBed.flushEffects()`, der prüft, dass `addItemBatched()` `total` exakt einmal recomputed — das zeigt, ob dein Batching wirklich greift.
- **Verwandt:** `linkedSignal()` und `resource()` nutzen intern ähnliche Batching-Mechanismen für ihre Updates.
