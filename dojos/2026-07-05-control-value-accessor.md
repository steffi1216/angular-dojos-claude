# Angular Dojo: Custom Form Controls mit ControlValueAccessor
**Datum:** 2026-07-05
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, eigene Angular-Komponenten als vollwertige Form-Controls zu implementieren, indem du das `ControlValueAccessor`-Interface verwendest – so integrieren sie sich nahtlos in Reactive Forms und Template-driven Forms.

## Hintergrund & Theorie

Angular's Forms-System kommuniziert mit nativen HTML-Controls (wie `<input>`, `<select>`) über eine interne Brücke: den `ControlValueAccessor`. Jedes native Element hat bereits eine eingebaute Implementierung. Wenn du eine eigene Komponente (z. B. ein Rating-Widget, ein Datepicker oder ein Multi-Select) als Form-Control verwenden willst, musst du dieses Interface selbst implementieren.

Das Interface besteht aus vier Methoden:
- **`writeValue(value)`** – Angular → Komponente: Übergibt den Wert aus dem Form-Model an deine Komponente.
- **`registerOnChange(fn)`** – registriert den Callback, den du aufrufen musst, wenn der Wert in deiner Komponente ändert.
- **`registerOnTouched(fn)`** – registriert den Callback für `touched`-Zustand (z. B. bei `blur`).
- **`setDisabledState(isDisabled)`** – optional: wird aufgerufen, wenn das Control aktiviert/deaktiviert wird.

Die Komponente muss sich außerdem als `NG_VALUE_ACCESSOR` Provider registrieren, damit Angular sie als Form-Control erkennt.

Seit Angular 14+ kann man `ControlValueAccessor` auch mit `host`-Bindings und `inject(NgControl)` eleganter implementieren. In neueren Angular-Versionen (16+) lässt sich alles auch mit Signals kombinieren.

## Aufgabe

Erstelle eine **Star-Rating-Komponente** (`<app-star-rating>`), die als vollwertiges Form-Control funktioniert. Sie soll:
- 1–5 Sterne anzeigen (klickbar)
- Den gewählten Wert via `ControlValueAccessor` an ein `FormControl` melden
- Bei `disabled`-Zustand nicht klickbar sein
- Mit `[formControl]` und `[(ngModel)]` verwendbar sein

### Schritte

1. **Komponente anlegen** – Erstelle `star-rating.component.ts` als Standalone Component mit einem Array `stars = [1, 2, 3, 4, 5]` und einem internen State `value = 0`.

2. **ControlValueAccessor implementieren** – Implementiere das Interface mit `writeValue`, `registerOnChange`, `registerOnTouched` und `setDisabledState`. Speichere die Callbacks in privaten Klassenvariablen.

3. **Provider registrieren** – Füge in den `providers` der Komponente folgenden Eintrag hinzu:
   ```typescript
   {
     provide: NG_VALUE_ACCESSOR,
     useExisting: forwardRef(() => StarRatingComponent),
     multi: true
   }
   ```

4. **Template bauen** – Rendere 5 Sterne-Buttons. Ein Stern ist gefüllt (`★`), wenn sein Index ≤ `value`. Bei Klick rufst du `onChange(newValue)` auf und setzt `value`.

5. **In AppComponent einbinden** – Verwende die Komponente sowohl mit `[formControl]` (Reactive Forms) als auch mit `[(ngModel)]` (Template-driven), um beide Wege zu testen.

## Hints

<details>
<summary>Hint 1 – Grundgerüst des Interfaces</summary>

```typescript
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

export class StarRatingComponent implements ControlValueAccessor {
  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(value: number): void {
    this.value = value ?? 0;
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }

  select(star: number): void {
    if (this.disabled) return;
    this.value = star;
    this.onChange(star);
    this.onTouched();
  }
}
```
</details>

<details>
<summary>Hint 2 – Provider-Registrierung & Template</summary>

```typescript
@Component({
  selector: 'app-star-rating',
  standalone: true,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true
    }
  ],
  template: `
    <span
      *ngFor="let star of stars"
      (click)="select(star)"
      [style.cursor]="disabled ? 'not-allowed' : 'pointer'"
      [style.opacity]="disabled ? 0.5 : 1"
      [style.fontSize.px]="28"
    >
      {{ star <= value ? '★' : '☆' }}
    </span>
  `
})
```

Verwendung in AppComponent:
```typescript
// Reactive Forms
rating = new FormControl(3);

// Template-driven
ngModelRating = 4;
```

```html
<!-- Reactive Forms -->
<app-star-rating [formControl]="rating" />
<p>Wert: {{ rating.value }}</p>

<!-- Template-driven -->
<app-star-rating [(ngModel)]="ngModelRating" />
<p>Wert: {{ ngModelRating }}</p>
```
</details>

## Beispiellösung

```typescript
// star-rating.component.ts
import {
  Component, forwardRef, signal, computed
} from '@angular/core';
import {
  ControlValueAccessor, NG_VALUE_ACCESSOR, ReactiveFormsModule, FormsModule
} from '@angular/forms';
import { NgFor } from '@angular/common';

@Component({
  selector: 'app-star-rating',
  standalone: true,
  imports: [NgFor],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true
    }
  ],
  template: `
    <span
      *ngFor="let star of stars"
      (click)="select(star)"
      [attr.aria-label]="'Bewertung ' + star + ' von 5'"
      [style.cursor]="disabled() ? 'not-allowed' : 'pointer'"
      [style.opacity]="disabled() ? 0.5 : 1"
      [style.fontSize.px]="28"
    >
      {{ star <= value() ? '★' : '☆' }}
    </span>
  `
})
export class StarRatingComponent implements ControlValueAccessor {
  stars = [1, 2, 3, 4, 5];

  value = signal(0);
  disabled = signal(false);

  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  select(star: number): void {
    if (this.disabled()) return;
    this.value.set(star);
    this.onChange(star);
    this.onTouched();
  }

  writeValue(value: number): void {
    this.value.set(value ?? 0);
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled.set(isDisabled);
  }
}
```

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { FormControl, ReactiveFormsModule, FormsModule } from '@angular/forms';
import { StarRatingComponent } from './star-rating.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [StarRatingComponent, ReactiveFormsModule, FormsModule],
  template: `
    <h2>Reactive Forms</h2>
    <app-star-rating [formControl]="reactiveRating" />
    <p>Wert: {{ reactiveRating.value }} | Touched: {{ reactiveRating.touched }}</p>
    <button (click)="reactiveRating.disable()">Disable</button>
    <button (click)="reactiveRating.enable()">Enable</button>

    <h2>Template-driven Forms</h2>
    <app-star-rating [(ngModel)]="ngModelRating" />
    <p>Wert: {{ ngModelRating }}</p>
  `
})
export class AppComponent {
  reactiveRating = new FormControl(3);
  ngModelRating = 4;
}
```

## Weiterführendes

- **NgControl injizieren** – Mit `inject(NgControl)` und `ngControl.valueAccessor = this` kannst du `forwardRef` und den expliziten Provider-Eintrag vermeiden (eleganter Ansatz ab Angular 14+).
- **AbstractControl-Validierung** – Kombiniere `ControlValueAccessor` mit `Validator`-Interface, um Custom-Validation direkt in die Komponente einzubauen.
- **Offizielle Docs:** [angular.dev/guide/forms/control-value-accessor](https://angular.dev/guide/forms/control-value-accessor)
- **Tipp:** Bibliotheken wie `ngxtension` bieten `provideValueAccessor()`-Helfer, die den Boilerplate weiter reduzieren.
