# Angular Dojo: Forms Events API (Angular 18+)
**Datum:** 2026-08-29
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst die neue typisierte `events`-Observable auf Reactive Forms kennen, die seit Angular 18 verfügbar ist. Du kannst gezielt auf einzelne Ereignistypen reagieren, ohne mehrere Observables zu kombinieren.

## Hintergrund & Theorie
Vor Angular 18 musste man `valueChanges`, `statusChanges` und zusätzliche Logik für `touched`/`pristine` separat subscriben und kombinieren. Angular 18 führt `AbstractControl.events` ein: eine einzige Observable, die **typisierte Ereignisobjekte** emittiert.

Die vier Event-Typen:
- **`ValueChangeEvent<T>`** – emittiert wenn der Wert sich ändert (äquivalent zu `valueChanges`)
- **`StatusChangeEvent`** – emittiert wenn der Status (`VALID`, `INVALID`, `PENDING`, `DISABLED`) wechselt
- **`TouchedChangeEvent`** – emittiert wenn das Control berührt oder unberührt wird
- **`PristineChangeEvent`** – emittiert wenn das Control dirty oder wieder pristine wird

Jedes Event-Objekt hat eine `source`-Referenz auf das auslösende `AbstractControl` und einen typspezifischen Wert. Durch `instanceof`-Checks oder den `type`-Discriminator kann man gezielt reagieren. Das ist besonders nützlich bei `FormGroup`-Ebene, wo Events von beliebigen Child-Controls aufsteigen.

```typescript
import { ValueChangeEvent, StatusChangeEvent } from '@angular/forms';

form.events.pipe(
  filter((e): e is ValueChangeEvent<MyType> => e instanceof ValueChangeEvent)
).subscribe(e => console.log(e.value));
```

## Aufgabe
Erstelle eine Standalone-Komponente `UserProfileFormComponent` mit einem reaktiven Formular für ein Benutzerprofil. Nutze ausschließlich `form.events` (kein `valueChanges`/`statusChanges`) um:

1. Ein Live-Validierungsprotokoll als Liste anzuzeigen (Timestamp + Ereignistyp + Wert/Status)
2. Einen Submit-Button nur zu aktivieren, wenn das Formular `VALID` und `dirty` ist — abgeleitet aus dem Events-Stream statt aus Template-Bindings auf `form.valid` und `form.dirty`
3. Eine „Unsaved changes"-Warnung anzuzeigen, sobald das Formular dirty wird, und sie beim Reset zu verstecken

### Schritte
1. Erstelle eine neue Standalone-Komponente mit `ReactiveFormsModule` und einem `FormGroup` mit den Feldern `username` (required, minLength 3) und `email` (required, email-Validator).
2. Erstelle in `ngOnInit` einen einzigen `events`-Stream und verwende `instanceof`-Guards (oder `type`-Discriminator), um die drei Zustände (`isValid`, `isDirty`, Protokolleinträge) mit Signals zu befüllen.
3. Leite `canSubmit = computed(() => isValid() && isDirty())` ab und binde es an den `[disabled]`-Attribute des Buttons.
4. Zeige das Protokoll als `<ul>` mit den letzten 10 Einträgen an. Füge einen Reset-Button hinzu, der `form.reset()` aufruft.
5. Stelle sicher, dass das Subscription per `takeUntilDestroyed()` aufgeräumt wird.

## Hints
<details>
<summary>Hint 1 – Imports</summary>

```typescript
import {
  ValueChangeEvent,
  StatusChangeEvent,
  TouchedChangeEvent,
  PristineChangeEvent,
} from '@angular/forms';
import { filter } from 'rxjs/operators';
```
`form.events` ist auf `AbstractControl` definiert, also auf `FormGroup`, `FormControl` und `FormArray`.
</details>

<details>
<summary>Hint 2 – Signal-Integration</summary>

Nutze `toSignal` aus `@angular/core/rxjs-interop`, um den Stream in Signals umzuwandeln, oder schreibe manuell in Signals innerhalb des `subscribe`-Callbacks. Mit `takeUntilDestroyed(this.destroyRef)` ist kein manuelles `unsubscribe` nötig:

```typescript
private destroyRef = inject(DestroyRef);

ngOnInit() {
  this.form.events.pipe(
    takeUntilDestroyed(this.destroyRef)
  ).subscribe(event => {
    if (event instanceof StatusChangeEvent) {
      this.isValid.set(event.status === 'VALID');
    }
    if (event instanceof PristineChangeEvent) {
      this.isDirty.set(!event.pristine);
    }
    // Protokolleintrag hinzufügen
    this.log.update(prev => [...prev.slice(-9), formatEvent(event)]);
  });
}
```
</details>

<details>
<summary>Hint 3 – Event-Discriminator</summary>

Statt `instanceof` kann man den `type`-String nutzen:

```typescript
// event.type === 'valueChange' | 'statusChange' | 'touchedChange' | 'pristineChange'
if (event.type === 'valueChange') { /* ... */ }
```

Das ist nützlich, wenn TypeScript-Discriminated-Unions bevorzugt werden.
</details>

## Beispiellösung

```typescript
import { Component, OnInit, signal, computed, inject, DestroyRef } from '@angular/core';
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';
import {
  ValueChangeEvent,
  StatusChangeEvent,
  PristineChangeEvent,
  TouchedChangeEvent,
} from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DatePipe, JsonPipe } from '@angular/common';

interface LogEntry {
  time: string;
  type: string;
  detail: string;
}

@Component({
  selector: 'app-user-profile-form',
  standalone: true,
  imports: [ReactiveFormsModule, DatePipe, JsonPipe],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div>
        <label>Username</label>
        <input formControlName="username" />
        @if (form.get('username')?.invalid && form.get('username')?.touched) {
          <small>Min. 3 Zeichen erforderlich</small>
        }
      </div>
      <div>
        <label>E-Mail</label>
        <input formControlName="email" type="email" />
      </div>

      @if (isDirty()) {
        <p class="warning">⚠ Ungespeicherte Änderungen</p>
      }

      <button type="submit" [disabled]="!canSubmit()">Speichern</button>
      <button type="button" (click)="form.reset()">Reset</button>
    </form>

    <h4>Ereignisprotokoll (letzte 10)</h4>
    <ul>
      @for (entry of log(); track entry.time) {
        <li><strong>{{ entry.time }}</strong> [{{ entry.type }}] {{ entry.detail }}</li>
      }
    </ul>
  `,
})
export class UserProfileFormComponent implements OnInit {
  private fb = inject(FormBuilder);
  private destroyRef = inject(DestroyRef);

  form = this.fb.group({
    username: ['', [Validators.required, Validators.minLength(3)]],
    email: ['', [Validators.required, Validators.email]],
  });

  isValid = signal(false);
  isDirty = signal(false);
  log = signal<LogEntry[]>([]);

  canSubmit = computed(() => this.isValid() && this.isDirty());

  ngOnInit() {
    this.form.events.pipe(takeUntilDestroyed(this.destroyRef)).subscribe(event => {
      const time = new Date().toLocaleTimeString();

      if (event instanceof StatusChangeEvent) {
        this.isValid.set(event.status === 'VALID');
        this.addLog(time, 'statusChange', event.status);
      } else if (event instanceof PristineChangeEvent) {
        this.isDirty.set(!event.pristine);
        this.addLog(time, 'pristineChange', event.pristine ? 'pristine' : 'dirty');
      } else if (event instanceof TouchedChangeEvent) {
        this.addLog(time, 'touchedChange', event.touched ? 'touched' : 'untouched');
      } else if (event instanceof ValueChangeEvent) {
        this.addLog(time, 'valueChange', JSON.stringify(event.value));
      }
    });
  }

  private addLog(time: string, type: string, detail: string) {
    this.log.update(prev => [...prev.slice(-9), { time, type, detail }]);
  }

  onSubmit() {
    if (this.canSubmit()) {
      console.log('Gespeichert:', this.form.value);
      this.form.reset();
    }
  }
}
```

## Weiterführendes
- [Angular 18 Blog – Forms Events](https://blog.angular.dev/angular-v18-is-now-available-e79d5ac0affe) – Announcement und Motivation für die API
- Kombiniere `form.events` mit `debounceTime` + `filter(e => e instanceof ValueChangeEvent)` für eine Auto-Save-Funktion, die nur bei validen Werten auslöst
- Für komplexe `FormArray`-Szenarien: `events` auf dem Array selbst liefert auch Events für Child-Controls — nutze `event.source` um zu identifizieren, welches Kind die Änderung ausgelöst hat
