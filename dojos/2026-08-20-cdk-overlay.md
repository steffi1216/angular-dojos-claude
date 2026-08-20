# Angular Dojo: CDK Overlay – Custom Positioned Overlays
**Datum:** 2026-08-20
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK `Overlay`-Service eigene positionierte Overlays (Dropdowns, Tooltips, Custom-Dialoge) programmatisch erstellst und steuerst – ohne auf fertige Material-Komponenten angewiesen zu sein.

## Hintergrund & Theorie

Das Angular CDK stellt mit `Overlay` einen Low-Level-Service bereit, der das Rendern von Inhalten in einem globalen `OverlayContainer` (außerhalb des normalen DOM-Flusses) ermöglicht. Es gibt drei Kernkonzepte:

**`OverlayRef`** – Repräsentiert ein geöffnetes Overlay. Darüber werden Inhalt angehängt, Events abonniert und das Overlay geschlossen.

**`OverlayConfig`** – Konfiguriert Position, Scroll-Strategie, Größe und Backdrop.

**`PositionStrategy`** – Bestimmt, wo das Overlay erscheint:
- `GlobalPositionStrategy` – absolute Position auf dem Bildschirm (z. B. zentriert).
- `FlexibleConnectedPositionStrategy` – relativ zu einem Ursprungselement (z. B. Button), mit automatischem Fallback bei wenig Platz.

**`ScrollStrategy`** – Was passiert beim Scrollen:
- `close` – Overlay schließt sich.
- `reposition` – Overlay folgt dem Ursprungselement.
- `block` – Scrollen wird blockiert.
- `noop` – nichts passiert.

Als Inhalt kann ein `TemplatePortal` (aus `ng-template`) oder ein `ComponentPortal` (dynamische Komponente) genutzt werden.

## Aufgabe

Baue eine wiederverwendbare `CustomDropdownDirective`, die per `[appCustomDropdown]` an beliebige Elemente gehängt werden kann. Beim Klick auf das Host-Element öffnet sich ein Template-Overlay direkt unterhalb des Elements. Ein Klick außerhalb oder Drücken von `Escape` schließt das Overlay.

### Schritte

1. **Projekt vorbereiten** – Stelle sicher, dass `@angular/cdk` installiert ist (`npm i @angular/cdk`). Importiere in deiner `app.config.ts` (oder dem Modul) keine CDK-Module – `Overlay` ist direkt als `inject(Overlay)` verfügbar.

2. **Directive skelettieren** – Erstelle `CustomDropdownDirective` als Standalone-Directive mit `selector: '[appCustomDropdown]'`. Injiziere `Overlay`, `ViewContainerRef` und `ElementRef`.

3. **OverlayRef erzeugen** – Erstelle beim ersten Klick (oder im Konstruktor) eine `OverlayConfig` mit:
   - `FlexibleConnectedPositionStrategy` zum Host-Element, bevorzugte Position: unten-links → Fallback: oben-links.
   - `ScrollStrategy`: `close` (schließt bei Scroll).
   - Optional: `hasBackdrop: true` mit `backdropClass: 'cdk-overlay-transparent-backdrop'`.

4. **TemplatePortal anhängen** – Die Directive soll einen `@Input() dropdownTemplate: TemplateRef<unknown>` entgegen nehmen. Erstelle daraus einen `TemplatePortal` und hänge ihn an den `OverlayRef` an.

5. **Schließ-Logik implementieren** – Subscribte auf `overlayRef.backdropClick()` und auf ein globales `keydown`-Event (`Escape`), um das Overlay zu schließen. Räume beim Schließen mit `overlayRef.detach()` auf und deregistriere die Subscriptions. Nutze `DestroyRef` für automatisches Cleanup.

6. **Template im Parent definieren** – Nutze die Directive in einer Komponente:
   ```html
   <button [appCustomDropdown]="myMenu">Öffnen</button>

   <ng-template #myMenu>
     <div class="dropdown-panel">
       <a href="#">Option 1</a>
       <a href="#">Option 2</a>
     </div>
   </ng-template>
   ```

7. **Styling** – Gib `.dropdown-panel` etwas CSS (weißer Background, Schatten, `z-index`).

## Hints

<details>
<summary>Hint 1 – OverlayConfig und PositionStrategy</summary>

```typescript
const positionStrategy = this.overlay
  .position()
  .flexibleConnectedTo(this.elementRef)
  .withPositions([
    { originX: 'start', originY: 'bottom', overlayX: 'start', overlayY: 'top' },
    { originX: 'start', originY: 'top',    overlayX: 'start', overlayY: 'bottom' },
  ]);

const config = new OverlayConfig({
  positionStrategy,
  scrollStrategy: this.overlay.scrollStrategies.close(),
  hasBackdrop: true,
  backdropClass: 'cdk-overlay-transparent-backdrop',
});

this.overlayRef = this.overlay.create(config);
```
</details>

<details>
<summary>Hint 2 – TemplatePortal anhängen und Schließen</summary>

```typescript
// Öffnen
const portal = new TemplatePortal(this.dropdownTemplate, this.viewContainerRef);
this.overlayRef.attach(portal);

// Schließen
const sub = merge(
  this.overlayRef.backdropClick(),
  fromEvent<KeyboardEvent>(document, 'keydown').pipe(
    filter(e => e.key === 'Escape')
  )
).subscribe(() => {
  this.overlayRef?.detach();
  sub.unsubscribe();
});
```
</details>

## Beispiellösung

```typescript
import {
  Directive, ElementRef, HostListener, inject, input, OnDestroy, ViewContainerRef
} from '@angular/core';
import { Overlay, OverlayConfig, OverlayRef } from '@angular/cdk/overlay';
import { TemplatePortal } from '@angular/cdk/portal';
import { fromEvent, merge, Subscription } from 'rxjs';
import { filter } from 'rxjs/operators';
import { TemplateRef } from '@angular/core';

@Directive({
  selector: '[appCustomDropdown]',
  standalone: true,
})
export class CustomDropdownDirective implements OnDestroy {
  readonly dropdownTemplate = input.required<TemplateRef<unknown>>({
    alias: 'appCustomDropdown',
  });

  private readonly overlay = inject(Overlay);
  private readonly elementRef = inject(ElementRef);
  private readonly viewContainerRef = inject(ViewContainerRef);

  private overlayRef: OverlayRef | null = null;
  private closeSub: Subscription | null = null;

  @HostListener('click')
  toggle(): void {
    if (this.overlayRef?.hasAttached()) {
      this.close();
    } else {
      this.open();
    }
  }

  private open(): void {
    if (!this.overlayRef) {
      const positionStrategy = this.overlay
        .position()
        .flexibleConnectedTo(this.elementRef)
        .withPositions([
          { originX: 'start', originY: 'bottom', overlayX: 'start', overlayY: 'top' },
          { originX: 'start', originY: 'top',    overlayX: 'start', overlayY: 'bottom' },
        ]);

      this.overlayRef = this.overlay.create(new OverlayConfig({
        positionStrategy,
        scrollStrategy: this.overlay.scrollStrategies.close(),
        hasBackdrop: true,
        backdropClass: 'cdk-overlay-transparent-backdrop',
      }));
    }

    const portal = new TemplatePortal(this.dropdownTemplate(), this.viewContainerRef);
    this.overlayRef.attach(portal);

    this.closeSub = merge(
      this.overlayRef.backdropClick(),
      fromEvent<KeyboardEvent>(document, 'keydown').pipe(filter(e => e.key === 'Escape'))
    ).subscribe(() => this.close());
  }

  private close(): void {
    this.overlayRef?.detach();
    this.closeSub?.unsubscribe();
    this.closeSub = null;
  }

  ngOnDestroy(): void {
    this.overlayRef?.dispose();
  }
}
```

```html
<!-- In deiner Komponente -->
<button [appCustomDropdown]="myMenu">Menü öffnen</button>

<ng-template #myMenu>
  <div class="dropdown-panel">
    <a href="#">Profil</a>
    <a href="#">Einstellungen</a>
    <a href="#">Abmelden</a>
  </div>
</ng-template>
```

```css
.dropdown-panel {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  padding: 8px 0;
  min-width: 160px;
  display: flex;
  flex-direction: column;
}

.dropdown-panel a {
  padding: 10px 16px;
  text-decoration: none;
  color: #333;
}

.dropdown-panel a:hover {
  background: #f5f5f5;
}
```

## Weiterführendes

- **`ComponentPortal` statt `TemplatePortal`**: Rendere eine eigenständige Komponente ins Overlay – nützlich für komplexe Popovers mit eigenem Zustand.
- **`GlobalPositionStrategy`**: Für modale Dialoge (`centerHorizontally().centerVertically()`). Kombiniere mit einem opaken Backdrop (`'cdk-overlay-dark-backdrop'`).
- **`reposition`-ScrollStrategy**: Lass das Overlay dem Anker-Element folgen, wenn die Seite gescrollt wird – ideal für lange Listen mit eingebetteten Aktionsmenüs.
- Offizielle Docs: [Angular CDK Overlay](https://material.angular.io/cdk/overlay/overview)
