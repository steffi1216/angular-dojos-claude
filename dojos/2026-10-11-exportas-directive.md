# Angular Dojo: `exportAs` – Direktiven als Template-Referenz-Variablen
**Datum:** 2026-10-11
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Direktiven über die `exportAs`-Eigenschaft als Template-Referenz-Variable zugänglich machst und damit komplexe komponentenübergreifende Interaktionen direkt im Template ermöglichst – ohne `@Output`, `@ViewChild` oder Service-Umwege.

## Hintergrund & Theorie

Die `exportAs`-Eigenschaft im `@Directive`-Decorator erlaubt es, eine Direktive (oder Komponente) unter einem Namen im Template zu "exportieren". Dieses Muster kommt in Angular Material extensiv zum Einsatz – zum Beispiel:

```html
<input [matDatepicker]="picker" />
<mat-datepicker #picker="matDatepicker"></mat-datepicker>
<button (click)="picker.open()">Öffnen</button>
```

Die Direktive `MatDatepicker` deklariert `exportAs: 'matDatepicker'`, sodass `#picker` direkt eine Instanz der Direktive hält und deren public API (hier: `open()`) zugreifbar macht.

**Warum ist das nützlich?**
- Das Elternelement muss die Direktive nicht kennen – nur das Template interagiert direkt.
- Kein `@ViewChild` oder `@Output` nötig, wenn alles im gleichen Template bleibt.
- Perfekt für wiederverwendbare UI-Bausteine wie Tooltips, Popover, Accordion, Stepper.
- Mehrere `exportAs`-Namen sind per Komma möglich: `exportAs: 'name1, name2'`.

Eine Komponente kann ebenfalls `exportAs` definieren – dann hält die Template-Referenz statt der native DOM-Instanz die Komponenteninstanz.

## Aufgabe

Erstelle eine `PasswordStrengthDirective`, die einem `<input type="password">` hinzugefügt werden kann. Die Direktive soll:

1. Die Passwortstärke als `'weak' | 'medium' | 'strong'` berechnen.
2. Über `exportAs: 'passwordStrength'` im Template nutzbar sein.
3. Ein Signal `strength` und eine Methode `getScore(): number` anbieten.
4. In einem Formular-Template genutzt werden, um neben dem Input direkt die Stärke anzuzeigen – **ohne `@ViewChild` im Component-Code**.

### Schritte

1. Generiere (oder erstelle) eine Direktive `PasswordStrengthDirective` mit `exportAs: 'passwordStrength'`.
2. Injiziere `NgControl` (optional, mit `@Optional() @Self()`), um den Formularwert zu lesen – oder nutze `HostListener` auf das `input`-Event.
3. Berechne den Score anhand einfacher Regeln:
   - Länge ≥ 8 → +1 Punkt
   - Enthält Zahl → +1 Punkt
   - Enthält Sonderzeichen → +1 Punkt
   - Enthält Groß- **und** Kleinbuchstaben → +1 Punkt
4. Mappe den Score auf `'weak'` (0–1), `'medium'` (2), `'strong'` (3–4).
5. Speichere `strength` als `signal<'weak' | 'medium' | 'strong'>('weak')`.
6. Nutze die Direktive in einer Standalone-Komponente:
   ```html
   <input type="password" passwordStrength #ps="passwordStrength" />
   <span [class]="ps.strength()">{{ ps.strength() }}</span>
   ```
7. Füge ein wenig CSS hinzu: `weak` = rot, `medium` = gelb, `strong` = grün.

## Hints

<details>
<summary>Hint 1 – Direktiven-Grundgerüst mit exportAs</summary>

```typescript
@Directive({
  selector: '[passwordStrength]',
  exportAs: 'passwordStrength',
  standalone: true,
})
export class PasswordStrengthDirective {
  strength = signal<'weak' | 'medium' | 'strong'>('weak');

  @HostListener('input', ['$event.target.value'])
  onInput(value: string): void {
    this.strength.set(this.evaluate(value));
  }

  private evaluate(value: string): 'weak' | 'medium' | 'strong' {
    // Score-Logik hier
  }
}
```

</details>

<details>
<summary>Hint 2 – Score-Berechnung und Template-Nutzung</summary>

```typescript
getScore(value: string): number {
  let score = 0;
  if (value.length >= 8) score++;
  if (/\d/.test(value)) score++;
  if (/[^a-zA-Z0-9]/.test(value)) score++;
  if (/[a-z]/.test(value) && /[A-Z]/.test(value)) score++;
  return score;
}
```

Im Template brauchst du keinen `@ViewChild` – die `#ps`-Referenz ist sofort die Direktiven-Instanz:

```html
<input type="password" passwordStrength #ps="passwordStrength" [(ngModel)]="password" />
<div class="meter" [class]="ps.strength()">
  Stärke: {{ ps.strength() }}
</div>
```

</details>

## Beispiellösung

```typescript
// password-strength.directive.ts
import { Directive, HostListener, signal } from '@angular/core';

@Directive({
  selector: '[passwordStrength]',
  exportAs: 'passwordStrength',
  standalone: true,
})
export class PasswordStrengthDirective {
  strength = signal<'weak' | 'medium' | 'strong'>('weak');

  @HostListener('input', ['$event.target.value'])
  onInput(value: string): void {
    this.strength.set(this.computeStrength(value));
  }

  getScore(value: string): number {
    let score = 0;
    if (value.length >= 8) score++;
    if (/\d/.test(value)) score++;
    if (/[^a-zA-Z0-9]/.test(value)) score++;
    if (/[a-z]/.test(value) && /[A-Z]/.test(value)) score++;
    return score;
  }

  private computeStrength(value: string): 'weak' | 'medium' | 'strong' {
    const score = this.getScore(value);
    if (score <= 1) return 'weak';
    if (score === 2) return 'medium';
    return 'strong';
  }
}

// app.component.ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { PasswordStrengthDirective } from './password-strength.directive';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [FormsModule, PasswordStrengthDirective],
  template: `
    <div class="form">
      <label>Passwort:
        <input
          type="password"
          passwordStrength
          #ps="passwordStrength"
          [(ngModel)]="password"
          placeholder="Passwort eingeben..."
        />
      </label>

      @if (password) {
        <div class="strength-meter" [class]="ps.strength()">
          Stärke: <strong>{{ ps.strength() }}</strong>
          (Score: {{ ps.getScore(password) }}/4)
        </div>
      }
    </div>
  `,
  styles: [`
    .form { display: flex; flex-direction: column; gap: 1rem; max-width: 400px; padding: 2rem; }
    input { padding: 0.5rem; font-size: 1rem; width: 100%; }
    .strength-meter { padding: 0.5rem 1rem; border-radius: 4px; font-size: 0.9rem; }
    .strength-meter.weak   { background: #fde8e8; color: #c0392b; border: 1px solid #c0392b; }
    .strength-meter.medium { background: #fef9e7; color: #d68910; border: 1px solid #d68910; }
    .strength-meter.strong { background: #eafaf1; color: #1e8449; border: 1px solid #1e8449; }
  `],
})
export class AppComponent {
  password = '';
}
```

**Bonus – Mehrere Direktiven auf demselben Element:**
```html
<!-- Beide Direktiven am gleichen Input nutzbar -->
<input
  type="password"
  passwordStrength #ps="passwordStrength"
  myTooltip #tip="myTooltip"
  (focus)="tip.show()"
/>
```

## Weiterführendes
- Studiere Angular Material's Quellcode (z. B. `MatTooltip`, `MatDatepicker`), um zu sehen, wie `exportAs` in komplexen Bibliotheken skaliert – das Muster ist überall gleich: die Direktive exportiert eine reiche API, das Template orchestriert.
- Kombiniere `exportAs` mit `linkedSignal()` oder `computed()` für reaktive Abgeleitete, die das Consumer-Template direkt per `#ref.someComputed()` ablesen kann – kein `AsyncPipe`, kein `subscribe`.
