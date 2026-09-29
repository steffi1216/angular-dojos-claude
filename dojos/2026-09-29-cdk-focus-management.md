# Angular Dojo: CDK Focus Management
**Datum:** 2026-09-29
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, mit den CDK-Werkzeugen `FocusTrap`, `FocusMonitor` und `FocusKeyManager` vollständig tastaturzugängliche UI-Komponenten zu bauen – ohne sich auf den Browser-Default oder eigene Event-Listener zu verlassen.

## Hintergrund & Theorie

Angular CDK bietet drei spezialisierte Klassen für das Fokus-Management:

**`FocusTrap`** (via `FocusTrapFactory`) sperrt den Tastaturfokus innerhalb eines DOM-Elements ein. Wird für modale Dialoge und Drawers benötigt, damit `Tab` und `Shift+Tab` nicht aus dem Overlay hinauswandern. Das ARIA-Pattern „Dialog" schreibt dieses Verhalten vor.

**`FocusMonitor`** beobachtet, ob ein Element per Tastatur, Maus, Touch oder Programm fokussiert wird, und liefert diese Information als `FocusOrigin`. Damit lassen sich fokussierte Elemente je nach Eingabemethode unterschiedlich stylen (z. B. nur bei Tastatur-Fokus einen Outline zeigen).

**`FocusKeyManager`** implementiert das Composite-Widget-Pattern: Er verwaltet eine Liste von Elementen, die `FocusableOption` implementieren, und routet Pfeiltasten-Navigation (`↑ ↓` oder `← →`) korrekt durch die Liste – inklusive Wrap-around, Typeahead-Suche und optionaler horizontaler/vertikaler Achse.

Die drei Bausteine ergänzen sich: `FocusTrap` definiert die Grenze, `FocusKeyManager` steuert die interne Navigation, `FocusMonitor` liefert Kontext über die Fokus-Herkunft.

## Aufgabe

Erstelle eine `DropdownMenuComponent` – ein custom Dropdown-Menü, das vollständig tastaturzugänglich ist:

- Öffnet sich mit `Enter`/`Space` oder einem Klick auf den Trigger-Button.
- Schließt sich mit `Escape`.
- Navigiert Menüeinträge mit `↑`/`↓` (mit Wrap-around am Ende der Liste).
- Sperrt den Fokus innerhalb des Menü-Panels mit `FocusTrap`.
- Zeigt einen visuellen Fokus-Ring **nur** bei Tastatur-Navigation (`FocusMonitor`).

### Schritte

1. **Projekt vorbereiten**
   Installiere das CDK (falls nicht vorhanden): `npm i @angular/cdk`.
   Importiere `A11yModule` (oder die einzelnen Klassen) in deinem Standalone-Component.

2. **`MenuItemComponent` erstellen**
   Erstelle eine `MenuItemComponent`, die `FocusableOption` implementiert und über eine `focus()`-Methode verfügt. Jede Instanz kennt ihren eigenen `ElementRef`.

3. **`FocusKeyManager` einrichten**
   Hole alle `MenuItemComponent`-Instanzen mit `@ViewChildren`. Initialisiere in `ngAfterViewInit` einen `FocusKeyManager` mit `.withWrap()` und `.withTypeAhead()`. Leite `keydown`-Events des Panels an `keyManager.onKeydown(event)` weiter.

4. **`FocusTrap` aktivieren**
   Injiziere `FocusTrapFactory`. Wenn das Panel geöffnet wird, erzeuge mit `focusTrapFactory.create(panelElement)` einen Trap und rufe `.focusInitialElementWhenReady()` auf. Beim Schließen: `trap.destroy()`.

5. **`FocusMonitor` für den Trigger**
   Injiziere `FocusMonitor`. Beobachte den Trigger-Button: Wenn `origin === 'keyboard'`, füge eine CSS-Klasse `keyboard-focused` hinzu; bei Maus/Touch entferne sie. Denk daran, den Monitor in `ngOnDestroy` abzumelden.

## Hints

<details>
<summary>Hint 1 – FocusKeyManager Typ-Parameter</summary>

```typescript
import { FocusKeyManager, FocusableOption } from '@angular/cdk/a11y';

// MenuItemComponent muss FocusableOption implementieren:
export class MenuItemComponent implements FocusableOption {
  disabled = false;
  constructor(private host: ElementRef<HTMLElement>) {}
  focus() { this.host.nativeElement.focus(); }
  getLabel?() { return this.host.nativeElement.textContent ?? ''; }
}

// Im DropdownMenuComponent:
@ViewChildren(MenuItemComponent) items!: QueryList<MenuItemComponent>;
private keyManager!: FocusKeyManager<MenuItemComponent>;

ngAfterViewInit() {
  this.keyManager = new FocusKeyManager(this.items)
    .withWrap()
    .withTypeAhead();
}
```
</details>

<details>
<summary>Hint 2 – FocusTrap erstellen und zerstören</summary>

```typescript
import { FocusTrapFactory, FocusTrap } from '@angular/cdk/a11y';

private focusTrapFactory = inject(FocusTrapFactory);
private trap?: FocusTrap;

openMenu(panelEl: HTMLElement) {
  this.isOpen = true;
  // Warte einen Tick, bis die View gerendert ist:
  afterNextRender(() => {
    this.trap = this.focusTrapFactory.create(panelEl);
    this.trap.focusInitialElementWhenReady();
  }, { injector: this.injector });
}

closeMenu() {
  this.trap?.destroy();
  this.trap = undefined;
  this.isOpen = false;
  this.triggerEl.nativeElement.focus(); // Fokus zurück zum Trigger
}
```
</details>

<details>
<summary>Hint 3 – FocusMonitor für keyboard-only Styles</summary>

```typescript
import { FocusMonitor, FocusOrigin } from '@angular/cdk/a11y';

private focusMonitor = inject(FocusMonitor);
private triggerRef = inject(ElementRef);
private renderer = inject(Renderer2);
private destroyRef = inject(DestroyRef);

ngOnInit() {
  this.focusMonitor.monitor(this.triggerRef, false)
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe((origin: FocusOrigin) => {
      if (origin === 'keyboard') {
        this.renderer.addClass(this.triggerRef.nativeElement, 'keyboard-focused');
      } else {
        this.renderer.removeClass(this.triggerRef.nativeElement, 'keyboard-focused');
      }
    });
}
```
</details>

## Beispiellösung

```typescript
// menu-item.component.ts
import { Component, ElementRef, inject } from '@angular/core';
import { FocusableOption } from '@angular/cdk/a11y';

@Component({
  selector: 'app-menu-item',
  standalone: true,
  template: `<button class="menu-item" tabindex="-1"><ng-content /></button>`,
  styles: [`
    .menu-item { display: block; width: 100%; padding: 8px 16px; text-align: left;
                 background: none; border: none; cursor: pointer; }
    .menu-item:focus { outline: 2px solid #005fcc; outline-offset: -2px; }
  `],
})
export class MenuItemComponent implements FocusableOption {
  private host = inject(ElementRef<HTMLElement>);
  disabled = false;

  focus() {
    this.host.nativeElement.querySelector('button')?.focus();
  }
  getLabel() {
    return this.host.nativeElement.textContent?.trim() ?? '';
  }
}

// dropdown-menu.component.ts
import {
  AfterViewInit, Component, DestroyRef, ElementRef,
  Injector, OnDestroy, QueryList, ViewChild, ViewChildren, inject,
} from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { afterNextRender } from '@angular/core';
import {
  FocusKeyManager, FocusMonitor, FocusTrap,
  FocusTrapFactory, FocusOrigin,
} from '@angular/cdk/a11y';
import { MenuItemComponent } from './menu-item.component';

@Component({
  selector: 'app-dropdown-menu',
  standalone: true,
  imports: [MenuItemComponent],
  template: `
    <button #trigger class="trigger" [class.keyboard-focused]="keyboardFocused"
            (click)="toggle()" (keydown.enter)="open()" (keydown.space)="open()">
      Menü öffnen
    </button>

    @if (isOpen) {
      <div #panel class="panel" role="menu"
           (keydown)="onPanelKeydown($event)">
        @for (item of menuItems; track item) {
          <app-menu-item>{{ item }}</app-menu-item>
        }
      </div>
    }
  `,
  styles: [`
    .trigger { padding: 8px 16px; }
    .keyboard-focused { outline: 3px solid #005fcc; }
    .panel { border: 1px solid #ccc; min-width: 160px; background: white; }
  `],
})
export class DropdownMenuComponent implements AfterViewInit, OnDestroy {
  @ViewChild('trigger') triggerEl!: ElementRef<HTMLButtonElement>;
  @ViewChild('panel')   panelEl!: ElementRef<HTMLElement>;
  @ViewChildren(MenuItemComponent) items!: QueryList<MenuItemComponent>;

  menuItems = ['Profil bearbeiten', 'Einstellungen', 'Hilfe', 'Abmelden'];
  isOpen = false;
  keyboardFocused = false;

  private keyManager!: FocusKeyManager<MenuItemComponent>;
  private trap?: FocusTrap;

  private focusTrapFactory = inject(FocusTrapFactory);
  private focusMonitor     = inject(FocusMonitor);
  private injector         = inject(Injector);
  private destroyRef       = inject(DestroyRef);

  ngAfterViewInit() {
    this.focusMonitor.monitor(this.triggerEl, false)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((origin: FocusOrigin) => {
        this.keyboardFocused = origin === 'keyboard';
      });
  }

  ngOnDestroy() {
    this.trap?.destroy();
    this.focusMonitor.stopMonitoring(this.triggerEl);
  }

  toggle() { this.isOpen ? this.close() : this.open(); }

  open() {
    this.isOpen = true;
    afterNextRender(() => {
      this.keyManager = new FocusKeyManager(this.items).withWrap().withTypeAhead();
      this.trap = this.focusTrapFactory.create(this.panelEl.nativeElement);
      this.trap.focusInitialElementWhenReady();
      this.keyManager.setFirstItemActive();
    }, { injector: this.injector });
  }

  close() {
    this.trap?.destroy();
    this.trap = undefined;
    this.isOpen = false;
    this.triggerEl.nativeElement.focus();
  }

  onPanelKeydown(event: KeyboardEvent) {
    if (event.key === 'Escape') {
      this.close();
      return;
    }
    this.keyManager.onKeydown(event);
  }
}
```

## Weiterführendes

- Die offizielle CDK-Dokumentation listet alle `FocusKeyManager`-Optionen: `.withHorizontalOrientation()`, `.withHomeAndEnd()` für komplexe Grids – ideal für Custom Datepicker oder Toolbar-Widgets.
- Kombiniere `FocusMonitor` mit dem `:focus-visible`-Pseudo-Selektor als CSS-only Fallback, um die JavaScript-Logik minimal zu halten.
- Für modale Dialoge mit Angular Material ist `MatDialog` intern identisch aufgebaut – der Source-Code des CDK `DialogRef` ist ein hervorragendes Lese-Beispiel für produktionsreifen FocusTrap-Einsatz.
