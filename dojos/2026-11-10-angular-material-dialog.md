# Angular Dojo: Angular Material Dialog – Typed Dialoge mit MatDialogRef
**Datum:** 2026-11-10
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular Material Dialoge vollständig typsicher implementierst – mit generisch typisiertem `MatDialogRef<C, R>`, strukturierter Datenweitergabe via `MAT_DIALOG_DATA` und signal-basiertem Reaktionsmuster auf Dialogergebnisse.

## Hintergrund & Theorie

Angular Material's `MatDialog`-Service ermöglicht es, beliebige Komponenten als modale Dialoge zu öffnen. Die Stärke liegt in der Typsicherheit: `MatDialogRef<DialogComponent, ReturnType>` legt fest, welchen Typ der Dialog zurückgibt. `MAT_DIALOG_DATA` ist ein `InjectionToken`, das strukturierte Eingabedaten in die Dialogkomponente transportiert.

Seit Angular 17+ gibt es zwei bevorzugte Patterns:
1. **Typed Dialog Data & Result**: Interfaces für Input (`DialogData`) und Output (`DialogResult`) definieren, damit TypeScript den gesamten Dialog-Lifecycle absichert.
2. **Signal-basiertes Ergebnis**: `.afterClosed()` gibt ein `Observable<R | undefined>` zurück, das mit `toSignal()` in ein Signal überführt werden kann, um reaktiv auf den Dialogschluss zu reagieren.

Ein häufiger Fehler ist, `dialog.open()` untypisiert aufzurufen und das Ergebnis mit `any` zu behandeln – das führt zu schwer debugbaren Laufzeitfehlern. Die korrekte Typisierung über Generics verhindert das.

## Aufgabe

Erstelle einen vollständig typisierten „Bestätigungsdialog" (`ConfirmDialogComponent`), der eine Frage und zwei konfigurierbare Button-Labels anzeigt. Die aufrufende Komponente öffnet den Dialog und reagiert reaktiv (mit Signals) auf das Ergebnis.

### Schritte

1. **Dialog-Interfaces definieren**: Erstelle `ConfirmDialogData` (mit `title`, `message`, `confirmLabel`, `cancelLabel`) und `ConfirmDialogResult` (mit `confirmed: boolean`).

2. **`ConfirmDialogComponent` implementieren**: Die Komponente injiziert `MAT_DIALOG_DATA` als `ConfirmDialogData` und `MatDialogRef<ConfirmDialogComponent, ConfirmDialogResult>`. Die Buttons rufen `dialogRef.close({ confirmed: true/false })` auf.

3. **Aufrufende Komponente bauen**: Injiziere `MatDialog` und öffne den Dialog mit korrektem Typ: `this.dialog.open<ConfirmDialogComponent, ConfirmDialogData, ConfirmDialogResult>(ConfirmDialogComponent, { data: { ... } })`. Leite das `afterClosed()`-Observable mit `toSignal()` in ein Signal um.

4. **Reaktiv auf Ergebnis reagieren**: Nutze `effect()`, um auf das Ergebnis-Signal zu reagieren und eine Statusmeldung anzuzeigen.

5. **Keyboard-Zugänglichkeit sicherstellen**: Setze `cdkFocusInitial` auf den primären Button und stelle sicher, dass `Escape` den Dialog mit `{ confirmed: false }` schließt (via `disableClose: false` und `backdropClass`).

## Hints

<details>
<summary>Hint 1 – MAT_DIALOG_DATA typisiert injizieren</summary>

```typescript
// Statt:
constructor(@Inject(MAT_DIALOG_DATA) public data: any) {}

// Besser mit inject():
readonly data = inject<ConfirmDialogData>(MAT_DIALOG_DATA);
readonly dialogRef = inject<MatDialogRef<ConfirmDialogComponent, ConfirmDialogResult>>(
  MatDialogRef
);
```

</details>

<details>
<summary>Hint 2 – afterClosed() mit toSignal() verbinden</summary>

```typescript
// In der aufrufenden Komponente:
openDialog(): void {
  const ref = this.dialog.open<
    ConfirmDialogComponent,
    ConfirmDialogData,
    ConfirmDialogResult
  >(ConfirmDialogComponent, {
    data: { title: 'Löschen?', message: 'Wirklich löschen?',
            confirmLabel: 'Löschen', cancelLabel: 'Abbrechen' },
    width: '400px',
  });

  // afterClosed$ als Signal
  this.lastResult = toSignal(ref.afterClosed(), { initialValue: undefined });
}
```

</details>

<details>
<summary>Hint 3 – effect() für Ergebnisreaktion</summary>

```typescript
constructor() {
  effect(() => {
    const result = this.lastResult();
    if (result?.confirmed) {
      this.statusMessage.set('Eintrag wurde gelöscht.');
    } else if (result !== undefined) {
      this.statusMessage.set('Abgebrochen.');
    }
  });
}
```

Achtung: `toSignal()` auf einem `afterClosed()`-Observable gibt nach Dialogschluss einen Wert, aber das Signal bleibt danach auf diesem Wert stehen. Für mehrere Dialoge brauchst du entweder ein neues Signal pro Dialog oder ein `BehaviorSubject`.

</details>

## Beispiellösung

```typescript
// confirm-dialog.types.ts
export interface ConfirmDialogData {
  title: string;
  message: string;
  confirmLabel: string;
  cancelLabel: string;
}

export interface ConfirmDialogResult {
  confirmed: boolean;
}

// confirm-dialog.component.ts
import { Component, inject } from '@angular/core';
import { MAT_DIALOG_DATA, MatDialogRef, MatDialogModule } from '@angular/material/dialog';
import { MatButtonModule } from '@angular/material/button';
import { ConfirmDialogData, ConfirmDialogResult } from './confirm-dialog.types';

@Component({
  selector: 'app-confirm-dialog',
  standalone: true,
  imports: [MatDialogModule, MatButtonModule],
  template: `
    <h2 mat-dialog-title>{{ data.title }}</h2>
    <mat-dialog-content>{{ data.message }}</mat-dialog-content>
    <mat-dialog-actions align="end">
      <button mat-button (click)="close(false)">{{ data.cancelLabel }}</button>
      <button mat-flat-button color="warn" cdkFocusInitial (click)="close(true)">
        {{ data.confirmLabel }}
      </button>
    </mat-dialog-actions>
  `,
})
export class ConfirmDialogComponent {
  readonly data = inject<ConfirmDialogData>(MAT_DIALOG_DATA);
  private readonly dialogRef =
    inject<MatDialogRef<ConfirmDialogComponent, ConfirmDialogResult>>(MatDialogRef);

  close(confirmed: boolean): void {
    this.dialogRef.close({ confirmed });
  }
}

// item-list.component.ts
import { Component, inject, signal, effect } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { MatDialog } from '@angular/material/dialog';
import { MatButtonModule } from '@angular/material/button';
import { of } from 'rxjs';
import { ConfirmDialogComponent } from './confirm-dialog.component';
import type { ConfirmDialogData, ConfirmDialogResult } from './confirm-dialog.types';

@Component({
  selector: 'app-item-list',
  standalone: true,
  imports: [MatButtonModule],
  template: `
    <button mat-raised-button color="primary" (click)="confirmDelete()">
      Eintrag löschen
    </button>
    @if (statusMessage()) {
      <p>{{ statusMessage() }}</p>
    }
  `,
})
export class ItemListComponent {
  private readonly dialog = inject(MatDialog);

  readonly statusMessage = signal('');

  // Hält das zuletzt beobachtete Ergebnis; initial undefined
  private lastResult = signal<ConfirmDialogResult | undefined>(undefined);

  constructor() {
    effect(() => {
      const result = this.lastResult();
      if (result === undefined) return;
      this.statusMessage.set(result.confirmed ? 'Eintrag gelöscht!' : 'Abgebrochen.');
    });
  }

  confirmDelete(): void {
    const ref = this.dialog.open<
      ConfirmDialogComponent,
      ConfirmDialogData,
      ConfirmDialogResult
    >(ConfirmDialogComponent, {
      data: {
        title: 'Eintrag löschen',
        message: 'Dieser Vorgang kann nicht rückgängig gemacht werden.',
        confirmLabel: 'Löschen',
        cancelLabel: 'Abbrechen',
      },
      width: '420px',
      autoFocus: 'dialog',
    });

    // Ergebnis in ein Signal umwandeln und speichern
    ref.afterClosed().subscribe((result) => {
      this.lastResult.set(result ?? { confirmed: false });
    });
  }
}
```

## Weiterführendes

- **Multiple Dialoge verwalten**: Nutze `MatDialog.openDialogs` (Array aller offenen `MatDialogRef`s) und `MatDialog.afterAllClosed` (Observable, das feuert wenn alle Dialoge geschlossen sind) für komplexe UI-Flows.
- **Dialog mit Formularen**: Kombiniere den Dialog mit einem Reactive Form und gib das Form-Ergebnis als `DialogResult` zurück – validiere dabei innerhalb des Dialogs bevor `dialogRef.close()` aufgerufen wird.
- Offizielle Doku: [material.angular.io/components/dialog](https://material.angular.io/components/dialog/overview)
