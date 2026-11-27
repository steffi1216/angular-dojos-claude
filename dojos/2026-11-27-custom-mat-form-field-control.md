# Angular Dojo: Custom MatFormFieldControl
**Datum:** 2026-11-27
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du ein eigenes Angular-Formularfeld-Widget als `MatFormFieldControl<T>` implementierst, das nahtlos mit `<mat-form-field>` zusammenarbeitet – inklusive Floating-Label, Error-State, Required-Marker und Hint-Text.

## Hintergrund & Theorie

Angular Material's `<mat-form-field>` ist ein mächtiges Layout-Wrapper-Element: Es rendert das Label, Unterzeile (Hint/Error), Outline oder Underline und den Required-Marker. Damit ein eigenes Control in diesem Wrapper funktioniert, muss es das Interface `MatFormFieldControl<T>` implementieren und sich via `MAT_FORM_FIELD_CONTROL` als Provider registrieren.

Das Interface verlangt:
- **`value`** – der aktuelle Wert (Getter/Setter)
- **`stateChanges`** – ein `Subject<void>`, das bei jeder Zustandsänderung feuert, damit `<mat-form-field>` ein `ChangeDetection`-Zyklus auslösen kann
- **`id`** – eine eindeutige HTML-ID
- **`placeholder`** – angezeigter Placeholder-Text
- **`ngControl`** – Referenz zum `NgControl` (falls vorhanden)
- **`focused`**, **`empty`**, **`shouldLabelFloat`**, **`required`**, **`disabled`**, **`errorState`** – verschiedene Zustände
- **`onContainerClick()`** – wird aufgerufen, wenn der Nutzer auf den Form-Field-Container klickt

Durch `implements ControlValueAccessor` und Registrierung als `NG_VALUE_ACCESSOR` kann das eigene Control auch in `[(ngModel)]` und Reactive Forms verwendet werden.

## Aufgabe

Erstelle eine wiederverwendbare Komponente `PhoneInputComponent`, die eine Telefonnummer als strukturierten Wert (`{ countryCode: string; number: string }`) behandelt, sich als `MatFormFieldControl` registriert und korrekt mit `<mat-form-field>` und Reactive Forms zusammenarbeitet.

### Schritte

1. **Datenmodell** anlegen: Erstelle ein Interface `PhoneNumber` mit `countryCode: string` und `number: string`.

2. **Komponente** erstellen mit zwei separaten `<input>`-Elementen für `countryCode` und `number`. Implementiere `MatFormFieldControl<PhoneNumber | null>` und `ControlValueAccessor`.

3. **Pflicht-Properties** implementieren:
   - `stateChanges = new Subject<void>()`
   - eindeutige `id` per statischem Counter
   - `ngControl` via `inject(NgControl)` (mit `self: true, optional: true`)
   - Setter/Getter für `value`, `disabled`, `required`, `placeholder`
   - `focused`, `empty`, `shouldLabelFloat`, `errorState` korrekt ableiten

4. **Provider** registrieren:
   ```typescript
   providers: [
     { provide: MatFormFieldControl, useExisting: PhoneInputComponent },
     { provide: NG_VALUE_ACCESSOR, useExisting: PhoneInputComponent, multi: true },
   ]
   ```

5. **Einbetten** in einem Demo-Formular:
   ```html
   <mat-form-field appearance="outline">
     <mat-label>Telefon</mat-label>
     <app-phone-input formControlName="phone"></app-phone-input>
     <mat-error *ngIf="form.get('phone')?.hasError('required')">Pflichtfeld</mat-error>
   </mat-form-field>
   ```

6. Teste `shouldLabelFloat` (soll `true` sein, wenn focused ODER nicht leer), `empty` (beide Felder leer) und `errorState` (Validierungsfehler + touched).

## Hints

<details>
<summary>Hint 1 – ngControl und ErrorStateMatcher</summary>

Injiziere `NgControl` mit `self: true` und `optional: true`. Falls vorhanden, setze `this.ngControl.valueAccessor = this` im Konstruktor, damit kein zirkulärer Provider entsteht. Für `errorState` gilt die Faustregel: `!!(this.ngControl?.invalid && this.ngControl?.touched)`.

```typescript
constructor() {
  const ngControl = inject(NgControl, { self: true, optional: true });
  if (ngControl) {
    this.ngControl = ngControl;
    ngControl.valueAccessor = this;
  }
}
```
</details>

<details>
<summary>Hint 2 – stateChanges korrekt auslösen</summary>

Jede Zustandsänderung (value, focus, disabled, …) muss `this.stateChanges.next()` aufrufen, damit Angular Material die Darstellung aktualisiert. Vergiss nicht, `stateChanges.complete()` im `ngOnDestroy` zu rufen. Nutze dafür `DestroyRef`:

```typescript
inject(DestroyRef).onDestroy(() => this.stateChanges.complete());
```
</details>

<details>
<summary>Hint 3 – FocusMonitor für korrekten focused-State</summary>

Der Angular CDK `FocusMonitor` ist ideal, um den Fokus auf dem Host-Element zu überwachen:

```typescript
const fm = inject(FocusMonitor);
const el = inject(ElementRef);
fm.monitor(el, true).pipe(takeUntilDestroyed()).subscribe(origin => {
  this.focused = !!origin;
  this.stateChanges.next();
});
```
</details>

## Beispiellösung

```typescript
import {
  Component, ElementRef, inject, DestroyRef, signal,
  OnInit, HostBinding, booleanAttribute,
} from '@angular/core';
import {
  MatFormFieldControl, MAT_FORM_FIELD,
} from '@angular/material/form-field';
import {
  ControlValueAccessor, NG_VALUE_ACCESSOR, NgControl,
  ReactiveFormsModule, FormBuilder, Validators,
} from '@angular/forms';
import { FocusMonitor } from '@angular/cdk/a11y';
import { Subject } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { CommonModule } from '@angular/common';
import { MatInputModule } from '@angular/material/input';

export interface PhoneNumber {
  countryCode: string;
  number: string;
}

let nextId = 0;

@Component({
  selector: 'app-phone-input',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, MatInputModule],
  template: `
    <div class="phone-container" [formGroup]="parts">
      <input formControlName="countryCode" placeholder="+41" size="3"
             (blur)="_onTouched()" (input)="_handleInput()" />
      <span>&nbsp;</span>
      <input formControlName="number" placeholder="79 123 45 67"
             (blur)="_onTouched()" (input)="_handleInput()" />
    </div>
  `,
  styles: [`.phone-container { display: flex; gap: 4px; }`],
  providers: [
    { provide: MatFormFieldControl, useExisting: PhoneInputComponent },
    { provide: NG_VALUE_ACCESSOR, useExisting: PhoneInputComponent, multi: true },
  ],
})
export class PhoneInputComponent
  implements MatFormFieldControl<PhoneNumber | null>, ControlValueAccessor, OnInit
{
  static idCounter = 0;

  readonly stateChanges = new Subject<void>();
  readonly id = `phone-input-${nextId++}`;
  readonly controlType = 'phone-input';

  @HostBinding('attr.aria-describedby') userAriaDescribedBy = '';

  focused = false;
  touched = false;
  ngControl: NgControl | null = null;

  private _value: PhoneNumber | null = null;
  private _placeholder = '';
  private _required = false;
  private _disabled = false;
  private _onChange = (_: PhoneNumber | null) => {};
  _onTouched = () => {};

  readonly parts = inject(FormBuilder).group({
    countryCode: [''],
    number: [''],
  });

  constructor() {
    const nc = inject(NgControl, { self: true, optional: true });
    if (nc) {
      this.ngControl = nc;
      nc.valueAccessor = this;
    }
    inject(DestroyRef).onDestroy(() => this.stateChanges.complete());

    const fm = inject(FocusMonitor);
    const el = inject(ElementRef<HTMLElement>);
    fm.monitor(el, true).pipe(takeUntilDestroyed()).subscribe(origin => {
      this.focused = !!origin;
      this.stateChanges.next();
    });
  }

  ngOnInit() {
    this.parts.valueChanges.pipe(takeUntilDestroyed()).subscribe(() => {
      this._handleInput();
    });
  }

  get value(): PhoneNumber | null { return this._value; }
  set value(val: PhoneNumber | null) {
    this._value = val;
    this.parts.setValue({ countryCode: val?.countryCode ?? '', number: val?.number ?? '' }, { emitEvent: false });
    this.stateChanges.next();
  }

  get placeholder() { return this._placeholder; }
  set placeholder(p: string) { this._placeholder = p; this.stateChanges.next(); }

  get required() { return this._required; }
  set required(r: boolean) { this._required = booleanAttribute(r); this.stateChanges.next(); }

  get disabled() { return this._disabled; }
  set disabled(d: boolean) {
    this._disabled = d;
    d ? this.parts.disable() : this.parts.enable();
    this.stateChanges.next();
  }

  get empty() {
    const { countryCode, number } = this.parts.value;
    return !countryCode && !number;
  }

  get shouldLabelFloat() { return this.focused || !this.empty; }

  get errorState() {
    return !!(this.ngControl?.invalid && (this.ngControl.touched || this.touched));
  }

  setDescribedByIds(ids: string[]) { this.userAriaDescribedBy = ids.join(' '); }
  onContainerClick() { /* focus first input */ }

  writeValue(val: PhoneNumber | null): void { this.value = val; }
  registerOnChange(fn: (v: PhoneNumber | null) => void): void { this._onChange = fn; }
  registerOnTouched(fn: () => void): void { this._onTouched = fn; }
  setDisabledState(disabled: boolean): void { this.disabled = disabled; }

  _handleInput(): void {
    const { countryCode, number } = this.parts.value;
    this._value = countryCode || number
      ? { countryCode: countryCode ?? '', number: number ?? '' }
      : null;
    this._onChange(this._value);
    this.stateChanges.next();
  }
}

// Demo-Nutzung:
@Component({
  selector: 'app-phone-demo',
  standalone: true,
  imports: [ReactiveFormsModule, MatFormFieldModule, MatInputModule, PhoneInputComponent, CommonModule],
  template: `
    <form [formGroup]="form">
      <mat-form-field appearance="outline">
        <mat-label>Telefonnummer</mat-label>
        <app-phone-input formControlName="phone"></app-phone-input>
        <mat-hint>Format: +41 79 123 45 67</mat-hint>
        <mat-error>Pflichtfeld</mat-error>
      </mat-form-field>
      <p>Wert: {{ form.value | json }}</p>
    </form>
  `,
})
export class PhoneDemoComponent {
  form = inject(FormBuilder).group({ phone: [null, Validators.required] });
}
```

## Weiterführendes
- [Angular Material – Creating a custom form field control](https://material.angular.io/guide/creating-a-custom-form-field-control): Die offizielle Dokumentation zeigt weitere Feinheiten wie `MatFormField`-Container-Interaktion und `setDescribedByIds`.
- Für noch tiefere Integration mit Error-States empfiehlt sich ein eigener `ErrorStateMatcher`, der via `MAT_ERROR_STATE_MATCHER` bereitgestellt werden kann.
- Mit dem neuen Signal-basierten Ansatz lässt sich `stateChanges` durch ein `effect()` ersetzen, das alle relevanten Signals beobachtet – ein spannender Refactoring-Pfad für Angular 18+.
