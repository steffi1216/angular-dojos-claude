# Angular Dojo: Change Detection Strategien
**Datum:** 2026-06-13
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst den Unterschied zwischen der Standard-Change-Detection und der `OnPush`-Strategie, weißt wann Angular Views neu rendert, und kannst mit `ChangeDetectorRef` gezielt steuern, wann eine Komponente aktualisiert wird.

## Hintergrund & Theorie
Angular prüft standardmäßig bei jedem Browser-Event, Timer oder HTTP-Request den gesamten Komponentenbaum auf Änderungen (Default-Strategie via Zone.js). Das ist einfach, aber bei großen Apps teuer.

Mit `ChangeDetectionStrategy.OnPush` markierst du eine Komponente als "nur aktualisieren, wenn":
- ein `@Input()`-Wert eine neue Referenz erhält,
- ein eigenes Event (Output/DOM) in der Komponente ausgelöst wird,
- ein `async`-Pipe im Template auflöst,
- du manuell `markForCheck()` oder `detectChanges()` aufrufst.

**Wichtige `ChangeDetectorRef`-Methoden:**
| Methode | Wirkung |
|---|---|
| `markForCheck()` | Markiert die Komponente + Vorfahren für den nächsten Zyklus |
| `detectChanges()` | Führt Change Detection sofort für diese Komponente aus |
| `detach()` | Hängt die Komponente komplett aus dem Zyklus aus |
| `reattach()` | Hängt sie wieder ein |

Mit Angulars neuem Zoneless-Modus (experimentell, ab Angular 18) entfällt Zone.js ganz – Reaktivität kommt ausschließlich von Signals oder expliziten `markForCheck()`-Aufrufen.

## Aufgabe
Baue eine kleine App mit zwei Komponenten: einem **Counter-Parent** und einer **Display-Child**-Komponente. Die Child-Komponente soll `OnPush` nutzen und trotzdem korrekt aktualisiert werden – durch richtige Referenzübergabe und durch `ChangeDetectorRef`.

### Schritte

1. **Erstelle `DisplayComponent`** (Standalone, `OnPush`):
   - Empfängt ein `@Input() count: number` und ein `@Input() label: string`
   - Zeigt `label: count` im Template an
   - Logge in `ngDoCheck()`: `console.log('DisplayComponent checked')`

2. **Erstelle `CounterComponent`** (Parent, Standard-CD):
   - Hat zwei Zähler: `primitiveCount` (number) und `objectCount` (ein Objekt `{ value: number }`)
   - Zwei Buttons: „Increment Primitive" und „Increment Object"
   - **Wichtig für den Bug:** Beim Objekt-Button mutiere das Objekt direkt (`this.objectCount.value++`) statt eine neue Referenz zu erstellen
   - Binde beide an je eine `DisplayComponent`

3. **Beobachte das Verhalten:**
   - Der Primitiv-Zähler aktualisiert die Child-Komponente korrekt (neue Referenz bei jedem Wert)
   - Der Objekt-Zähler aktualisiert die Child-Komponente **nicht** (gleiche Referenz – OnPush ignoriert die Mutation)

4. **Fixe den Object-Button:**
   - Erstelle stattdessen eine neue Referenz: `this.objectCount = { value: this.objectCount.value + 1 }`
   - Beobachte, dass das Update jetzt korrekt ankommt

5. **Bonus:** Füge einen dritten Button hinzu, der manuell `detectChanges()` aus dem Parent triggert, indem er eine `@ViewChild`-Referenz auf die Child-Komponente nutzt und deren `ChangeDetectorRef` aufruft.

## Hints

<details>
<summary>Hint 1 – OnPush deklarieren</summary>

```typescript
import { ChangeDetectionStrategy, Component, Input } from '@angular/core';

@Component({
  selector: 'app-display',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ label }}: {{ count }}</p>`,
})
export class DisplayComponent {
  @Input() count = 0;
  @Input() label = '';

  ngDoCheck() {
    console.log('DisplayComponent checked');
  }
}
```

</details>

<details>
<summary>Hint 2 – Neue Referenz statt Mutation</summary>

```typescript
// Falsch (OnPush merkt keine Änderung):
this.objectCount.value++;

// Richtig (neue Referenz → OnPush erkennt Änderung):
this.objectCount = { value: this.objectCount.value + 1 };
```

</details>

<details>
<summary>Hint 3 – ViewChild + detectChanges</summary>

```typescript
import { ViewChild, ChangeDetectorRef } from '@angular/core';
import { DisplayComponent } from './display.component';

// Im Parent:
@ViewChild(DisplayComponent) displayRef!: DisplayComponent;

// Bonus-Button-Handler:
forceUpdate() {
  // Direkt auf die CDRef der Child-Instanz zugreifen:
  this.displayRef['cdRef'].detectChanges();
  // Alternativ: im Child ein public cdRef injizieren und von außen aufrufen
}
```

Besser: Injiziere `ChangeDetectorRef` im Child als `public` und rufe es vom Parent aus auf.

</details>

## Beispiellösung

```typescript
// display.component.ts
import {
  ChangeDetectionStrategy,
  ChangeDetectorRef,
  Component,
  Input,
  inject,
} from '@angular/core';

@Component({
  selector: 'app-display',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ label }}: {{ count }}</p>`,
})
export class DisplayComponent {
  @Input() count = 0;
  @Input() label = '';

  readonly cdr = inject(ChangeDetectorRef);

  ngDoCheck() {
    console.log(`[${this.label}] checked`);
  }
}
```

```typescript
// counter.component.ts
import { Component, ViewChild } from '@angular/core';
import { DisplayComponent } from './display.component';

@Component({
  selector: 'app-counter',
  standalone: true,
  imports: [DisplayComponent],
  template: `
    <app-display label="Primitive" [count]="primitiveCount" />
    <app-display label="Object"    [count]="objectCount.value" #objDisplay />

    <button (click)="incrementPrimitive()">Increment Primitive</button>
    <button (click)="incrementObject()">Increment Object</button>
    <button (click)="forceUpdate()">Force Update (detectChanges)</button>
  `,
})
export class CounterComponent {
  primitiveCount = 0;
  objectCount = { value: 0 };

  @ViewChild('objDisplay') objDisplay!: DisplayComponent;

  incrementPrimitive() {
    this.primitiveCount++; // neue Referenz (number ist primitiv)
  }

  incrementObject() {
    // Neue Referenz erzeugen – OnPush erkennt die Änderung
    this.objectCount = { value: this.objectCount.value + 1 };
  }

  forceUpdate() {
    // Manuelle Mutation + detectChanges – nur zum Demonstrieren
    this.objectCount.value += 10;
    this.objDisplay.cdr.detectChanges();
  }
}
```

## Weiterführendes
- **Zoneless Angular:** Ab Angular 18 kann Zone.js komplett deaktiviert werden (`provideExperimentalZonelessChangeDetection()`). Dann funktioniert Reaktivität nur noch über Signals oder explizite CDRef-Aufrufe – ein guter nächster Dojo!
- **Angular DevTools:** Im Browser-DevTools-Plugin kannst du unter „Profiler" live sehen, welche Komponenten bei welchem Event gecheckt werden – sehr hilfreich zum Debuggen von OnPush-Problemen.
- [Angular Docs – Change Detection](https://angular.dev/best-practices/skipping-subtrees)
