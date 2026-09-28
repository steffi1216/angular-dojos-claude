# Angular Dojo: CDK Stepper – Multi-Step-Wizards bauen
**Datum:** 2026-09-28
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK `CdkStepper` einen vollständig kontrollierten Multi-Step-Wizard baust – mit Custom-Stepper-Komponente, Validierung pro Schritt und programmatischer Navigation.

## Hintergrund & Theorie
Das Angular CDK bietet mit `CdkStepper` eine layout-unabhängige Basis für Schritt-für-Schritt-Workflows. Im Gegensatz zu `MatStepper` aus Angular Material liefert der CDK-Stepper nur die Logik (Zustandsverwaltung, Tastatur-Navigation, Validierung) – das UI baust du komplett selbst.

Kernkonzepte:
- **`CdkStep`**: Jeder Schritt ist eine Direktive, die eine `TemplateRef` hält und optional ein `stepControl` (AbstractControl) für Validierung entgegennimmt.
- **`CdkStepper`**: Die Eltern-Direktive verwaltet die aktive Schritt-Liste, prüft vor dem Weiternavigieren ob der aktuelle Schritt valide ist (`linear`-Modus), und stellt `next()`, `previous()`, `reset()` und `selectedIndex` bereit.
- **Custom Stepper**: Durch Erweiterung von `CdkStepper` und Bereitstellung von `CDK_STEPPER` als Provider kannst du ein komplett eigenes Template und Styling einbringen, ohne die Logik neu zu schreiben.
- **`linear` Modus**: Wenn `linear="true"` gesetzt ist, wird `next()` geblockt, solange `stepControl` invalid ist. Im nicht-linearen Modus kann jeder Schritt direkt angesprungen werden.

## Aufgabe
Baue einen dreiteiligen „Onboarding-Wizard" mit einer eigenen `AppStepperComponent`, die `CdkStepper` erweitert. Jeder Schritt enthält ein Reactive-Forms-Fragment. Der Wizard ist `linear`, d.h. Schritt 2 ist nur erreichbar, wenn Schritt 1 valide ist, etc.

### Schritte
1. **Custom Stepper anlegen**: Erstelle `AppStepperComponent`, die `CdkStepper` extended und sich selbst als `CDK_STEPPER` Provider bereitstellt. Das Template iteriert über `steps` und rendert den aktiven Schritt per `selectedStep.content`.
2. **Drei Formgruppen definieren**: Im `WizardComponent` (Host) erzeuge drei `FormGroup`-Instanzen (z.B. Schritt 1: Name/E-Mail, Schritt 2: Adresse, Schritt 3: Passwort). Jeder `<cdk-step>` bekommt sein `[stepControl]` zugewiesen.
3. **Navigation implementieren**: Nutze `stepper.next()` und `stepper.previous()` via Template-Referenz (`#stepper="cdkStepper"`). Zeige einen „Abschicken"-Button nur im letzten Schritt.
4. **Schritt-Indikatoren**: Rendere oben eine Fortschrittsleiste, die den aktiven Schritt hervorhebt und bereits abgeschlossene Schritte markiert (`step.completed`, `stepper.selectedIndex`).
5. **Reset**: Implementiere einen „Neu starten"-Button, der `stepper.reset()` aufruft und alle Formulare zurücksetzt.

## Hints
<details>
<summary>Hint 1 – CdkStepper erweitern und bereitstellen</summary>

```typescript
import { CdkStepper, CDK_STEPPER } from '@angular/cdk/stepper';
import { Component, ChangeDetectionStrategy } from '@angular/core';
import { NgTemplateOutlet } from '@angular/common';

@Component({
  selector: 'app-stepper',
  standalone: true,
  imports: [NgTemplateOutlet],
  providers: [{ provide: CDK_STEPPER, useExisting: AppStepperComponent }],
  template: `
    <!-- Fortschrittsleiste -->
    <div class="steps-header">
      @for (step of steps; track $index) {
        <div
          class="step-indicator"
          [class.active]="selectedIndex === $index"
          [class.completed]="step.completed">
          {{ $index + 1 }}
        </div>
      }
    </div>
    <!-- Aktiver Schritt -->
    @if (selected) {
      <ng-container [ngTemplateOutlet]="selected.content" />
    }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class AppStepperComponent extends CdkStepper {}
```

</details>
<details>
<summary>Hint 2 – Wizard-Host mit FormGroups und CdkStep</summary>

```typescript
import { Component } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { CdkStepperModule } from '@angular/cdk/stepper';
import { AppStepperComponent } from './stepper.component';

@Component({
  selector: 'app-wizard',
  standalone: true,
  imports: [ReactiveFormsModule, CdkStepperModule, AppStepperComponent],
  template: `
    <app-stepper #stepper="cdkStepper" [linear]="true">
      <!-- Schritt 1 -->
      <cdk-step [stepControl]="step1">
        <form [formGroup]="step1">
          <input formControlName="name" placeholder="Name" />
          <input formControlName="email" placeholder="E-Mail" />
          <button type="button" cdkStepperNext>Weiter</button>
        </form>
      </cdk-step>

      <!-- Schritt 2 -->
      <cdk-step [stepControl]="step2">
        <form [formGroup]="step2">
          <input formControlName="city" placeholder="Stadt" />
          <button type="button" cdkStepperPrevious>Zurück</button>
          <button type="button" cdkStepperNext>Weiter</button>
        </form>
      </cdk-step>

      <!-- Schritt 3 -->
      <cdk-step [stepControl]="step3">
        <form [formGroup]="step3">
          <input formControlName="password" type="password" placeholder="Passwort" />
          <button type="button" cdkStepperPrevious>Zurück</button>
          <button type="submit" (click)="onSubmit()">Abschicken</button>
        </form>
      </cdk-step>
    </app-stepper>

    <button (click)="stepper.reset(); resetForms()">Neu starten</button>
  `,
})
export class WizardComponent {
  private fb = inject(FormBuilder);

  step1 = this.fb.group({
    name:  ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
  });

  step2 = this.fb.group({
    city: ['', Validators.required],
  });

  step3 = this.fb.group({
    password: ['', [Validators.required, Validators.minLength(8)]],
  });

  onSubmit() {
    console.log({ ...this.step1.value, ...this.step2.value, ...this.step3.value });
  }

  resetForms() {
    this.step1.reset();
    this.step2.reset();
    this.step3.reset();
  }
}
```

</details>

## Beispiellösung

```typescript
// stepper.component.ts
import { CdkStepper, CDK_STEPPER } from '@angular/cdk/stepper';
import { Component, ChangeDetectionStrategy } from '@angular/core';
import { NgTemplateOutlet, NgClass } from '@angular/common';

@Component({
  selector: 'app-stepper',
  standalone: true,
  imports: [NgTemplateOutlet, NgClass],
  providers: [{ provide: CDK_STEPPER, useExisting: AppStepperComponent }],
  template: `
    <nav class="stepper-header" aria-label="Fortschritt">
      @for (step of steps; track $index; let last = $last) {
        <button
          class="step-btn"
          [class.active]="selectedIndex === $index"
          [class.completed]="step.completed"
          [attr.aria-current]="selectedIndex === $index ? 'step' : null"
          (click)="selectedIndex = $index">
          <span class="step-number">{{ $index + 1 }}</span>
          <span class="step-label">{{ step.label }}</span>
        </button>
        @if (!last) { <span class="divider" aria-hidden="true">›</span> }
      }
    </nav>

    <div class="step-content">
      @if (selected) {
        <ng-container [ngTemplateOutlet]="selected.content" />
      }
    </div>
  `,
  styles: [`
    .stepper-header { display: flex; align-items: center; gap: 0.5rem; margin-bottom: 1.5rem; }
    .step-btn { border: none; background: none; cursor: pointer; display: flex; flex-direction: column;
                align-items: center; gap: 0.25rem; opacity: 0.4; transition: opacity 0.2s; }
    .step-btn.active, .step-btn.completed { opacity: 1; }
    .step-number { width: 2rem; height: 2rem; border-radius: 50%; background: #e0e0e0;
                   display: grid; place-items: center; font-weight: 600; }
    .step-btn.active .step-number { background: #1976d2; color: white; }
    .step-btn.completed .step-number { background: #388e3c; color: white; }
    .divider { color: #9e9e9e; }
    .step-content { padding: 1rem 0; }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class AppStepperComponent extends CdkStepper {}
```

```typescript
// wizard.component.ts
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { CdkStepperModule } from '@angular/cdk/stepper';
import { AppStepperComponent } from './stepper.component';

@Component({
  selector: 'app-wizard',
  standalone: true,
  imports: [ReactiveFormsModule, CdkStepperModule, AppStepperComponent],
  template: `
    <app-stepper #stepper="cdkStepper" [linear]="true">
      <cdk-step [stepControl]="step1" label="Persönlich">
        <form [formGroup]="step1" class="step-form">
          <h3>Persönliche Daten</h3>
          <label>
            Name
            <input formControlName="name" placeholder="Max Mustermann" />
          </label>
          <label>
            E-Mail
            <input formControlName="email" type="email" placeholder="max@example.com" />
          </label>
          <div class="actions">
            <button type="button" cdkStepperNext [disabled]="step1.invalid">
              Weiter →
            </button>
          </div>
        </form>
      </cdk-step>

      <cdk-step [stepControl]="step2" label="Adresse">
        <form [formGroup]="step2" class="step-form">
          <h3>Adresse</h3>
          <label>
            Stadt
            <input formControlName="city" placeholder="Berlin" />
          </label>
          <label>
            PLZ
            <input formControlName="zip" placeholder="10115" maxlength="5" />
          </label>
          <div class="actions">
            <button type="button" cdkStepperPrevious>← Zurück</button>
            <button type="button" cdkStepperNext [disabled]="step2.invalid">
              Weiter →
            </button>
          </div>
        </form>
      </cdk-step>

      <cdk-step [stepControl]="step3" label="Sicherheit">
        <form [formGroup]="step3" class="step-form">
          <h3>Passwort festlegen</h3>
          <label>
            Passwort (min. 8 Zeichen)
            <input formControlName="password" type="password" />
          </label>
          <label>
            Wiederholen
            <input formControlName="confirm" type="password" />
          </label>
          @if (step3.errors?.['mismatch']) {
            <p class="error">Passwörter stimmen nicht überein.</p>
          }
          <div class="actions">
            <button type="button" cdkStepperPrevious>← Zurück</button>
            <button type="button" (click)="onSubmit()" [disabled]="step3.invalid">
              ✓ Abschicken
            </button>
          </div>
        </form>
      </cdk-step>
    </app-stepper>

    @if (submitted) {
      <div class="success">
        Registrierung erfolgreich! 🎉
        <button (click)="reset(stepper)">Neu starten</button>
      </div>
    }
  `,
})
export class WizardComponent {
  private fb = inject(FormBuilder);
  submitted = false;

  step1 = this.fb.group({
    name:  ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
  });

  step2 = this.fb.group({
    city: ['', Validators.required],
    zip:  ['', [Validators.required, Validators.pattern(/^\d{5}$/)]],
  });

  step3 = this.fb.group(
    {
      password: ['', [Validators.required, Validators.minLength(8)]],
      confirm:  ['', Validators.required],
    },
    { validators: (g) => g.value.password === g.value.confirm ? null : { mismatch: true } }
  );

  onSubmit() {
    if (this.step1.valid && this.step2.valid && this.step3.valid) {
      const data = { ...this.step1.value, ...this.step2.value, password: this.step3.value.password };
      console.log('Wizard-Ergebnis:', data);
      this.submitted = true;
    }
  }

  reset(stepper: AppStepperComponent) {
    stepper.reset();
    this.step1.reset();
    this.step2.reset();
    this.step3.reset();
    this.submitted = false;
  }
}
```

## Weiterführendes
- **`CdkStepper` Quellcode lesen**: Der Quellcode in `@angular/cdk/stepper` zeigt, wie `_stateChanged()`, `_getIndicatorType()` und die ARIA-Rollen intern funktionieren – ideal, um eigene Erweiterungen sicher zu implementieren.
- **Animationen**: Kombiniere den Custom Stepper mit `@angular/animations` und `CdkStepper.animationDone` (EventEmitter), um Slide-in/out-Übergänge zwischen Schritten zu realisieren.
- **Persistenz**: Speichere `selectedIndex` und Formwerte im `sessionStorage` (z.B. mit einem `FormPersistenceService`), damit der Nutzer nach einem Tab-Reload an der letzten Stelle weitermachen kann.
