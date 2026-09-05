# Angular Dojo: FormRecord — Typsichere dynamische Formulare
**Datum:** 2026-09-05
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Lerne `FormRecord<TControl>` kennen — Angulars typsichere Alternative zu `FormGroup` für Formulare mit dynamischen, zur Laufzeit hinzugefügten oder entfernten Feldern mit einheitlichem Typ.

## Hintergrund & Theorie

Seit Angular 14 sind Reactive Forms vollständig typsiert. Dabei decken drei Klassen unterschiedliche Szenarien ab:

| Klasse | Einsatz |
|---|---|
| `FormControl<T>` | Einzelnes Feld mit festem Typ |
| `FormArray<TControl>` | Liste von gleichartigen Controls (indiziert) |
| `FormGroup<TControls>` | Feste Menge benannter Controls (Struktur fix) |
| **`FormRecord<TControl>`** | **Dynamische Menge benannter Controls (Keys zur Laufzeit)** |

`FormRecord` ist ein Spezialfall von `FormGroup`, bei dem alle Werte den **gleichen Control-Typ** haben und Controls per `addControl()` / `removeControl()` frei hinzugefügt oder entfernt werden können. Der TypeScript-Typ für `value` ist dabei `Partial<Record<string, T>>`.

**Typischer Use-Case:** Ein Formular, bei dem der Nutzer selbst Felder benennt — z. B. ein „Eigenschaften"-Editor, Multi-Language-Texte, Feature-Flags pro Umgebung oder ein dynamisches Filter-Panel.

`FormRecord` bietet gegenüber ungetyptem `FormGroup` volle Type Safety: `setValue`, `patchValue`, `value`, und Control-Zugriff sind alle korrekt typisiert.

## Aufgabe

Baue eine **Feature-Flag-Konfigurationskomponente**: Der Nutzer kann Feature-Flags (Key-Value-Paare, Wert = boolean) dynamisch hinzufügen, per Toggle an- und ausschalten und wieder entfernen. Die aktuelle Konfiguration wird live als JSON angezeigt.

### Schritte

1. Erstelle eine Standalone-Komponente `FeatureFlagsComponent`. Initialisiere ein `FormRecord<FormControl<boolean>>` mit zwei vordefinierten Flags (`darkMode: true`, `betaFeatures: false`).

2. Füge eine Methode `addFlag(name: string)` hinzu, die per `addControl()` ein neues `FormControl<boolean>(false)` zum Record hinzufügt. Verhindere doppelte Keys und leere Namen.

3. Füge eine Methode `removeFlag(name: string)` hinzu, die per `removeControl()` ein Flag entfernt.

4. Erstelle das Template: Eine Liste aller Flags (iteriere über `Object.keys(form.controls)`) mit je einem `<input type="checkbox">` (gebunden per `formControlName`), dem Flag-Namen und einem Löschen-Button. Ergänze ein Eingabefeld + Button zum Hinzufügen neuer Flags.

5. Zeige den aktuellen `form.value` als live-aktualisiertes JSON-Preview an (nutze `form.valueChanges` mit `startWith` oder ein Signal via `toSignal`).

## Hints

<details>
<summary>Hint 1 — FormRecord initialisieren</summary>

```typescript
import { FormRecord, FormControl, ReactiveFormsModule } from '@angular/forms';

form = new FormRecord<FormControl<boolean>>({
  darkMode: new FormControl(true, { nonNullable: true }),
  betaFeatures: new FormControl(false, { nonNullable: true }),
});
```

`nonNullable: true` stellt sicher, dass `reset()` auf den Initialwert zurückfällt statt auf `null`.
</details>

<details>
<summary>Hint 2 — Controls iterieren im Template</summary>

```typescript
// Im Component
get flagKeys(): string[] {
  return Object.keys(this.form.controls);
}
```

```html
<!-- Im Template -->
<div [formGroup]="form">
  @for (key of flagKeys; track key) {
    <label>
      <input type="checkbox" [formControlName]="key" />
      {{ key }}
    </label>
    <button (click)="removeFlag(key)">✕</button>
  }
</div>
```
</details>

<details>
<summary>Hint 3 — Live-Preview mit toSignal</summary>

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { startWith } from 'rxjs';

preview = toSignal(
  this.form.valueChanges.pipe(startWith(this.form.value))
);
```

Im Template: `<pre>{{ preview() | json }}</pre>`
</details>

## Beispiellösung

```typescript
import { Component, inject, signal } from '@angular/core';
import { FormControl, FormRecord, ReactiveFormsModule } from '@angular/forms';
import { toSignal } from '@angular/core/rxjs-interop';
import { JsonPipe } from '@angular/common';
import { startWith } from 'rxjs';

@Component({
  selector: 'app-feature-flags',
  standalone: true,
  imports: [ReactiveFormsModule, JsonPipe],
  template: `
    <h2>Feature Flags</h2>

    <form [formGroup]="form">
      @for (key of flagKeys; track key) {
        <div class="flag-row">
          <label>
            <input type="checkbox" [formControlName]="key" />
            {{ key }}
          </label>
          <button type="button" (click)="removeFlag(key)">Entfernen</button>
        </div>
      }
    </form>

    <div class="add-flag">
      <input #nameInput placeholder="Neues Flag..." (keydown.enter)="addFlag(nameInput.value); nameInput.value = ''" />
      <button (click)="addFlag(nameInput.value); nameInput.value = ''">Hinzufügen</button>
    </div>

    @if (error()) {
      <p class="error">{{ error() }}</p>
    }

    <h3>Aktuelle Konfiguration</h3>
    <pre>{{ preview() | json }}</pre>
  `,
  styles: [`
    .flag-row { display: flex; align-items: center; gap: 8px; margin: 4px 0; }
    .add-flag { margin: 16px 0; display: flex; gap: 8px; }
    .error { color: red; }
    pre { background: #f4f4f4; padding: 12px; border-radius: 4px; }
  `],
})
export class FeatureFlagsComponent {
  form = new FormRecord<FormControl<boolean>>({
    darkMode: new FormControl(true, { nonNullable: true }),
    betaFeatures: new FormControl(false, { nonNullable: true }),
  });

  error = signal('');

  preview = toSignal(
    this.form.valueChanges.pipe(startWith(this.form.value)),
  );

  get flagKeys(): string[] {
    return Object.keys(this.form.controls);
  }

  addFlag(name: string): void {
    const trimmed = name.trim();
    if (!trimmed) {
      this.error.set('Name darf nicht leer sein.');
      return;
    }
    if (this.form.contains(trimmed)) {
      this.error.set(`Flag "${trimmed}" existiert bereits.`);
      return;
    }
    this.error.set('');
    this.form.addControl(trimmed, new FormControl(false, { nonNullable: true }));
  }

  removeFlag(name: string): void {
    this.form.removeControl(name);
  }
}
```

### Warum nicht einfach `FormGroup`?

```typescript
// FormGroup: Struktur ist statisch und muss im Typ deklariert sein
form = new FormGroup({
  darkMode: new FormControl(true),
  // Weitere Controls zur Laufzeit hinzuzufügen bricht die Typisierung
});

// FormRecord: explizit für dynamische Keys konzipiert — volle Type Safety
form = new FormRecord<FormControl<boolean>>({});
form.addControl('myFlag', new FormControl(false, { nonNullable: true }));
// form.value ist Partial<Record<string, boolean>> ✓
```

## Weiterführendes

- **`FormGroup` vs `FormRecord` vs `FormArray`** — Die Wahl hängt vom Datenmodell ab: Array-Index → `FormArray`, feste benannte Keys → `FormGroup`, dynamische gleichartige Keys → `FormRecord`.
- Kombiniere `FormRecord` mit **Validators**: `this.form.addControl('flag', new FormControl(false, { nonNullable: true, validators: [Validators.required] }))` funktioniert genauso wie bei `FormGroup`.
- **Offizielle Docs:** [Angular Typed Forms](https://angular.dev/guide/forms/typed-forms) — besonders den Abschnitt „FormRecord".
- Für **heterogene dynamische Felder** (verschiedene Typen pro Key) bleibt `FormGroup` mit explizitem Cast die einzige Option — `FormRecord` erzwingt einen einheitlichen Control-Typ.
