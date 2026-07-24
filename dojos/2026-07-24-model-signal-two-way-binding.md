# Angular Dojo: model() – Signal-basierte bidirektionale Datenbindung
**Datum:** 2026-07-24
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, `model()` aus Angular 17.2 einzusetzen, um in Kind-Komponenten schreibbare Signale zu deklarieren, die bidirektionale Datenbindung ohne `@Input()` + `@Output() EventEmitter` ermöglichen.

## Hintergrund & Theorie

Mit Angular 17.2 wurde `model()` als neues Signal-Primitiv eingeführt. Es ersetzt das klassische Muster aus `@Input() value` + `@Output() valueChange = new EventEmitter()` durch eine einzige, typsichere Deklaration.

```typescript
// Klassisches Muster (alt)
@Input() value = '';
@Output() valueChange = new EventEmitter<string>();

// model()-Muster (neu)
value = model('');          // optional, Startwert ''
value = model.required<string>(); // required, kein Defaultwert
```

`model()` gibt ein `ModelSignal<T>` zurück – ein Objekt, das gleichzeitig:
- **lesbares Signal** ist (`value()` → `T`)
- **schreibbares Signal** ist (`value.set(...)`, `value.update(...)`)
- **automatisch ein Change-Event** emittiert (`valueChange`), wenn der Wert geschrieben wird

Im Parent-Template wird `[(value)]` verwendet – syntaktischer Zucker für `[value]="x" (valueChange)="x = $event"`.

**Wichtige Regeln:**
- `model()` ist ausschließlich für Komponenten/Direktiven gedacht, nicht für Services
- Der Elternteil kann auch nur einseitig binden: `[value]="x"` ohne `(valueChange)`
- Beim Schreiben via `.set()` oder `.update()` in der Kind-Komponente wird der neue Wert automatisch nach oben propagiert
- `model.required<T>()` erzeugt ein pflichtiges Two-Way-Binding (kein Defaultwert)

## Aufgabe

Erstelle eine wiederverwendbare `RatingComponent`, die eine Sternbewertung (1–5) anzeigt und bearbeitet. Die Komponente soll `model()` für das bidirektionale Binding verwenden und von einer `AppComponent` aus gesteuert werden.

### Schritte

1. Erstelle `RatingComponent` als Standalone Component mit einem `rating = model.required<number>()` Signal
2. Rendere 5 anklickbare Sterne im Template (`★` / `☆`) basierend auf dem aktuellen `rating()`-Wert
3. Beim Klick auf einen Stern: schreibe den neuen Wert via `rating.set(n)` – das löst automatisch das Change-Event aus
4. Nutze in `AppComponent` die `RatingComponent` mit `[(rating)]="myRating"`, wobei `myRating` ein Signal ist
5. Zeige in `AppComponent` die aktuelle Bewertung an und stelle einen Reset-Button bereit, der `myRating.set(0)` aufruft
6. **Bonus:** Füge ein zweites `readonly = model(false)` Signal hinzu, das Klicks auf den Sternen deaktiviert

## Hints

<details>
<summary>Hint 1 – Grundstruktur von model()</summary>

```typescript
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-rating',
  standalone: true,
  template: `...`,
})
export class RatingComponent {
  // Pflichtiges Two-Way-Binding:
  rating = model.required<number>();

  // Oder optional mit Startwert:
  // rating = model(0);

  setRating(n: number): void {
    this.rating.set(n); // propagiert automatisch an den Parent
  }
}
```

Im Template der Parent-Komponente:
```html
<app-rating [(rating)]="myRating" />
```

`myRating` kann ein `signal(0)` oder ein normales Property sein.
</details>

<details>
<summary>Hint 2 – Sterne-Template mit @for</summary>

```typescript
template: `
  <span class="stars">
    @for (star of stars; track star) {
      <button (click)="setRating(star)">
        {{ star <= rating() ? '★' : '☆' }}
      </button>
    }
  </span>
`,

// Im Component:
stars = [1, 2, 3, 4, 5];
```

Mit `track star` wird die Performance-Optimierung sichergestellt; `star <= rating()` vergleicht gegen den Signalwert.
</details>

<details>
<summary>Hint 3 – Bonus: readonly mit model()</summary>

```typescript
readonly = model(false);

// Template:
@for (star of stars; track star) {
  <button
    (click)="!readonly() && setRating(star)"
    [disabled]="readonly()"
  >
    {{ star <= rating() ? '★' : '☆' }}
  </button>
}
```

Im Parent:
```html
<app-rating [(rating)]="myRating" [readonly]="isReadonly()" />
```

`readonly` ohne `(readonlyChange)` ist ein einseitiges Binding – der Parent übergibt nur, die Kind-Komponente schreibt nicht zurück.
</details>

## Beispiellösung

```typescript
// rating.component.ts
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-rating',
  standalone: true,
  styles: [`
    button { background: none; border: none; font-size: 1.5rem; cursor: pointer; }
    button:disabled { cursor: default; opacity: 0.5; }
  `],
  template: `
    <span role="group" aria-label="Sternebewertung">
      @for (star of stars; track star) {
        <button
          (click)="setRating(star)"
          [disabled]="readonly()"
          [attr.aria-label]="star + ' Stern' + (star > 1 ? 'e' : '')"
        >
          {{ star <= rating() ? '★' : '☆' }}
        </button>
      }
    </span>
    <span>{{ rating() }}/5</span>
  `,
})
export class RatingComponent {
  rating = model.required<number>();
  readonly = model(false);

  stars = [1, 2, 3, 4, 5];

  setRating(n: number): void {
    if (!this.readonly()) {
      this.rating.set(n);
    }
  }
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
    <h2>Produkt bewerten</h2>

    <app-rating [(rating)]="myRating" />

    <p>Deine Bewertung: {{ myRating() === 0 ? 'keine' : myRating() + ' Stern(e)' }}</p>

    <button (click)="myRating.set(0)">Zurücksetzen</button>

    <hr />

    <h3>Nur-Lesen-Vorschau</h3>
    <app-rating [(rating)]="myRating" [readonly]="true" />
  `,
})
export class AppComponent {
  myRating = signal(0);
}
```

**Warum `model()` statt `@Input()` + `@Output()`?**

```typescript
// Vorher: 3 Zeilen, Namenskonvention muss stimmen (valueChange!)
@Input() rating = 0;
@Output() ratingChange = new EventEmitter<number>();
// Im Template: this.ratingChange.emit(n) -- nie vergessen!

// Nachher: 1 Zeile, typsicher, kein EventEmitter
rating = model.required<number>();
// Im Template: this.rating.set(n) -- automatisches Emit
```

## Weiterführendes
- Kombiniere `model()` mit `computed()`: `displayLabel = computed(() => this.rating() === 5 ? '🏆 Perfekt!' : '')` – das computed-Signal reagiert automatisch auf Änderungen von außen und innen
- `model()` gibt ein `ModelSignal<T>` zurück, das `InputSignal<T>` erweitert – du kannst es mit `isSignal()` prüfen und im Test mit `fixture.componentInstance.rating()` lesen
- Für komplexe Formulare: `model()` ist kein Ersatz für `ControlValueAccessor`, da es kein `FormControl` kennt – nutze beide Patterns situationsgerecht
- [Angular Docs – model()](https://angular.dev/guide/signals/model)
- [Angular Blog – Two-way binding with signals](https://blog.angular.dev/signal-two-way-bindings-are-here-with-angular-17-2-53ee59be8f3e)
