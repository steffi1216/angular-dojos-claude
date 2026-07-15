# Angular Dojo: Typed Reactive Forms
**Datum:** 2026-07-15
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie Angular's stark typisierte reaktive Formulare (seit Angular 14) funktionieren und wie du mit `FormControl<T>`, `FormGroup<{...}>`, `FormArray<FormControl<T>>` und `NonNullableFormBuilder` typsichere Formulare baust.

## Hintergrund & Theorie

Seit Angular 14 sind reaktive Formulare vollständig generisch typisiert. Das bedeutet: TypeScript kennt den Typ jedes Feldes zur Compile-Zeit, und falsche Zugriffe werden direkt als Fehler markiert.

**Wichtige Konzepte:**

- `FormControl<string | null>` – Standardverhalten: `null` ist immer erlaubt (Reset setzt auf `null`)
- `FormControl<string>` mit `nonNullable: true` – Reset setzt auf den Initialwert statt `null`
- `NonNullableFormBuilder` – erzeugt alle Controls automatisch mit `nonNullable: true`
- `FormGroup<{name: FormControl<string>}>` – Typ des `.value` wird automatisch inferiert
- `FormArray<FormControl<number>>` – alle Elemente haben denselben Typ
- `.getRawValue()` liefert immer alle Werte (auch disabled), `.value` kann `undefined` für disabled-Felder enthalten

**Warum ist das wichtig?**

Ohne Typen war `form.value.firstName` immer `any` – Tippfehler wurden erst zur Laufzeit bemerkt. Mit Typen erkennt TypeScript sofort, ob ein Feld existiert oder falsch geschrieben wurde.

```typescript
// Vorher (untyped):
const name = form.get('naem')?.value; // kein Fehler, aber Tippfehler!

// Jetzt (typed):
const name = form.controls.name.value; // TypeScript-Fehler, wenn 'name' nicht existiert
```

## Aufgabe

Erstelle ein typsicheres Registrierungsformular für eine Benutzerregistrierung mit folgenden Feldern:
- `username` (string, mindestens 3 Zeichen, required)
- `email` (string, E-Mail-Format, required)
- `age` (number, zwischen 18 und 99, optional – nullable)
- `skills` (dynamisches Array aus Strings, mindestens 1 Eintrag)

Das Formular soll:
1. Vollständig typisiert sein (kein `any`)
2. `NonNullableFormBuilder` für alle nicht-nullable Felder nutzen
3. Den Typ des submitted Werts explizit definieren (`type FormValue = ...`)
4. Bei Submit den Wert als typsicheres Objekt in der Konsole ausgeben

### Schritte

1. Erstelle eine Standalone-Komponente `RegistrationFormComponent`
2. Injiziere `NonNullableFormBuilder` (via `inject()`) und definiere das `FormGroup`-Schema mit expliziten Typen
3. Füge `age` als nullable `FormControl<number | null>` hinzu (Sonderfall)
4. Implementiere `addSkill()` und `removeSkill(index: number)` für das `FormArray`
5. Definiere `type FormValue` basierend auf dem Rückgabetyp von `.getRawValue()`
6. Implementiere `onSubmit()` mit typsicherem Zugriff auf alle Felder

## Hints

<details>
<summary>Hint 1 – NonNullableFormBuilder und nullable Sonderfelder</summary>

```typescript
import { inject } from '@angular/core';
import { NonNullableFormBuilder, FormControl } from '@angular/forms';

// NonNullableFormBuilder für nicht-nullable Felder
private fb = inject(NonNullableFormBuilder);

// Für nullable Felder: explizite FormControl mit null
age: new FormControl<number | null>(null),

// Alles andere über NonNullableFormBuilder:
this.fb.group({
  username: ['', [Validators.required, Validators.minLength(3)]],
  email: ['', [Validators.required, Validators.email]],
})
```

</details>

<details>
<summary>Hint 2 – Typen von FormGroup und FormArray ableiten</summary>

```typescript
import { FormGroup, FormArray, FormControl } from '@angular/forms';

// Den Typ des Formulars explizit ableiten:
type RegistrationForm = FormGroup<{
  username: FormControl<string>;
  email: FormControl<string>;
  age: FormControl<number | null>;
  skills: FormArray<FormControl<string>>;
}>;

// Oder den Wert-Typ vom getRawValue() ableiten:
type FormValue = ReturnType<RegistrationForm['getRawValue']>;
// => { username: string; email: string; age: number | null; skills: string[] }
```

</details>

<details>
<summary>Hint 3 – FormArray dynamisch verwalten</summary>

```typescript
get skillsArray(): FormArray<FormControl<string>> {
  return this.form.controls.skills;
}

addSkill(): void {
  this.skillsArray.push(this.fb.control('', Validators.required));
}

removeSkill(index: number): void {
  this.skillsArray.removeAt(index);
}

onSubmit(): void {
  if (this.form.valid) {
    const value: FormValue = this.form.getRawValue();
    console.log('Username:', value.username); // string – kein any!
    console.log('Skills:', value.skills);     // string[] – typsicher!
  }
}
```

</details>

## Beispiellösung

```typescript
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import {
  NonNullableFormBuilder,
  FormControl,
  FormGroup,
  FormArray,
  ReactiveFormsModule,
  Validators,
} from '@angular/forms';

type RegistrationForm = FormGroup<{
  username: FormControl<string>;
  email: FormControl<string>;
  age: FormControl<number | null>;
  skills: FormArray<FormControl<string>>;
}>;

type FormValue = ReturnType<RegistrationForm['getRawValue']>;

@Component({
  selector: 'app-registration-form',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div>
        <label>Username</label>
        <input formControlName="username" />
        @if (form.controls.username.errors?.['minlength']) {
          <span>Mindestens 3 Zeichen</span>
        }
      </div>

      <div>
        <label>E-Mail</label>
        <input formControlName="email" type="email" />
      </div>

      <div>
        <label>Alter (optional)</label>
        <input formControlName="age" type="number" />
      </div>

      <div formArrayName="skills">
        <label>Skills</label>
        @for (skill of skillsArray.controls; track $index) {
          <div>
            <input [formControlName]="$index" />
            <button type="button" (click)="removeSkill($index)">–</button>
          </div>
        }
        <button type="button" (click)="addSkill()">+ Skill hinzufügen</button>
      </div>

      <button type="submit" [disabled]="form.invalid">Registrieren</button>
    </form>

    @if (submittedValue) {
      <pre>{{ submittedValue | json }}</pre>
    }
  `,
})
export class RegistrationFormComponent {
  private fb = inject(NonNullableFormBuilder);

  form: RegistrationForm = this.fb.group({
    username: ['', [Validators.required, Validators.minLength(3)]],
    email: ['', [Validators.required, Validators.email]],
    age: new FormControl<number | null>(null),
    skills: this.fb.array([this.fb.control('', Validators.required)]),
  });

  submittedValue: FormValue | null = null;

  get skillsArray(): FormArray<FormControl<string>> {
    return this.form.controls.skills;
  }

  addSkill(): void {
    this.skillsArray.push(this.fb.control('', Validators.required));
  }

  removeSkill(index: number): void {
    this.skillsArray.removeAt(index);
  }

  onSubmit(): void {
    if (this.form.valid) {
      const value: FormValue = this.form.getRawValue();
      this.submittedValue = value;
      console.log('Registrierung:', value);
    }
  }
}
```

## Weiterführendes

- **Untyped Forms als Migration:** Bestehende Formulare können schrittweise migriert werden – `UntypedFormControl` ist ein Alias für das alte Verhalten und kann als Zwischenschritt genutzt werden.
- **`FormRecord<T>`:** Für Formulare mit dynamischen, unbekannten Schlüsseln (ähnlich `Record<string, T>`) – nützlich z. B. für Konfigurationsformulare mit variablen Feldern.
- **Offizielle Doku:** [Angular – Typed Forms](https://angular.dev/guide/forms/typed-forms) mit weiteren Migrationshinweisen und Edge Cases.
