# Angular Dojo: Accessibility (a11y) mit CDK
**Datum:** 2026-08-07
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK a11y-Modul barrierefreie Komponenten baust — inklusive Focus Management, ARIA Live Regions und Keyboard-Navigation.

## Hintergrund & Theorie
Barrierefreiheit ist kein optionales Add-on, sondern ein Qualitätsmerkmal professioneller Anwendungen. Das Angular CDK stellt im Paket `@angular/cdk/a11y` mächtige Werkzeuge bereit:

- **`FocusTrap`** – sperrt den Fokus innerhalb eines Dialogs oder Overlays, damit Tab-Navigation nicht aus dem aktiven Bereich ausbricht (wichtig für Modale).
- **`LiveAnnouncer`** – schreibt Nachrichten in eine ARIA-Live-Region, sodass Screenreader sie vorlesen, ohne dass der Fokus wechselt. Ideal für Status-Updates (z. B. „3 Ergebnisse geladen").
- **`FocusMonitor`** – verfolgt, wie ein Element fokussiert wurde (Maus, Tastatur, Touch, Programm). Damit lassen sich Fokus-Ringe nur bei Tastaturnutzung anzeigen.
- **`CdkTrapFocus`** (Direktive) – deklarative Alternative zu `FocusTrap` direkt im Template.

Kombiniert mit semantischem HTML (`role`, `aria-label`, `aria-describedby`) und korrekter Tastatur-Reihenfolge (`tabindex`) entstehen Komponenten, die mit Screenreadern, nur per Tastatur und in barrierefreien Umgebungen funktionieren.

## Aufgabe
Baue eine barrierefreie **Modal-Dialog-Komponente** von Grund auf — ohne Angular Material, nur mit CDK a11y.

### Schritte
1. **Setup**: Stelle sicher, dass `@angular/cdk` installiert ist (`npm i @angular/cdk`). Importiere `A11yModule` in deiner Standalone-Komponente.
2. **Grundstruktur**: Erstelle eine `AccessibleModalComponent` mit einem Trigger-Button und einem Dialog-Overlay (`<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">`).
3. **Focus Trap**: Verwende die `cdkTrapFocus`-Direktive auf dem Dialog-Container und setze `cdkTrapFocusAutoCapture="true"`, damit der Fokus beim Öffnen automatisch in den Dialog springt.
4. **Keyboard-Handler**: Schließe den Dialog per `Escape`-Taste (`@HostListener('keydown.escape')`).
5. **Focus Restore**: Merke das zuvor fokussierte Element (`document.activeElement`) und gib ihm den Fokus nach dem Schließen zurück.
6. **LiveAnnouncer**: Nutze `LiveAnnouncer.announce()`, um beim Öffnen „Dialog geöffnet" und beim Schließen „Dialog geschlossen" anzukündigen.
7. **Teste** die Komponente mit der Tastatur (Tab, Shift+Tab, Escape) und einem Screenreader oder dem NVDA-Simulator im Browser.

## Hints
<details>
<summary>Hint 1 – FocusTrap manuell verwenden</summary>

Falls du den Focus Trap programmatisch steuern willst, injiziere `FocusTrapFactory`:

```typescript
import { FocusTrapFactory, FocusTrap } from '@angular/cdk/a11y';

export class AccessibleModalComponent implements OnDestroy {
  private focusTrap?: FocusTrap;
  private dialogRef = viewChild.required<ElementRef>('dialog');

  constructor(private focusTrapFactory: FocusTrapFactory) {}

  open() {
    this.focusTrap = this.focusTrapFactory.create(this.dialogRef().nativeElement);
    this.focusTrap.focusInitialElementWhenReady();
  }

  close() {
    this.focusTrap?.destroy();
  }
}
```
</details>

<details>
<summary>Hint 2 – LiveAnnouncer und Focus Restore</summary>

```typescript
import { LiveAnnouncer } from '@angular/cdk/a11y';

export class AccessibleModalComponent {
  private liveAnnouncer = inject(LiveAnnouncer);
  private previousFocus: HTMLElement | null = null;

  openDialog() {
    this.previousFocus = document.activeElement as HTMLElement;
    this.isOpen = true;
    this.liveAnnouncer.announce('Dialog geöffnet', 'assertive');
  }

  closeDialog() {
    this.isOpen = false;
    this.liveAnnouncer.announce('Dialog geschlossen', 'polite');
    // Fokus zurückgeben, nachdem Angular das DOM aktualisiert hat
    setTimeout(() => this.previousFocus?.focus(), 0);
  }
}
```
</details>

## Beispiellösung

```typescript
import {
  Component, signal, inject, HostListener, ElementRef, viewChild
} from '@angular/core';
import { A11yModule, LiveAnnouncer } from '@angular/cdk/a11y';

@Component({
  selector: 'app-accessible-modal',
  standalone: true,
  imports: [A11yModule],
  template: `
    <button
      (click)="openDialog()"
      aria-haspopup="dialog">
      Einstellungen öffnen
    </button>

    @if (isOpen()) {
      <div
        class="backdrop"
        (click)="closeDialog()"
        aria-hidden="true">
      </div>

      <div
        #dialogEl
        cdkTrapFocus
        cdkTrapFocusAutoCapture
        role="dialog"
        aria-modal="true"
        aria-labelledby="dialog-title"
        aria-describedby="dialog-desc"
        class="dialog">

        <h2 id="dialog-title">Einstellungen</h2>
        <p id="dialog-desc">
          Passe deine Anwendungseinstellungen an.
        </p>

        <label>
          Benachrichtigungen
          <input type="checkbox" />
        </label>

        <label>
          Sprache
          <select>
            <option>Deutsch</option>
            <option>English</option>
          </select>
        </label>

        <div class="dialog-actions">
          <button (click)="saveAndClose()">Speichern</button>
          <button (click)="closeDialog()">Abbrechen</button>
        </div>
      </div>
    }
  `,
  styles: [`
    .backdrop {
      position: fixed; inset: 0;
      background: rgba(0,0,0,0.5);
      z-index: 100;
    }
    .dialog {
      position: fixed;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      background: white;
      border-radius: 8px;
      padding: 24px;
      min-width: 320px;
      z-index: 101;
      display: flex;
      flex-direction: column;
      gap: 16px;
    }
    .dialog-actions {
      display: flex;
      gap: 8px;
      justify-content: flex-end;
    }
  `]
})
export class AccessibleModalComponent {
  isOpen = signal(false);

  private liveAnnouncer = inject(LiveAnnouncer);
  private previousFocus: HTMLElement | null = null;

  @HostListener('keydown.escape')
  onEscape() {
    if (this.isOpen()) {
      this.closeDialog();
    }
  }

  openDialog() {
    this.previousFocus = document.activeElement as HTMLElement;
    this.isOpen.set(true);
    this.liveAnnouncer.announce('Einstellungsdialog geöffnet', 'assertive');
  }

  closeDialog() {
    this.isOpen.set(false);
    this.liveAnnouncer.announce('Dialog geschlossen', 'polite');
    setTimeout(() => this.previousFocus?.focus(), 0);
  }

  saveAndClose() {
    // Einstellungen speichern...
    this.liveAnnouncer.announce('Einstellungen gespeichert', 'assertive');
    this.closeDialog();
  }
}
```

### Bonus: FocusMonitor für Fokus-Ring nur bei Tastatur

```typescript
import { FocusMonitor } from '@angular/cdk/a11y';
import { AfterViewInit, OnDestroy, ElementRef, inject } from '@angular/core';

// In einer Button-Komponente:
export class KeyboardFocusButtonComponent implements AfterViewInit, OnDestroy {
  private el = inject(ElementRef);
  private focusMonitor = inject(FocusMonitor);

  ngAfterViewInit() {
    this.focusMonitor.monitor(this.el, true).subscribe(origin => {
      // origin: 'keyboard' | 'mouse' | 'touch' | 'program' | null
      if (origin === 'keyboard') {
        this.el.nativeElement.classList.add('keyboard-focused');
      } else {
        this.el.nativeElement.classList.remove('keyboard-focused');
      }
    });
  }

  ngOnDestroy() {
    this.focusMonitor.stopMonitoring(this.el);
  }
}
```

## Weiterführendes
- **ARIA Authoring Practices Guide (APG)**: https://www.w3.org/WAI/ARIA/apg/ – offizielle Muster für barrierefreie Widgets (Accordions, Tabs, Dialoge etc.)
- **Angular CDK a11y Doku**: https://material.angular.io/cdk/a11y/overview
- **axe-core / @axe-core/angular**: Automatisiertes a11y-Testing in Unit- und E2E-Tests — `@axe-core/playwright` für Playwright-Tests
- **Lighthouse a11y Audit** im Chrome DevTools gibt sofortiges Feedback ohne Screenreader
- Nächste Vertiefung: **`FocusKeyManager`** für Listbox/Toolbar-Navigation mit Pfeiltasten (erlaubt "roving tabindex"-Muster)
