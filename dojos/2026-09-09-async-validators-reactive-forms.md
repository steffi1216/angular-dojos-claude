# Angular Dojo: Async Validators in Reactive Forms
**Datum:** 2026-09-09
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie man asynchrone Validatoren in Angular Reactive Forms implementiert – z. B. um Benutzernamen gegen eine API zu prüfen – und verstehst dabei das Zusammenspiel von `AsyncValidatorFn`, `Observable`, `AbstractControl` und dem `pending`-Status.

## Hintergrund & Theorie

Synchrone Validatoren (`ValidatorFn`) geben sofort einen Fehler oder `null` zurück. Für Prüfungen, die einen API-Call benötigen (z. B. „Ist dieser Username bereits vergeben?"), braucht man **asynchrone Validatoren** (`AsyncValidatorFn`).

Ein async Validator gibt ein `Observable<ValidationErrors | null>` (oder `Promise`) zurück. Angular setzt das betroffene Control in den Status `pending`, solange das Observable noch nicht completed hat. Erst danach wechselt der Status zu `valid` oder `invalid`.

Wichtige Punkte:
- Async Validators werden als **dritter Parameter** an `FormControl`, `FormGroup` oder `FormBuilder.control()` übergeben.
- Sie werden **nach** allen synchronen Validatoren ausgeführt – nur wenn diese alle bestehen.
- Ohne Debouncing feuert der Validator bei jedem Tastenanschlag. `debounceTime` + `switchMap` ist das Standardmuster, um API-Spam zu verhindern.
- `distinctUntilChanged()` verhindert redundante Calls bei gleichen Werten.
- Der Rückgabe-Observable muss **einmal emittieren und dann completen** – `take(1)` oder `first()` als Absicherung verwenden.

```typescript
// Minimales Beispiel
const asyncValidator: AsyncValidatorFn = (control: AbstractControl) => {
  return myService.checkUsername(control.value).pipe(
    map(isTaken => (isTaken ? { usernameTaken: true } : null))
  );
};
```

## Aufgabe

Baue eine Registrierungsseite mit einem Reactive Form, das eine **asynchrone Username-Validierung** gegen einen simulierten Backend-Service enthält.

### Anforderungen
- Das Form hat die Felder: `username` (async-validiert) und `email`
- Der `UsernameValidatorService` prüft, ob der Username bereits „vergeben" ist (simuliert mit einem `delay` + fester Liste)
- Der Validator nutzt `debounceTime(400)` und `switchMap`, um API-Calls zu drosseln
- Die UI zeigt:
  - einen Ladeindikator während `pending`
  - eine Fehlermeldung bei `usernameTaken`
  - ein grünes Häkchen bei freiem Username

### Schritte

1. **Service erstellen** – `UsernameValidatorService` mit einer Methode `isUsernameTaken(username: string): Observable<boolean>`, die aus einer Hardcoded-Liste (`['admin', 'test', 'angular']`) prüft und `delay(800)` simuliert.

2. **Async Validator Factory** – Erstelle eine Funktion `uniqueUsernameValidator(service: UsernameValidatorService): AsyncValidatorFn`, die:
   - `debounceTime(400)` und `distinctUntilChanged()` auf den Kontroll-Value-Changes anwendet
   - `switchMap` nutzt, um den Service aufzurufen
   - Bei positivem Befund `{ usernameTaken: true }` zurückgibt, sonst `null`
   - Mit `first()` abschließt

3. **Formular aufbauen** – Erstelle das `FormGroup` via `FormBuilder` und weise dem `username`-Control den async Validator als dritten Parameter zu. Injiziere den Service über `inject()`.

4. **Template** – Zeige Status-Indikatoren an:
   - `*ngIf="username.pending"` → Spinner-Text „Prüfe..."
   - `*ngIf="username.errors?.['usernameTaken']"` → Fehlermeldung
   - `*ngIf="username.valid"` → Erfolgs-Icon ✓

## Hints

<details>
<summary>Hint 1 – Async Validator mit debounce korrekt bauen</summary>

Der Validator erhält ein `AbstractControl`. Um `debounceTime` zu nutzen, musst du auf `control.valueChanges` lauschen, nicht direkt den Wert verarbeiten:

```typescript
export function uniqueUsernameValidator(
  service: UsernameValidatorService
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return control.valueChanges.pipe(
      startWith(control.value),
      debounceTime(400),
      distinctUntilChanged(),
      switchMap(value => service.isUsernameTaken(value)),
      map(isTaken => (isTaken ? { usernameTaken: true } : null)),
      first()
    );
  };
}
```

`startWith(control.value)` stellt sicher, dass auch beim initialen Wert validiert wird.

</details>

<details>
<summary>Hint 2 – Control im FormBuilder registrieren</summary>

```typescript
// Mit FormBuilder und inject()
private fb = inject(FormBuilder);
private usernameService = inject(UsernameValidatorService);

form = this.fb.group({
  username: [
    '',
    [Validators.required, Validators.minLength(3)],       // sync validators
    [uniqueUsernameValidator(this.usernameService)]        // async validators
  ],
  email: ['', [Validators.required, Validators.email]]
});

get username() {
  return this.form.get('username')!;
}
```

</details>

<details>
<summary>Hint 3 – Template mit pending-Status</summary>

```html
<div class="field">
  <input formControlName="username" placeholder="Benutzername" />

  @if (username.pending) {
    <span class="hint">Prüfe Verfügbarkeit...</span>
  }
  @if (username.errors?.['required'] && username.touched) {
    <span class="error">Pflichtfeld</span>
  }
  @if (username.errors?.['usernameTaken']) {
    <span class="error">Dieser Benutzername ist bereits vergeben.</span>
  }
  @if (username.valid && username.dirty) {
    <span class="success">✓ Verfügbar</span>
  }
</div>
```

</details>

## Beispiellösung

```typescript
// username-validator.service.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UsernameValidatorService {
  private takenUsernames = ['admin', 'test', 'angular', 'user123'];

  isUsernameTaken(username: string): Observable<boolean> {
    const isTaken = this.takenUsernames.includes(username.toLowerCase());
    return of(isTaken).pipe(delay(800));
  }
}
```

```typescript
// unique-username.validator.ts
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable } from 'rxjs';
import { debounceTime, distinctUntilChanged, first, map, startWith, switchMap } from 'rxjs/operators';
import { UsernameValidatorService } from './username-validator.service';

export function uniqueUsernameValidator(
  service: UsernameValidatorService
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return control.valueChanges.pipe(
      startWith(control.value),
      debounceTime(400),
      distinctUntilChanged(),
      switchMap(value => service.isUsernameTaken(value ?? '')),
      map(isTaken => (isTaken ? { usernameTaken: true } : null)),
      first()
    );
  };
}
```

```typescript
// register.component.ts
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { UsernameValidatorService } from './username-validator.service';
import { uniqueUsernameValidator } from './unique-username.validator';

@Component({
  selector: 'app-register',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div class="field">
        <label>Benutzername</label>
        <input formControlName="username" placeholder="z. B. johndoe" />
        @if (username.pending) {
          <span class="hint">Prüfe Verfügbarkeit...</span>
        }
        @if (username.errors?.['required'] && username.touched) {
          <span class="error">Pflichtfeld.</span>
        }
        @if (username.errors?.['minlength'] && username.touched) {
          <span class="error">Mind. 3 Zeichen.</span>
        }
        @if (username.errors?.['usernameTaken']) {
          <span class="error">Dieser Benutzername ist bereits vergeben.</span>
        }
        @if (username.valid && username.dirty) {
          <span class="success">✓ Verfügbar</span>
        }
      </div>

      <div class="field">
        <label>E-Mail</label>
        <input formControlName="email" type="email" placeholder="name@example.com" />
        @if (email.errors?.['email'] && email.touched) {
          <span class="error">Ungültige E-Mail-Adresse.</span>
        }
      </div>

      <button type="submit" [disabled]="form.invalid || form.pending">
        Registrieren
      </button>
    </form>
  `
})
export class RegisterComponent {
  private fb = inject(FormBuilder);
  private usernameService = inject(UsernameValidatorService);

  form = this.fb.group({
    username: [
      '',
      [Validators.required, Validators.minLength(3)],
      [uniqueUsernameValidator(this.usernameService)]
    ],
    email: ['', [Validators.required, Validators.email]]
  });

  get username() { return this.form.get('username')!; }
  get email() { return this.form.get('email')!; }

  onSubmit() {
    if (this.form.valid) {
      console.log('Formular abgesendet:', this.form.value);
    }
  }
}
```

## Weiterführendes

- **Cross-Field Async Validators**: Async Validatoren lassen sich auch auf `FormGroup`-Ebene setzen, um mehrere Felder gemeinsam zu prüfen (z. B. ob Username + E-Mail-Kombination existiert).
- **`updateOn: 'blur'`**: Wer Debouncing vermeiden will, kann stattdessen den Validator nur bei `blur` auslösen: `new FormControl('', { asyncValidators: [...], updateOn: 'blur' })`.
- Offizielle Doku: [Angular – Async Validation](https://angular.dev/guide/forms/form-validation#async-validation)
