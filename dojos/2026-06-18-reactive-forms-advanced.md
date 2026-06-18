# Angular Dojo: Reactive Forms Advanced
**Datum:** 2026-06-18
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie man mit `FormArray` dynamisch wachsende Formulare baut und eigene synchrone sowie asynchrone Validatoren erstellt, die über einzelne Felder hinaus die gesamte Gruppe validieren können.

## Hintergrund & Theorie

Reactive Forms bieten gegenüber Template-Driven-Forms maximale Kontrolle: Der gesamte Formular-Zustand lebt im TypeScript-Code und ist direkt testbar.

**FormBuilder** ist eine Komfort-Klasse, die das manuelle `new FormGroup(...)` / `new FormControl(...)` ersetzt und den Code erheblich kürzer macht.

**FormArray** ermöglicht eine variable Anzahl von Formular-Einträgen (z. B. eine Liste von E-Mail-Adressen oder Zeilen in einer Tabelle). Einträge lassen sich zur Laufzeit per `push()`, `removeAt()` und `insert()` verwalten.

**Eigene Validatoren** sind einfache Funktionen mit der Signatur:
```typescript
(control: AbstractControl): ValidationErrors | null
```
Gibt die Funktion `null` zurück, ist das Feld gültig. Asynchrone Validatoren geben stattdessen `Observable<ValidationErrors | null>` oder `Promise<ValidationErrors | null>` zurück und werden als drittes Argument übergeben.

**Cross-Field-Validatoren** werden auf einem `FormGroup`-Control gesetzt und haben Zugriff auf alle Kind-Controls gleichzeitig – ideal für Vergleiche wie „Passwort == Passwort wiederholen".

## Aufgabe

Baue ein **Buchungs-Formular** für einen Kurs, das folgende Anforderungen erfüllt:

1. Ein Teilnehmer hat einen **Namen** (Pflichtfeld, min. 2 Zeichen) und eine **E-Mail-Adresse** (Pflichtfeld, valides E-Mail-Format).
2. Es können **1 bis 5 Teilnehmer** dynamisch hinzugefügt oder entfernt werden.
3. Ein **Cross-Field-Validator** auf der Gruppe prüft, ob alle E-Mail-Adressen innerhalb der Liste **eindeutig** sind – doppelte Einträge sollen einen Fehler `duplicateEmail` erzeugen.
4. Ein **asynchroner Validator** simuliert eine Backend-Prüfung des Namens: Der Name `"blocked"` soll nach 500 ms den Fehler `nameTaken` zurückgeben.
5. Der Submit-Button ist nur aktiv, wenn das gesamte Formular gültig ist.
6. Beim Absenden werden alle Teilnehmer als JSON in der Konsole ausgegeben.

### Schritte

1. Erstelle eine neue Komponente `BookingFormComponent` (standalone).
2. Injiziere `FormBuilder` und lege ein `FormGroup` mit einem `FormArray` namens `participants` an.
3. Schreibe eine Hilfsmethode `createParticipant()`, die eine `FormGroup` mit den Feldern `name` und `email` zurückgibt. Hänge den asynchronen Validator an das `name`-Control.
4. Implementiere den synchronen Cross-Field-Validator `uniqueEmailsValidator` als eigenständige Funktion.
5. Setze den Validator auf das `FormArray` (nicht auf die äußere Gruppe).
6. Implementiere in der Klasse die Methoden `addParticipant()`, `removeParticipant(index: number)` und `onSubmit()`.
7. Baue das Template mit `*ngFor` über `participants.controls`, zeige Fehlermeldungen an und binde den Submit-Button.

## Hints

<details>
<summary>Hint 1 – FormArray anlegen</summary>

```typescript
this.form = this.fb.group({
  participants: this.fb.array(
    [this.createParticipant()],
    { validators: uniqueEmailsValidator }
  )
});

get participants(): FormArray {
  return this.form.get('participants') as FormArray;
}
```

</details>

<details>
<summary>Hint 2 – Asynchroner Validator</summary>

```typescript
function nameAvailableValidator(
  control: AbstractControl
): Observable<ValidationErrors | null> {
  return of(control.value).pipe(
    delay(500),
    map(name => (name === 'blocked' ? { nameTaken: true } : null))
  );
}
```
Übergabe als drittes Argument: `this.fb.control('', [Validators.required], [nameAvailableValidator])`.

</details>

<details>
<summary>Hint 3 – Cross-Field-Validator für eindeutige E-Mails</summary>

```typescript
function uniqueEmailsValidator(array: AbstractControl): ValidationErrors | null {
  const formArray = array as FormArray;
  const emails = formArray.controls
    .map(c => c.get('email')?.value?.toLowerCase())
    .filter(Boolean);
  const hasDuplicates = emails.length !== new Set(emails).size;
  return hasDuplicates ? { duplicateEmail: true } : null;
}
```

</details>

## Beispiellösung

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import {
  ReactiveFormsModule,
  FormBuilder,
  FormGroup,
  FormArray,
  Validators,
  AbstractControl,
  ValidationErrors,
} from '@angular/forms';
import { Observable, of } from 'rxjs';
import { delay, map } from 'rxjs/operators';

// --- Validators ---

function uniqueEmailsValidator(array: AbstractControl): ValidationErrors | null {
  const fa = array as FormArray;
  const emails = fa.controls
    .map(c => c.get('email')?.value?.toLowerCase()?.trim())
    .filter(Boolean);
  return emails.length !== new Set(emails).size ? { duplicateEmail: true } : null;
}

function nameAvailableValidator(
  control: AbstractControl
): Observable<ValidationErrors | null> {
  return of(control.value).pipe(
    delay(500),
    map(name => (name?.toLowerCase() === 'blocked' ? { nameTaken: true } : null))
  );
}

// --- Component ---

@Component({
  selector: 'app-booking-form',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div formArrayName="participants">
        <div
          *ngFor="let participant of participants.controls; let i = index"
          [formGroupName]="i"
          style="border:1px solid #ccc; padding:8px; margin-bottom:8px"
        >
          <h4>Teilnehmer {{ i + 1 }}</h4>

          <label>Name
            <input formControlName="name" />
            <span *ngIf="participant.get('name')?.hasError('required') && participant.get('name')?.touched">
              Pflichtfeld
            </span>
            <span *ngIf="participant.get('name')?.hasError('minlength') && participant.get('name')?.touched">
              Mindestens 2 Zeichen
            </span>
            <span *ngIf="participant.get('name')?.hasError('nameTaken')">
              Name bereits vergeben
            </span>
            <span *ngIf="participant.get('name')?.status === 'PENDING'">
              Prüfe…
            </span>
          </label>

          <label>E-Mail
            <input formControlName="email" type="email" />
            <span *ngIf="participant.get('email')?.hasError('required') && participant.get('email')?.touched">
              Pflichtfeld
            </span>
            <span *ngIf="participant.get('email')?.hasError('email') && participant.get('email')?.touched">
              Ungültige E-Mail
            </span>
          </label>

          <button type="button" (click)="removeParticipant(i)" [disabled]="participants.length === 1">
            Entfernen
          </button>
        </div>
      </div>

      <p *ngIf="participants.hasError('duplicateEmail')" style="color:red">
        Doppelte E-Mail-Adressen sind nicht erlaubt.
      </p>

      <button
        type="button"
        (click)="addParticipant()"
        [disabled]="participants.length >= 5"
      >
        Teilnehmer hinzufügen
      </button>

      <button type="submit" [disabled]="form.invalid || form.pending">
        Buchen
      </button>
    </form>
  `,
})
export class BookingFormComponent {
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      participants: this.fb.array(
        [this.createParticipant()],
        { validators: uniqueEmailsValidator }
      ),
    });
  }

  get participants(): FormArray {
    return this.form.get('participants') as FormArray;
  }

  createParticipant(): FormGroup {
    return this.fb.group({
      name: this.fb.control(
        '',
        [Validators.required, Validators.minLength(2)],
        [nameAvailableValidator]
      ),
      email: this.fb.control('', [Validators.required, Validators.email]),
    });
  }

  addParticipant(): void {
    if (this.participants.length < 5) {
      this.participants.push(this.createParticipant());
    }
  }

  removeParticipant(index: number): void {
    if (this.participants.length > 1) {
      this.participants.removeAt(index);
    }
  }

  onSubmit(): void {
    if (this.form.valid) {
      console.log(JSON.stringify(this.participants.value, null, 2));
    }
  }
}
```

## Weiterführendes

- **`updateOn: 'blur'`** auf einzelnen Controls setzt die Validierung auf „nach Fokusverlust", was bei asynchronen Validatoren Netzwerkanfragen spart: `this.fb.control('', { updateOn: 'blur' })`.
- Schau dir **`FormRecord`** an (seit Angular 14): ein typsicheres `FormGroup`-Äquivalent für dynamische Keys – nützlich, wenn die Control-Namen erst zur Laufzeit bekannt sind.
- Für komplexe Formulare lohnt sich die Bibliothek **`ngx-sub-form`**, die Formulare in wiederverwendbare Unter-Komponenten aufteilt.
