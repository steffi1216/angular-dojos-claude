# Angular Dojo: Signal-basierte Queries
**Datum:** 2026-07-14
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst die neuen signal-basierten Varianten von `@ViewChild`, `@ViewChildren`, `@ContentChild` und `@ContentChildren` kennen und verstehst, warum sie im Zusammenspiel mit Signals und OnPush deutlich ergonomischer sind als die dekorator-basierten Vorgänger.

## Hintergrund & Theorie

Ab Angular 17.1 ersetzt die Signal Query API die klassikenbasierten Dekoratoren `@ViewChild`, `@ViewChildren`, `@ContentChild` und `@ContentChildren` durch Funktionsaufrufe, die ein `Signal` zurückgeben.

**Vorher (dekorator-basiert):**
```typescript
@ViewChild('myEl') myEl!: ElementRef;
// Verfügbar erst ab ngAfterViewInit – kein reaktiver Kontext
```

**Nachher (signal-basiert):**
```typescript
myEl = viewChild.required<ElementRef>('myEl');
// Gibt ein Signal zurück – direkt reaktiv nutzbar
```

Die neuen Funktionen:

| Funktion | Entspricht | Rückgabe |
|---|---|---|
| `viewChild(ref)` | `@ViewChild` | `Signal<T \| undefined>` |
| `viewChild.required(ref)` | `@ViewChild({ ... })` | `Signal<T>` |
| `viewChildren(ref)` | `@ViewChildren` | `Signal<readonly T[]>` |
| `contentChild(ref)` | `@ContentChild` | `Signal<T \| undefined>` |
| `contentChild.required(ref)` | `@ContentChild({ ... })` | `Signal<T>` |
| `contentChildren(ref)` | `@ContentChildren` | `Signal<readonly T[]>` |

Der zentrale Vorteil: Da sie Signals zurückgeben, können sie direkt in `computed()`, `effect()` und Templates genutzt werden – ohne den Umweg über Lifecycle-Hooks wie `ngAfterViewInit`.

## Aufgabe

Baue eine `<app-color-palette>`-Komponente, die eine Liste von Farbboxen rendert. Eine übergeordnete `<app-demo>`-Komponente soll:

1. Per `viewChildren` alle Farbbox-Elemente als Signal abfragen
2. Die Anzahl der sichtbaren Boxen als `computed()` ableiten
3. Eine einzelne, per Template-Variable markierte "aktive" Box per `viewChild` abfragen und deren Hintergrundfarbe per `effect()` in die Konsole loggen, wenn sie sich ändert
4. Content Projection: Eine `<app-panel>`-Komponente nutzt `contentChildren`, um alle projizierten `<app-color-box>`-Elemente zu zählen

### Schritte

1. **Erstelle `ColorBoxComponent`** – eine einfache Standalone-Komponente mit einem `color`-Input (Signal-basiert via `input()`) und einem sichtbaren farbigen `div`.

2. **Erstelle `DemoComponent`** – rendere 5 `ColorBoxComponent`-Instanzen. Nutze `viewChildren(ColorBoxComponent)` um alle zu erfassen und zeige ihre Anzahl via `computed()` im Template an.

3. **Template-Referenz abfragen** – Versehe eine der Boxen mit `#activeBox` und frage sie via `viewChild.required<ColorBoxComponent>('activeBox')` ab. Logge die aktuelle Farbe dieser Box in einem `effect()`.

4. **`PanelComponent` mit Content Projection** – Erstelle eine `<app-panel>`-Komponente, die `<ng-content>` nutzt. Projiziere die Farbboxen hinein und nutze `contentChildren(ColorBoxComponent)` um die Anzahl der projizierten Boxen anzuzeigen.

5. **Bonus:** Füge einen Button hinzu, der dynamisch eine weitere Box hinzufügt, und beobachte, wie die `viewChildren`-Zählung automatisch reagiert.

## Hints

<details>
<summary>Hint 1 – Imports und Grundstruktur</summary>

Alle Query-Funktionen kommen aus `@angular/core`:

```typescript
import { viewChild, viewChildren, contentChild, contentChildren } from '@angular/core';
```

Die Funktionen müssen im **Initialisierungskontext** der Klasse aufgerufen werden (als Feld-Initializer), nicht in einem Lifecycle-Hook. Das ist dasselbe Prinzip wie bei `signal()` und `input()`.

```typescript
@Component({ ... })
export class DemoComponent {
  // Korrekt – Feld-Initializer
  boxes = viewChildren(ColorBoxComponent);

  // FALSCH – darf nicht im Constructor oder ngOnInit stehen
}
```

</details>

<details>
<summary>Hint 2 – computed() und effect() auf Query-Signals</summary>

Da `viewChildren()` ein `Signal<readonly T[]>` zurückgibt, kannst du es direkt in `computed()` verwenden:

```typescript
boxCount = computed(() => this.boxes().length);
```

Und in `effect()`:

```typescript
constructor() {
  effect(() => {
    const active = this.activeBox();
    console.log('Aktive Farbe:', active.color());
  });
}
```

Beachte: `effect()` muss im Injection-Kontext aufgerufen werden (Constructor oder `runInInjectionContext`).

</details>

<details>
<summary>Hint 3 – contentChildren und Timing</summary>

`contentChildren()` liefert ein Signal – kein `QueryList` mehr, kein `ngAfterContentInit` nötig:

```typescript
@Component({
  selector: 'app-panel',
  template: `
    <p>Enthält {{ projectedCount() }} Farbboxen</p>
    <ng-content />
  `
})
export class PanelComponent {
  private boxes = contentChildren(ColorBoxComponent);
  projectedCount = computed(() => this.boxes().length);
}
```

Vergleich zum alten Ansatz: Früher musstest du `QueryList.changes` subscriben. Jetzt reagiert das Signal automatisch.

</details>

## Beispiellösung

```typescript
// color-box.component.ts
import { Component, input } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-color-box',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div
      class="color-box"
      [style.background-color]="color()"
    >{{ color() }}</div>
  `,
  styles: [`
    .color-box {
      width: 100px;
      height: 100px;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      color: white;
      font-size: 12px;
      border-radius: 8px;
      margin: 4px;
      font-weight: bold;
    }
  `]
})
export class ColorBoxComponent {
  color = input.required<string>();
}
```

```typescript
// panel.component.ts
import { Component, computed, contentChildren } from '@angular/core';
import { ColorBoxComponent } from './color-box.component';

@Component({
  selector: 'app-panel',
  standalone: true,
  template: `
    <div style="border: 2px solid #ccc; padding: 16px; border-radius: 8px;">
      <h3>Panel – {{ projectedCount() }} Box(en) projiziert</h3>
      <ng-content />
    </div>
  `
})
export class PanelComponent {
  private boxes = contentChildren(ColorBoxComponent);
  projectedCount = computed(() => this.boxes().length);
}
```

```typescript
// demo.component.ts
import {
  Component,
  computed,
  effect,
  signal,
  viewChild,
  viewChildren
} from '@angular/core';
import { ColorBoxComponent } from './color-box.component';
import { PanelComponent } from './panel.component';

const COLORS = ['#e74c3c', '#3498db', '#2ecc71', '#f39c12', '#9b59b6'];

@Component({
  selector: 'app-demo',
  standalone: true,
  imports: [ColorBoxComponent, PanelComponent],
  template: `
    <h2>Signal Queries Demo</h2>

    <!-- viewChildren: zählt alle direkten Boxen -->
    <p>Direkte Boxen im View: <strong>{{ boxCount() }}</strong></p>

    <app-color-box
      *ngFor="let c of colors()"
      [color]="c"
    />

    <!-- viewChild mit Template-Referenz -->
    <app-color-box #activeBox color="#e74c3c" />
    <p>Aktive Box (per #activeBox): {{ activeBox().color() }}</p>

    <button (click)="addBox()">Box hinzufügen</button>

    <!-- contentChildren in PanelComponent -->
    <app-panel>
      <app-color-box color="#1abc9c" />
      <app-color-box color="#e67e22" />
      <app-color-box color="#34495e" />
    </app-panel>
  `
})
export class DemoComponent {
  colors = signal([...COLORS]);

  // viewChildren – alle ColorBoxComponent-Instanzen im Template
  private allBoxes = viewChildren(ColorBoxComponent);
  boxCount = computed(() => this.allBoxes().length);

  // viewChild.required – die spezifische Box per Template-Referenz
  activeBox = viewChild.required<ColorBoxComponent>('activeBox');

  constructor() {
    effect(() => {
      // Reaktiv: wird neu ausgeführt, wenn sich activeBox oder ihre Farbe ändert
      console.log('[effect] Aktive Box Farbe:', this.activeBox().color());
    });
  }

  addBox() {
    const randomColor = `hsl(${Math.random() * 360}, 70%, 50%)`;
    this.colors.update(c => [...c, randomColor]);
  }
}
```

**Kernpunkte der Lösung:**
- `viewChildren(ColorBoxComponent)` – reaktives Signal über alle Instanzen im View
- `viewChild.required<T>('ref')` – wirft einen Fehler zur Entwicklungszeit, wenn `#activeBox` nicht im Template existiert (kein `!`-Assertion nötig)
- `contentChildren(ColorBoxComponent)` – zählt projizierte Komponenten ohne `QueryList.changes`
- Alle Queries sind direkt in `computed()` und `effect()` nutzbar – kein `ngAfterViewInit`-Boilerplate

## Weiterführendes

- **Offizielle Docs:** [Angular Signal Queries Guide](https://angular.dev/guide/components/queries)
- **Migration:** Das Angular-Team stellt einen `@angular/core` Schematic zur Verfügung (`ng generate @angular/core:signal-queries-migration`), der `@ViewChild`-Dekoratoren automatisch migriert
- **Kombination mit `resource()`:** Signal Queries lassen sich direkt als Dependencies in `resource()` verwenden – z.B. um Daten nachzuladen, sobald ein Element im DOM erscheint
- **Tiefergehend:** Vergleiche das Timing-Verhalten: Signal Queries spiegeln den aktuellen Render-Stand wider und sind ab dem ersten Rendering-Zyklus verfügbar – früher als `ngAfterViewInit` für einfache Fälle
