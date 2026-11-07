# Angular Dojo: Angular CDK Stepper
**Datum:** 2026-11-07
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem **Angular CDK Stepper** einen vollständig barrierefreien, formulargesteuerten Schritt-für-Schritt-Assistenten (Wizard) baust – ohne auf Angular Material angewiesen zu sein, aber mit eigenen Styles und Validierungslogik pro Schritt.

## Hintergrund & Theorie

Das Angular CDK enthält in `@angular/cdk/stepper` eine abstrakte Stepper-Implementierung, die das gesamte Keyboard-Handling, ARIA-Attribute und die Schritt-Navigation übernimmt. Angular Material baut darauf auf – du kannst es aber auch direkt nutzen und vollständig eigenes Styling einsetzen.

**Kernklassen:**
- `CdkStepper` – abstrakte Basisklasse, die du per `extends` oder Template-Referenz ansteuerst
- `CdkStep` – eine einzelne Schritt-Komponente mit `stepControl`, `label`, `optional`
- `CdkStepLabel` – alternative Template-basierte Labels (`<ng-template cdkStepLabel>`)
- `CdkStepperNext` / `CdkStepperPrevious` – Direktiven für Navigations-Buttons

**Validierung pro Schritt:**
Jeder `CdkStep` akzeptiert ein `stepControl`-Property (ein `AbstractControl`). Ist das Control invalid, kann der Nutzer nicht zum nächsten Schritt – es sei denn, `linear` ist auf `false` gesetzt.

```typescript
@Component({
  selector: 'app-custom-stepper',
  hostDirectives: [],
  providers: [{ provide: CdkStepper, useExisting: CustomStepperComponent }],
})
export class CustomStepperComponent extends CdkStepper {
  selectStepByIndex(index: number): void {
    this.selectedIndex = index;
  }
}
```

Das `[linear]="true"`-Flag auf dem Host-Stepper erzwingt sequentiellen Fortschritt und ist Standard für Formulare.

## Aufgabe

Baue einen **3-Schritt-Registrierungsassistenten** mit einem eigenen Stepper auf Basis des Angular CDK. Jeder Schritt hat sein eigenes Reactive-Form-Stück, und der Nutzer kann erst weiter, wenn der aktuelle Schritt valide ist.

**Anforderungen:**
- Eigene `CustomStepperComponent`, die `CdkStepper` erweitert
- Schritt 1: E-Mail + Passwort (required, minLength: 8)
- Schritt 2: Vorname + Nachname (beide required)
- Schritt 3: Zusammenfassung der eingegebenen Daten (read-only)
- Navigations-Buttons „Weiter" und „Zurück"
- Aktiver Schritt visuell hervorgehoben (z. B. per CSS-Klasse)
- Der „Weiter"-Button ist disabled, solange der aktuelle Schritt invalid ist

### Schritte

1. `@angular/cdk/stepper` importieren und `CustomStepperComponent` erstellen, die `CdkStepper` extended und sich selbst als Provider registriert.
2. Die drei `CdkStep`-Instanzen im Template mit `stepControl` verdrahten.
3. Drei `FormGroup`-Instanzen (z. B. via `FormBuilder`) in der Host-Komponente anlegen und an die Steps binden.
4. Einen einfachen Schritt-Header bauen, der `selectedIndex` liest und den aktiven Schritt hervorhebt.
5. `CdkStepperNext` und `CdkStepperPrevious` als Button-Direktiven nutzen; „Weiter" per `[disabled]` an die Form-Validität binden.
6. Im letzten Schritt alle gesammelten Werte aus den drei Forms anzeigen.

## Hints

<details>
<summary>Hint 1 – CdkStepper als eigene Komponente</summary>

```typescript
import { CdkStepper } from '@angular/cdk/stepper';

@Component({
  selector: 'app-custom-stepper',
  standalone: true,
  imports: [CdkStepperModule],
  template: `
    <div class="stepper-header">
      @for (step of steps; track $index) {
        <span
          class="step-indicator"
          [class.active]="selectedIndex === $index"
          (click)="!linear && selectStepByIndex($index)"
        >{{ $index + 1 }}</span>
      }
    </div>
    <ng-container [ngTemplateOutlet]="selected?.content ?? null" />
    <ng-content />
  `,
  providers: [{ provide: CdkStepper, useExisting: CustomStepperComponent }],
})
export class CustomStepperComponent extends CdkStepper {
  selectStepByIndex(index: number): void {
    this.selectedIndex = index;
  }
}
```
</details>

<details>
<summary>Hint 2 – Steps mit stepControl im Template der Host-Komponente</summary>

```typescript
@Component({
  standalone: true,
  imports: [CustomStepperComponent, CdkStepperModule, ReactiveFormsModule],
  template: `
    <app-custom-stepper [linear]="true">
      <cdk-step [stepControl]="step1Form">
        <form [formGroup]="step1Form">
          <input formControlName="email" placeholder="E-Mail" />
          <input formControlName="password" type="password" placeholder="Passwort" />
        </form>
        <button cdkStepperNext [disabled]="step1Form.invalid">Weiter</button>
      </cdk-step>

      <cdk-step [stepControl]="step2Form">
        <form [formGroup]="step2Form">
          <input formControlName="firstName" placeholder="Vorname" />
          <input formControlName="lastName" placeholder="Nachname" />
        </form>
        <button cdkStepperPrevious>Zurück</button>
        <button cdkStepperNext [disabled]="step2Form.invalid">Weiter</button>
      </cdk-step>

      <cdk-step>
        <p>E-Mail: {{ step1Form.value.email }}</p>
        <p>Name: {{ step2Form.value.firstName }} {{ step2Form.value.lastName }}</p>
        <button cdkStepperPrevious>Zurück</button>
        <button (click)="submit()">Registrieren</button>
      </cdk-step>
    </app-custom-stepper>
  `,
})
export class RegistrationWizardComponent {
  private fb = inject(FormBuilder);

  step1Form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
  });

  step2Form = this.fb.group({
    firstName: ['', Validators.required],
    lastName:  ['', Validators.required],
  });

  submit() {
    console.log({ ...this.step1Form.value, ...this.step2Form.value });
  }
}
```
</details>

## Beispiellösung

```typescript
// custom-stepper.component.ts
import { Component } from '@angular/core';
import { CdkStepper, CdkStepperModule } from '@angular/cdk/stepper';
import { NgTemplateOutlet } from '@angular/common';

@Component({
  selector: 'app-custom-stepper',
  standalone: true,
  imports: [CdkStepperModule, NgTemplateOutlet],
  providers: [{ provide: CdkStepper, useExisting: CustomStepperComponent }],
  styles: [`
    .stepper-header { display: flex; gap: 8px; margin-bottom: 16px; }
    .step-indicator {
      width: 32px; height: 32px; border-radius: 50%;
      display: grid; place-items: center;
      background: #e0e0e0; cursor: default;
    }
    .step-indicator.active { background: #3f51b5; color: #fff; }
    .step-indicator.completed { background: #4caf50; color: #fff; }
  `],
  template: `
    <div class="stepper-header">
      @for (step of steps; track $index) {
        <span
          class="step-indicator"
          [class.active]="selectedIndex === $index"
          [class.completed]="$index < selectedIndex"
        >{{ $index + 1 }}</span>
      }
    </div>
    <ng-container [ngTemplateOutlet]="selected?.content ?? null" />
    <ng-content />
  `,
})
export class CustomStepperComponent extends CdkStepper {}

// registration-wizard.component.ts
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { CdkStepperModule } from '@angular/cdk/stepper';
import { CustomStepperComponent } from './custom-stepper.component';

@Component({
  selector: 'app-registration-wizard',
  standalone: true,
  imports: [CustomStepperComponent, CdkStepperModule, ReactiveFormsModule],
  template: `
    <app-custom-stepper [linear]="true">

      <!-- Schritt 1: Zugangsdaten -->
      <cdk-step [stepControl]="step1Form" label="Zugangsdaten">
        <form [formGroup]="step1Form" style="display:flex;flex-direction:column;gap:8px">
          <input formControlName="email"    placeholder="E-Mail" />
          <input formControlName="password" type="password" placeholder="Passwort (min. 8 Zeichen)" />
        </form>
        <div style="margin-top:12px">
          <button cdkStepperNext [disabled]="step1Form.invalid">Weiter →</button>
        </div>
      </cdk-step>

      <!-- Schritt 2: Persönliche Daten -->
      <cdk-step [stepControl]="step2Form" label="Persönliche Daten">
        <form [formGroup]="step2Form" style="display:flex;flex-direction:column;gap:8px">
          <input formControlName="firstName" placeholder="Vorname" />
          <input formControlName="lastName"  placeholder="Nachname" />
        </form>
        <div style="margin-top:12px;display:flex;gap:8px">
          <button cdkStepperPrevious>← Zurück</button>
          <button cdkStepperNext [disabled]="step2Form.invalid">Weiter →</button>
        </div>
      </cdk-step>

      <!-- Schritt 3: Zusammenfassung -->
      <cdk-step label="Zusammenfassung">
        <p><strong>E-Mail:</strong> {{ step1Form.value.email }}</p>
        <p><strong>Name:</strong> {{ step2Form.value.firstName }} {{ step2Form.value.lastName }}</p>
        <div style="margin-top:12px;display:flex;gap:8px">
          <button cdkStepperPrevious>← Zurück</button>
          <button (click)="submit()">Registrieren ✓</button>
        </div>
      </cdk-step>

    </app-custom-stepper>
  `,
})
export class RegistrationWizardComponent {
  private fb = inject(FormBuilder);

  step1Form = this.fb.group({
    email:    ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
  });

  step2Form = this.fb.group({
    firstName: ['', Validators.required],
    lastName:  ['', Validators.required],
  });

  submit(): void {
    const payload = {
      ...this.step1Form.getRawValue(),
      ...this.step2Form.getRawValue(),
    };
    console.log('Registrierung:', payload);
    alert(`Willkommen, ${payload.firstName}!`);
  }
}
```

## Weiterführendes

- Für komplexere Nutzung: [`CdkStep.interacted`](https://material.angular.io/cdk/stepper/api#CdkStep) auswerten, um "touched"-Zustände zu visualisieren (Fehler-Icons im Header).
- Mit `StepperSelectionEvent` (Output `(selectionChange)` von `CdkStepper`) kannst du Analytics-Events pro Schritt-Wechsel abfeuern.
- Schau dir [`CdkStepHeader`](https://material.angular.io/cdk/stepper/api#CdkStepHeader) an, um vollständig barrierefreie Step-Indikatoren zu bauen (inkl. `aria-selected`, `aria-disabled`).
- Offizielle Doku: https://material.angular.io/cdk/stepper/overview
