# Angular Dojo: linkedSignal – Verknüpfte, überschreibbare Signale
**Datum:** 2026-07-13
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie `linkedSignal()` aus Angular 19 eine Lücke zwischen `signal()` und `computed()` schließt: ein abgeleitetes Signal, das automatisch aktualisiert wird, wenn sich seine Quelle ändert – aber trotzdem manuell überschrieben werden kann.

## Hintergrund & Theorie
In Angular Signals gibt es zwei bekannte Primitiven:

- **`signal()`** – vollständig schreibbar, aber keine automatische Ableitung
- **`computed()`** – automatisch abgeleitet, aber schreibgeschützt (read-only)

**`linkedSignal()`** füllt genau diese Lücke. Es erzeugt ein **WritableSignal**, das:
1. automatisch seinen Wert aus einem Quell-Signal ableitet
2. manuell überschrieben (`set()` / `update()`) werden kann
3. beim nächsten Ändern des Quell-Signals auf den neuen abgeleiteten Wert **zurückgesetzt** wird

```typescript
const source = signal('A');
const linked = linkedSignal(() => source() + '-default');

linked(); // 'A-default'
linked.set('mein Wert');
linked(); // 'mein Wert'

source.set('B');
linked(); // 'B-default' ← automatisch zurückgesetzt
```

Typischer Anwendungsfall: Ein Formularfeld, das einen sinnvollen Standardwert aus einer Auswahl ableitet, aber trotzdem vom Nutzer überschrieben werden soll.

`linkedSignal` akzeptiert auch eine erweiterte Form mit `source` und `computation`, die Zugriff auf den **vorherigen Wert** (`previous`) gibt – nützlich für fortgeschrittene Reset-Logik.

## Aufgabe

Baue eine **Produktkonfigurator-Komponente** in Angular 19 (standalone, signals-basiert):

1. Der Nutzer wählt eine Produktkategorie (`Laptop`, `Smartphone`, `Tablet`)
2. Jede Kategorie hat eine **Standard-Farbe** (`Silber`, `Schwarz`, `Rosé`)
3. Der Nutzer kann die Farbe **manuell überschreiben**
4. Wechselt der Nutzer die Kategorie, soll die Farbe auf den **neuen Standard** zurückgesetzt werden
5. Zeige in der UI an, ob die aktuelle Farbe dem Standard entspricht oder überschrieben wurde

### Schritte

1. Erstelle eine standalone Komponente `ProductConfiguratorComponent`
2. Definiere ein `category`-Signal und eine Map mit Standardfarben pro Kategorie
3. Verwende `linkedSignal()`, um die `color` automatisch aus der Kategorie abzuleiten
4. Füge Buttons zum Wechseln der Kategorie und ein `<select>` oder Buttons für die Farbe hinzu
5. Leite mit `computed()` einen `isDefault`-Indikator ab (`color() === defaultColor()`)
6. Zeige alle Werte reaktiv in der Template-Ausgabe

## Hints

<details>
<summary>Hint 1 – Standardfarben-Map und linkedSignal</summary>

```typescript
const COLOR_DEFAULTS: Record<string, string> = {
  Laptop: 'Silber',
  Smartphone: 'Schwarz',
  Tablet: 'Rosé',
};

category = signal<string>('Laptop');

// linkedSignal leitet den Standardwert ab
color = linkedSignal(() => COLOR_DEFAULTS[this.category()]);
```
Sobald `category` sich ändert, wird `color` automatisch auf den neuen Standardwert zurückgesetzt.
</details>

<details>
<summary>Hint 2 – isDefault und Template-Binding</summary>

```typescript
defaultColor = computed(() => COLOR_DEFAULTS[this.category()]);
isDefault = computed(() => this.color() === this.defaultColor());
```

Im Template:
```html
<p>Farbe: {{ color() }} <span *ngIf="!isDefault()">(angepasst)</span></p>
<button (click)="color.set('Blau')">Blau wählen</button>
<button (click)="category.set('Tablet')">Tablet wählen</button>
```
</details>

<details>
<summary>Hint 3 – Erweiterte linkedSignal-Form mit previous</summary>

Mit der Langform behältst du beim Kategorie-Wechsel die alte Farbe, falls sie dem alten Standard entsprach:
```typescript
color = linkedSignal<string, string>({
  source: this.category,
  computation: (newCategory, previous) => {
    // Wenn vorher kein Override → neuen Standard setzen
    if (!previous || previous.value === COLOR_DEFAULTS[previous.source]) {
      return COLOR_DEFAULTS[newCategory];
    }
    return previous.value; // Override beibehalten
  },
});
```
Achtung: `previous` ist beim ersten Aufruf `undefined`.
</details>

## Beispiellösung

```typescript
import { Component, computed, linkedSignal, signal } from '@angular/core';
import { NgFor, NgIf } from '@angular/common';

const COLOR_DEFAULTS: Record<string, string> = {
  Laptop: 'Silber',
  Smartphone: 'Schwarz',
  Tablet: 'Rosé',
};

const CATEGORIES = Object.keys(COLOR_DEFAULTS);
const ALL_COLORS = ['Silber', 'Schwarz', 'Rosé', 'Blau', 'Grün', 'Rot'];

@Component({
  selector: 'app-product-configurator',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <h2>Produktkonfigurator</h2>

    <section>
      <h3>Kategorie</h3>
      <button
        *ngFor="let cat of categories"
        [style.font-weight]="category() === cat ? 'bold' : 'normal'"
        (click)="category.set(cat)"
      >
        {{ cat }}
      </button>
    </section>

    <section>
      <h3>Farbe</h3>
      <button
        *ngFor="let col of colors"
        [style.outline]="color() === col ? '2px solid black' : 'none'"
        (click)="color.set(col)"
      >
        {{ col }}
      </button>
      <button (click)="resetColor()">Zurücksetzen</button>
    </section>

    <section>
      <h3>Konfiguration</h3>
      <p>Kategorie: <strong>{{ category() }}</strong></p>
      <p>
        Farbe: <strong>{{ color() }}</strong>
        <span *ngIf="!isDefault()"> ✏️ (angepasst, Standard: {{ defaultColor() }})</span>
        <span *ngIf="isDefault()"> ✓ (Standard)</span>
      </p>
    </section>
  `,
})
export class ProductConfiguratorComponent {
  readonly categories = CATEGORIES;
  readonly colors = ALL_COLORS;

  category = signal<string>('Laptop');

  // linkedSignal: schreibbar UND automatisch abgeleitet
  color = linkedSignal(() => COLOR_DEFAULTS[this.category()]);

  defaultColor = computed(() => COLOR_DEFAULTS[this.category()]);
  isDefault = computed(() => this.color() === this.defaultColor());

  resetColor() {
    this.color.set(this.defaultColor());
  }
}
```

**Kernpunkt**: Wechsel die Kategorie, während eine angepasste Farbe aktiv ist – `color` springt sofort auf den neuen Kategoriestandard zurück. Das ist der entscheidende Unterschied zu `signal()` + manuellem `effect()`.

## Weiterführendes

- **Langform mit `previous`**: Nützlich, wenn du beim Quell-Wechsel eine bedingte Reset-Logik brauchst (Override beibehalten oder verwerfen). Sieh dir Hint 3 und die [Angular-Doku zu `linkedSignal`](https://angular.dev/guide/signals#linked-signal) an.
- **Vergleich der Signal-Primitiven**: `signal` → frei schreibbar | `computed` → read-only abgeleitet | `linkedSignal` → schreibbar abgeleitet | `resource` → async abgeleitet (HTTP, Promises)
- **Kombination mit `resource()`**: Ein `linkedSignal` eignet sich als schreibbares Default-Ergebnis, das du mit einem `resource()` koppeln kannst, sobald du einen Remote-Default laden willst.
