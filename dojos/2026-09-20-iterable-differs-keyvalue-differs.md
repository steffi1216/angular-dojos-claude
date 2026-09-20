# Angular Dojo: IterableDiffers & KeyValueDiffers – Angulars interne Diffing-Algorithmen

**Datum:** 2026-09-20
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du verstehst, wie Angular intern Änderungen in Collections und Objekten erkennt, und kannst `IterableDiffers` sowie `KeyValueDiffers` nutzen, um eigene Direktiven zu bauen, die effizient auf Collection-Änderungen reagieren.

## Hintergrund & Theorie

Hinter `*ngFor` und `NgClass` stecken zwei wenig bekannte, aber mächtige Angular-Services: `IterableDiffers` und `KeyValueDiffers`.

**`IterableDiffers`** verwaltet eine Registry von Diff-Algorithmen für iterierbare Collections (Arrays, Sets). Es liefert einen `IterableDiffer`, der bei jedem `diff()`-Aufruf feststellt, welche Einträge hinzugefügt, entfernt oder bewegt wurden – ohne die gesamte Collection neu zu rendern.

**`KeyValueDiffers`** macht dasselbe für Objekte und `Map`s: Es erkennt, welche Properties hinzugekommen sind, sich geändert haben oder gelöscht wurden.

Angular nutzt diese Services intern überall, wo Collections verfolgt werden müssen. Als Entwickler:in kann man sie direkt injecten und in eigenen Direktiven verwenden – zum Beispiel für:
- Custom Logs, wenn sich ein Array ändert
- Animationen, die nur für neu hinzugekommene Elemente triggern
- Performante Custom-Rendering-Logik ohne `ngFor`

Das API ist absichtlich low-level: Man ruft `find()` auf dem `Differs`-Service auf, um den passenden Differ-Algorithmus zu finden, erstellt ihn mit `create()`, und ruft dann zyklisch `diff()` auf.

## Aufgabe

Erstelle eine Direktive `appTrackChanges`, die auf ein beliebiges Array angewendet werden kann und in der Konsole ausgibt, welche Elemente hinzugefügt oder entfernt wurden – ohne dass das DOM neu gerendert wird.

Danach erweitere den Ansatz mit einem Service, der `KeyValueDiffers` nutzt, um Änderungen an einem Konfigurationsobjekt zu verfolgen und ein Signal zu aktualisieren.

### Schritte

1. **`TrackChangesDirective` erstellen** – Injiziere `IterableDiffers` und erstelle einen Differ für das Input-Array `items`.
2. **`ngDoCheck` implementieren** – Rufe `diff()` auf und logge hinzugefügte und entfernte Elemente via `forEachAddedItem()` und `forEachRemovedItem()`.
3. **`ConfigTrackerService` erstellen** – Injiziere `KeyValueDiffers` und verfolge Änderungen an einem Konfigurationsobjekt. Aktualisiere ein `Signal<string[]>` mit den Namen der geänderten Properties.
4. **Demo-Komponente bauen** – Nutze die Direktive und den Service in einer kleinen Komponente, die eine Liste verwaltet und ein Konfigurationsobjekt per Button ändert.

## Hints

<details>
<summary>Hint 1 – IterableDiffer initialisieren</summary>

```typescript
// Im Constructor oder mit inject():
private differ: IterableDiffer<string> | null = null;

constructor(private differs: IterableDiffers) {}

ngOnChanges() {
  if (!this.differ && this.items) {
    this.differ = this.differs.find(this.items).create();
  }
}
```

Der erste Aufruf von `diff()` gibt immer `null` zurück (initialer Zustand).
Ab dem zweiten Aufruf liefert es ein `IterableChanges`-Objekt oder `null` wenn keine Änderung.

</details>

<details>
<summary>Hint 2 – Änderungen auswerten</summary>

```typescript
ngDoCheck(): void {
  if (this.differ) {
    const changes = this.differ.diff(this.items);
    if (changes) {
      changes.forEachAddedItem(record =>
        console.log('Hinzugefügt:', record.item, 'an Index', record.currentIndex)
      );
      changes.forEachRemovedItem(record =>
        console.log('Entfernt:', record.item, 'von Index', record.previousIndex)
      );
    }
  }
}
```

Für `KeyValueDiffers` entsprechend:
```typescript
const changes = this.kvDiffer.diff(this.config);
if (changes) {
  changes.forEachChangedItem(record =>
    console.log(`${record.key}: ${record.previousValue} → ${record.currentValue}`)
  );
}
```

</details>

## Beispiellösung

```typescript
// track-changes.directive.ts
import {
  Directive, Input, OnChanges, DoCheck,
  IterableDiffers, IterableDiffer
} from '@angular/core';

@Directive({ selector: '[appTrackChanges]', standalone: true })
export class TrackChangesDirective implements OnChanges, DoCheck {
  @Input('appTrackChanges') items: string[] = [];

  private differ: IterableDiffer<string> | null = null;

  constructor(private differs: IterableDiffers) {}

  ngOnChanges(): void {
    if (!this.differ && this.items) {
      this.differ = this.differs.find(this.items).create();
    }
  }

  ngDoCheck(): void {
    if (!this.differ) return;
    const changes = this.differ.diff(this.items);
    if (changes) {
      changes.forEachAddedItem(r =>
        console.log(`[TrackChanges] + "${r.item}" an Index ${r.currentIndex}`)
      );
      changes.forEachRemovedItem(r =>
        console.log(`[TrackChanges] - "${r.item}" von Index ${r.previousIndex}`)
      );
    }
  }
}

// config-tracker.service.ts
import { Injectable, KeyValueDiffers, KeyValueDiffer, signal, Signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ConfigTrackerService {
  private differ: KeyValueDiffer<string, unknown>;
  private _changedKeys = signal<string[]>([]);

  readonly changedKeys: Signal<string[]> = this._changedKeys.asReadonly();

  constructor(private differs: KeyValueDiffers) {
    this.differ = this.differs.find({}).create();
  }

  track(config: Record<string, unknown>): void {
    const changes = this.differ.diff(config);
    if (changes) {
      const keys: string[] = [];
      changes.forEachChangedItem(r => keys.push(String(r.key)));
      changes.forEachAddedItem(r => keys.push(String(r.key)));
      if (keys.length > 0) {
        this._changedKeys.set(keys);
      }
    }
  }
}

// demo.component.ts
import { Component, DoCheck, inject } from '@angular/core';
import { TrackChangesDirective } from './track-changes.directive';
import { ConfigTrackerService } from './config-tracker.service';

@Component({
  selector: 'app-demo',
  standalone: true,
  imports: [TrackChangesDirective],
  template: `
    <ul [appTrackChanges]="fruits">
      @for (fruit of fruits; track fruit) {
        <li>{{ fruit }}</li>
      }
    </ul>
    <button (click)="addFruit()">Frucht hinzufügen</button>
    <button (click)="removeLast()">Letzte entfernen</button>

    <hr />
    <button (click)="changeConfig()">Konfiguration ändern</button>
    <p>Geänderte Keys: {{ configTracker.changedKeys() | json }}</p>
  `
})
export class DemoComponent implements DoCheck {
  configTracker = inject(ConfigTrackerService);

  fruits = ['Apfel', 'Banane', 'Kirsche'];
  config: Record<string, unknown> = { theme: 'light', lang: 'de', version: 1 };

  addFruit(): void {
    this.fruits = [...this.fruits, `Frucht ${this.fruits.length + 1}`];
  }

  removeLast(): void {
    this.fruits = this.fruits.slice(0, -1);
  }

  changeConfig(): void {
    this.config = { ...this.config, theme: this.config['theme'] === 'light' ? 'dark' : 'light' };
  }

  ngDoCheck(): void {
    this.configTracker.track(this.config);
  }
}
```

## Weiterführendes

- **`IterableChangeRecord`** hat neben `item`, `currentIndex` und `previousIndex` auch `trackById`, das greift, wenn du ein Custom-`TrackByFunction` übergibst – ideal für komplexe Objekte.
- Angular nutzt intern `DefaultIterableDiffer` (für Arrays) und `DefaultKeyValueDiffer`. Du kannst eigene Differ-Implementierungen registrieren, indem du `ITERABLE_DIFFERS` als Multi-Provider bereitstellst – für exotische Collection-Typen wie Immutable.js oder eigene Datenstrukturen.
- Offizielle Doku: [IterableDiffers](https://angular.dev/api/core/IterableDiffers) und [KeyValueDiffers](https://angular.dev/api/core/KeyValueDiffers)
