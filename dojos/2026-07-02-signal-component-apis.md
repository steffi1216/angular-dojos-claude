# Angular Dojo: Signal-basierte Component APIs (input, output, model)
**Datum:** 2026-07-02
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Lerne die neuen signal-basierten Component-APIs `input()`, `output()` und `model()` kennen, die seit Angular 17 die klassischen `@Input()`/`@Output()`-Dekoratoren ersetzen und die reaktive Programmierung mit Signals nahtlos integrieren.

## Hintergrund & Theorie

Mit Angular 17 wurden funktionale Alternativen zu `@Input()` und `@Output()` eingeführt, die direkt mit dem Signal-System integriert sind:

**`input<T>()`** – Erzeugt ein readonly `InputSignal`. Der Wert ist automatisch reaktiv und kann in `computed()` und `effect()` verwendet werden. Mit `input.required<T>()` wird ein Pflichtparameter deklariert.

**`output<T>()`** – Erzeugt ein `OutputEmitterRef`. Kein Signal, aber eine typsichere Alternative zu `EventEmitter`. Emittiert Werte mit `.emit()`.

**`model<T>()`** – Kombination aus `input` und `output` für Two-Way-Binding. Erzeugt ein schreibbares `ModelSignal`, das intern sowohl ein Input als auch ein `Change`-Output bereitstellt. Damit funktioniert `[(value)]`-Binding automatisch.

Diese neuen APIs erlauben es, Komponenten vollständig ohne Zone.js zu betreiben (zoneless), da Änderungen durch das Signal-System erkannt werden statt durch `ngOnChanges` oder Dirty-Checking. Zudem sind die Typen präziser: `input.required()` erzwingt zur Compile-Zeit, dass ein Wert übergeben wird.

```typescript
// Alt
@Input() value: string = '';
@Output() valueChange = new EventEmitter<string>();

// Neu
value = model<string>('');
```

## Aufgabe

Erstelle eine wiederverwendbare `RatingComponent`, die eine Sternebewertung (1–5) darstellt. Die Komponente soll:

1. Die aktuelle Bewertung über `model<number>()` empfangen und zurücksenden (Two-Way-Binding)
2. Eine optionale `label`-Eigenschaft über `input<string>()` mit Standardwert `'Bewertung'` haben
3. Eine Pflicht-Eigenschaft `maxStars` über `input.required<number>()` anfordern
4. Ein `hovered`-Signal intern verwalten, um Hover-Feedback anzuzeigen
5. In einer Parent-Komponente mit `[(rating)]` verwendet werden

### Schritte

1. Generiere eine neue Standalone-Komponente `RatingComponent` und ersetze `@Input()`/`@Output()` durch die signal-basierten APIs.
2. Implementiere die Template-Logik mit `@for` und nutze das `model()`-Signal direkt im Template (lesend und schreibend via `.set()`).
3. Füge ein internes `hovered` Signal (WritableSignal) hinzu und nutze `computed()`, um die anzuzeigende Anzahl gefüllter Sterne zu berechnen.
4. Erstelle eine `AppComponent`, die `RatingComponent` einbindet und den aktuellen Wert per `[(rating)]` synchronisiert und anzeigt.
5. Verifiziere, dass der Typ-Compiler bei fehlendem `maxStars` einen Fehler wirft (kein Default-Wert bei `input.required()`).

## Hints

<details>
<summary>Hint 1 – model() und Two-Way-Binding</summary>

`model<T>()` erzeugt ein `ModelSignal<T>`. Um den Wert zu setzen, rufe `.set()` auf – das löst automatisch auch das Output-Event aus, sodass das Parent-Binding aktualisiert wird:

```typescript
rating = model<number>(0);

// Im Template oder in einer Methode:
this.rating.set(newValue);
```

Im Parent-Template:
```html
<app-rating [(rating)]="myRating" [maxStars]="5" />
```

Dabei muss `myRating` im Parent ein `WritableSignal<number>` oder eine normale Variable sein.

</details>

<details>
<summary>Hint 2 – input.required() und computed()</summary>

`input.required<number>()` hat keinen Standardwert. Angular wirft einen Laufzeit- und Compile-Fehler, wenn der Wert nicht gebunden wird.

Nutze `computed()`, um die Anzeige-Logik zu zentralisieren:

```typescript
maxStars = input.required<number>();
rating = model<number>(0);
hovered = signal<number>(0);

displayStars = computed(() =>
  this.hovered() > 0 ? this.hovered() : this.rating()
);
```

Im Template kannst du dann `displayStars()` für die visuelle Darstellung nutzen, ohne die Logik ins Template zu verlagern.

</details>

<details>
<summary>Hint 3 – Template mit @for und Sterndarstellung</summary>

Nutze `@for` mit einer `Array.from()`-Hilfsfunktion oder erzeuge im Component ein Array für die Iteration:

```typescript
stars = computed(() => Array.from({ length: this.maxStars() }, (_, i) => i + 1));
```

Im Template:
```html
@for (star of stars(); track star) {
  <span
    (click)="rating.set(star)"
    (mouseenter)="hovered.set(star)"
    (mouseleave)="hovered.set(0)"
  >
    {{ star <= displayStars() ? '★' : '☆' }}
  </span>
}
```

</details>

## Beispiellösung

```typescript
// rating.component.ts
import { Component, computed, input, model, signal } from '@angular/core';

@Component({
  selector: 'app-rating',
  standalone: true,
  template: `
    <div class="rating">
      <span class="label">{{ label() }}</span>
      @for (star of stars(); track star) {
        <span
          class="star"
          [class.filled]="star <= displayStars()"
          (click)="rating.set(star)"
          (mouseenter)="hovered.set(star)"
          (mouseleave)="hovered.set(0)"
        >
          {{ star <= displayStars() ? '★' : '☆' }}
        </span>
      }
      <span class="value">({{ rating() }}/{{ maxStars() }})</span>
    </div>
  `,
  styles: [`
    .rating { display: flex; align-items: center; gap: 4px; font-size: 1.5rem; }
    .star { cursor: pointer; color: #ccc; transition: color 0.15s; }
    .star.filled { color: gold; }
    .label { font-size: 1rem; margin-right: 8px; }
    .value { font-size: 0.9rem; color: #666; margin-left: 8px; }
  `],
})
export class RatingComponent {
  label = input<string>('Bewertung');
  maxStars = input.required<number>();
  rating = model<number>(0);

  hovered = signal<number>(0);

  stars = computed(() =>
    Array.from({ length: this.maxStars() }, (_, i) => i + 1)
  );

  displayStars = computed(() =>
    this.hovered() > 0 ? this.hovered() : this.rating()
  );
}
```

```typescript
// app.component.ts
import { Component, signal } from '@angular/core';
import { RatingComponent } from './rating.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RatingComponent],
  template: `
    <h2>Film bewerten</h2>
    <app-rating
      label="Deine Bewertung"
      [maxStars]="5"
      [(rating)]="filmRating"
    />
    <p>Aktuell gewählt: {{ filmRating() }} Stern(e)</p>

    <h2>Schwierigkeitsgrad</h2>
    <app-rating
      [maxStars]="3"
      [(rating)]="difficulty"
    />
    <p>Schwierigkeit: {{ difficulty() }}/3</p>
  `,
})
export class AppComponent {
  filmRating = signal<number>(0);
  difficulty = signal<number>(2);
}
```

### Warum ist das besser als @Input()/@Output()?

| Aspekt | Klassisch | Signal-API |
|---|---|---|
| Reaktivität | `ngOnChanges` nötig | direkt in `computed()`/`effect()` nutzbar |
| Two-Way | `@Input()` + `@Output(xxxChange)` manuell | `model()` automatisch |
| Pflichtfelder | Laufzeitfehler erst bei Nutzung | Compile-Zeit-Fehler mit `input.required()` |
| Zoneless | Nicht möglich | Vollständig kompatibel |

## Weiterführendes

- **`linkedSignal()`** (Angular 18+): Erzeugt ein WritableSignal, das sich automatisch aktualisiert, wenn ein Quell-Signal sich ändert – perfekt für abhängige Formularwerte.
- **`toSignal()` / `toObservable()`** aus `@angular/core/rxjs-interop`: Verbindet die neuen Signal-Inputs mit existierenden RxJS-Streams.
- **Offizielle Docs**: [angular.dev/guide/signals/inputs](https://angular.dev/guide/signals/inputs)
- **Tipp**: Kombiniere `model()` mit `effect()`, um Seiteneffekte (z.B. Local Storage Sync) automatisch auszulösen, wenn der Wert sich ändert – ohne ein einziges `ngOnChanges`.
