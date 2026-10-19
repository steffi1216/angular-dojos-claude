# Angular Dojo: ControlContainer & Sub-Form Komponenten
**Datum:** 2026-10-19
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `ControlContainer` wiederverwendbare Sub-Form-Komponenten erstellst, die sich nahtlos in übergeordnete Reactive Forms einklinken – ohne `@Input()`-Prop-Drilling und ohne den Formzustand zu duplizieren.

## Hintergrund & Theorie
In großen Formularen entstehen oft mehrere zusammengehörige Felder, die man in eigene Komponenten auslagern möchte (z. B. eine `AddressFormComponent`). Das naive Vorgehen: eine eigene `FormGroup` per `@Input()` hineingeben. Das hat jedoch Nachteile – enge Kopplung, umständliches Nesting.

Die elegante Lösung: **`ControlContainer`**. Dieses DI-Token liefert der Kindkomponente Zugriff auf die umgebende `FormGroup` oder `FormArray` des Elternteils. Indem man den `ControlContainer` über `viewProviders` bereitstellt, wird die Kindkomponente zu einem echten Teil des Parent-Formulars:

- Kein gesondertes `FormGroup`-Input nötig
- Validierungsstatus, dirty/touched, reset – alles synchron mit dem Parent
- Funktioniert mit `formGroupName` / `formArrayName` auf dem Host-Element der Kindkomponente
- Benötigt Angular ≥ 14 (standalone) oder jede ältere Version mit `ReactiveFormsModule`

```typescript
// Im Parent-Template:
<form [formGroup]="orderForm">
  <app-address-form formGroupName="shippingAddress"></app-address-form>
</form>

// In der Kindkomponente:
viewProviders: [{ provide: ControlContainer, useExisting: FormGroupDirective }]
```

## Aufgabe
Erstelle ein Bestellformular (`OrderFormComponent`) mit zwei wiederverwendbaren Adressblöcken (Liefer- und Rechnungsadresse) als eigene Komponente `AddressFormComponent`. Die Kindkomponente soll sich über `ControlContainer` in die übergeordnete `FormGroup` einklinken.

### Schritte
1. Erstelle `OrderFormComponent` mit einer `FormGroup` die zwei Untergruppen enthält: `shippingAddress` und `billingAddress`.
2. Erstelle `AddressFormComponent` als Standalone-Komponente mit `viewProviders: [{ provide: ControlContainer, useExisting: FormGroupDirective }]`.
3. Baue in `AddressFormComponent` die Felder `street`, `city` und `zip` mit `formControlName` – ohne eine eigene `FormGroup` zu erstellen.
4. Binde `<app-address-form formGroupName="shippingAddress">` und `<app-address-form formGroupName="billingAddress">` im Parent-Template ein.
5. Zeige den Gesamtstatus (`orderForm.valid`, `orderForm.value`) im Parent an und verifiziere, dass Änderungen in der Kindkomponente den Parent-State korrekt aktualisieren.
6. Füge in `AddressFormComponent` einen Custom Validator hinzu, der prüft, ob `zip` genau 5 Ziffern enthält – der Fehlerstatus soll im Parent-Formular sichtbar sein.

## Hints
<details>
<summary>Hint 1 – viewProviders vs. providers</summary>

`viewProviders` statt `providers` verwenden! Nur `viewProviders` macht den `ControlContainer` innerhalb der *View* der Kindkomponente sichtbar, ohne die Content Projection zu beeinflussen. Mit `providers` würde Angular einen neuen `ControlContainer` erzeugen statt den vom Parent zu erben.

```typescript
@Component({
  viewProviders: [
    { provide: ControlContainer, useExisting: FormGroupDirective }
  ]
})
export class AddressFormComponent {}
```

</details>
<details>
<summary>Hint 2 – inject() statt Constructor-DI für den ControlContainer</summary>

Du kannst den `ControlContainer` über `inject()` im Feld oder im Constructor holen, um die aktuelle `FormGroup` zu lesen:

```typescript
export class AddressFormComponent implements OnInit {
  private controlContainer = inject(ControlContainer);

  ngOnInit() {
    // Die formGroup des Parents, eingeschränkt auf diesen formGroupName
    const group = this.controlContainer.control as FormGroup;
    console.log(group.controls); // street, city, zip
  }
}
```

Das ist nützlich, wenn du programmatisch auf die Kindgruppe zugreifen musst, z. B. um dynamische Controls hinzuzufügen.

</details>

## Beispiellösung

```typescript
// address-form.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { ReactiveFormsModule, ControlContainer, FormGroupDirective, FormGroup, Validators, AbstractControl, ValidationErrors } from '@angular/forms';

function zipCodeValidator(control: AbstractControl): ValidationErrors | null {
  return /^\d{5}$/.test(control.value ?? '') ? null : { invalidZip: true };
}

@Component({
  selector: 'app-address-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <ng-container [formGroup]="addressGroup">
      <label>
        Straße
        <input formControlName="street" />
      </label>
      <label>
        Stadt
        <input formControlName="city" />
      </label>
      <label>
        PLZ
        <input formControlName="zip" />
        @if (addressGroup.get('zip')?.errors?.['invalidZip'] && addressGroup.get('zip')?.touched) {
          <span class="error">PLZ muss 5 Ziffern haben</span>
        }
      </label>
    </ng-container>
  `,
  viewProviders: [
    { provide: ControlContainer, useExisting: FormGroupDirective }
  ]
})
export class AddressFormComponent implements OnInit {
  private controlContainer = inject(ControlContainer);
  addressGroup!: FormGroup;

  ngOnInit() {
    this.addressGroup = this.controlContainer.control as FormGroup;
  }
}

// order-form.component.ts
import { Component } from '@angular/core';
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';
import { AddressFormComponent } from './address-form.component';

@Component({
  selector: 'app-order-form',
  standalone: true,
  imports: [ReactiveFormsModule, AddressFormComponent],
  template: `
    <form [formGroup]="orderForm" (ngSubmit)="onSubmit()">
      <h2>Lieferadresse</h2>
      <app-address-form formGroupName="shippingAddress"></app-address-form>

      <h2>Rechnungsadresse</h2>
      <app-address-form formGroupName="billingAddress"></app-address-form>

      <button type="submit" [disabled]="orderForm.invalid">Bestellen</button>

      <pre>Status: {{ orderForm.status }}</pre>
      <pre>{{ orderForm.value | json }}</pre>
    </form>
  `
})
export class OrderFormComponent {
  private fb = inject(FormBuilder);

  orderForm = this.fb.group({
    shippingAddress: this.fb.group({
      street: ['', Validators.required],
      city:   ['', Validators.required],
      zip:    ['', [Validators.required, zipCodeValidator]]
    }),
    billingAddress: this.fb.group({
      street: ['', Validators.required],
      city:   ['', Validators.required],
      zip:    ['', [Validators.required, zipCodeValidator]]
    })
  });

  onSubmit() {
    if (this.orderForm.valid) {
      console.log(this.orderForm.value);
    }
  }
}
```

## Weiterführendes
- **`ControlContainer` mit `FormArrayName`**: Das gleiche Prinzip funktioniert auch für `FormArray` – ideal für dynamische Listen von Adressblöcken (z. B. mehrere Lieferadressen).
- **Mit Signal-based Forms (Angular 20+)**: Beobachte, wie sich das `ControlContainer`-Muster mit dem kommenden Signals-basierten Formular-API entwickeln wird – die Grundidee der Komposition bleibt gleich.
- [Angular Docs – Reactive Forms](https://angular.dev/guide/forms/reactive-forms)
