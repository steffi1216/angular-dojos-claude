# Angular Dojo: CDK Portal
**Datum:** 2026-08-05
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK Portal-System (`PortalOutlet`, `ComponentPortal`, `TemplatePortal`) Inhalte dynamisch in beliebige DOM-Positionen einblendest – ohne Eltern-Kind-Hierarchie im Template.

## Hintergrund & Theorie
Das CDK Portal-System löst ein klassisches Problem: Manchmal soll eine Komponente Inhalt an einer Stelle rendern, die weit von ihr selbst entfernt ist – z. B. ein Toast in einem `<app-toasts>`-Container, ein Dialog-Body im `<body>`, oder ein Tab-Panel an einer anderen Stelle im DOM.

Das System besteht aus drei Bausteinen:

- **`Portal`** – der Inhalt, der gerendert werden soll. Es gibt zwei Arten:
  - `ComponentPortal<T>` – rendert eine Komponente
  - `TemplatePortal` – rendert ein `TemplateRef`
- **`PortalOutlet`** – der Zielpunkt, an dem der Portal-Inhalt eingeblendet wird. Die einfachste Variante ist die `CdkPortalOutlet`-Direktive (`[cdkPortalOutlet]`).
- **`DomPortalOutlet`** – ermöglicht das Rendern in einen beliebigen DOM-Knoten außerhalb des Angular Component Tree (z. B. direkt in `document.body`).

Das Portal-System ist die Grundlage für CDK Overlay, Tooltips und Dialoge. Es selbst zu nutzen gibt dir maximale Kontrolle bei minimaler Infrastruktur.

## Aufgabe
Baue eine `NotificationBoardComponent`, die Benachrichtigungen über ein Portal-System anzeigt. Ein Button in einer tief verschachtelten Kindkomponente soll eine Notification in einem zentralen `<notification-board>` einblenden – ohne `@Output`-Kaskade oder globalen Service-Overhead.

### Schritte

1. **Installiere CDK** (falls noch nicht vorhanden):
   ```bash
   ng add @angular/cdk
   ```

2. **Erstelle die `NotificationComponent`** – eine einfache Standalone-Komponente, die eine Nachricht als `input()` empfängt und anzeigt (z. B. als farbige Karte).

3. **Erstelle den `NotificationPortalService`** – ein injectable Service, der:
   - einen `PortalOutlet` registriert/hält
   - eine Methode `show(message: string)` anbietet, die einen `ComponentPortal<NotificationComponent>` erstellt, ihn in den Outlet attached und die `message` per `componentRef.setInput()` setzt.

4. **Erstelle die `NotificationBoardComponent`** – bindet `CdkPortalOutlet` in ihrem Template ein und registriert sich beim `NotificationPortalService` via `afterNextRender()`.

5. **Rufe `service.show('...')` in einer tief verschachtelten `TriggerComponent` auf** – ohne Props drilling, nur per `inject(NotificationPortalService)`.

6. Teste, dass die Notification tatsächlich im Board erscheint, obwohl der Trigger irgendwo anders im Component Tree liegt.

## Hints

<details>
<summary>Hint 1 – PortalOutlet registrieren</summary>

```typescript
// notification-board.component.ts
export class NotificationBoardComponent {
  private service = inject(NotificationPortalService);
  readonly outletRef = viewChild.required(CdkPortalOutlet);

  constructor() {
    afterNextRender(() => {
      this.service.registerOutlet(this.outletRef());
    });
  }
}
```

```html
<!-- notification-board.component.html -->
<section class="board">
  <ng-template cdkPortalOutlet />
</section>
```

</details>

<details>
<summary>Hint 2 – ComponentPortal attachen und Input setzen</summary>

```typescript
// notification-portal.service.ts
@Injectable({ providedIn: 'root' })
export class NotificationPortalService {
  private outlet: PortalOutlet | null = null;
  private injector = inject(Injector);

  registerOutlet(outlet: PortalOutlet) {
    this.outlet = outlet;
  }

  show(message: string) {
    if (!this.outlet) return;
    if (this.outlet.hasAttached()) {
      this.outlet.detach();
    }
    const portal = new ComponentPortal(NotificationComponent, null, this.injector);
    const ref = this.outlet.attach(portal);
    ref.setInput('message', message);
    // Auto-dismiss nach 3 Sekunden
    setTimeout(() => this.outlet?.detach(), 3000);
  }
}
```

</details>

<details>
<summary>Hint 3 – Mehrere Notifications mit DomPortalOutlet</summary>

Wenn du mehrere Notifications gleichzeitig anzeigen willst, kannst du statt `CdkPortalOutlet` (das immer nur einen Inhalt hält) mehrere `DomPortalOutlet`-Instanzen dynamisch anlegen:

```typescript
const container = document.createElement('div');
document.querySelector('.board')!.appendChild(container);
const outlet = new DomPortalOutlet(container, this.cfr, this.appRef, this.injector);
const ref = outlet.attach(new ComponentPortal(NotificationComponent));
```

Du brauchst dafür `ComponentFactoryResolver` (deprecated, aber noch nutzbar) oder alternativ `createComponent()` mit `EnvironmentInjector`.

</details>

## Beispiellösung

```typescript
// notification.component.ts
import { Component, input } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-notification',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="notification">
      <span>{{ message() }}</span>
    </div>
  `,
  styles: [`
    .notification {
      background: #4caf50;
      color: white;
      padding: 12px 20px;
      border-radius: 6px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.2);
      font-weight: 500;
    }
  `],
})
export class NotificationComponent {
  message = input.required<string>();
}
```

```typescript
// notification-portal.service.ts
import { Injectable, inject, Injector } from '@angular/core';
import { ComponentPortal, PortalOutlet } from '@angular/cdk/portal';
import { NotificationComponent } from './notification.component';

@Injectable({ providedIn: 'root' })
export class NotificationPortalService {
  private injector = inject(Injector);
  private outlet: PortalOutlet | null = null;
  private dismissTimer: ReturnType<typeof setTimeout> | null = null;

  registerOutlet(outlet: PortalOutlet) {
    this.outlet = outlet;
  }

  show(message: string, durationMs = 3000) {
    if (!this.outlet) {
      console.warn('NotificationPortalService: kein Outlet registriert');
      return;
    }
    if (this.dismissTimer) clearTimeout(this.dismissTimer);
    if (this.outlet.hasAttached()) this.outlet.detach();

    const portal = new ComponentPortal(NotificationComponent, null, this.injector);
    const ref = this.outlet.attach(portal);
    ref.setInput('message', message);

    this.dismissTimer = setTimeout(() => {
      this.outlet?.detach();
      this.dismissTimer = null;
    }, durationMs);
  }
}
```

```typescript
// notification-board.component.ts
import { Component, inject, afterNextRender, viewChild } from '@angular/core';
import { CdkPortalOutlet } from '@angular/cdk/portal';
import { NotificationPortalService } from './notification-portal.service';

@Component({
  selector: 'app-notification-board',
  standalone: true,
  imports: [CdkPortalOutlet],
  template: `
    <div class="board-wrapper">
      <ng-template cdkPortalOutlet />
    </div>
  `,
  styles: [`
    .board-wrapper {
      position: fixed;
      bottom: 24px;
      right: 24px;
      z-index: 1000;
    }
  `],
})
export class NotificationBoardComponent {
  private service = inject(NotificationPortalService);
  private outlet = viewChild.required(CdkPortalOutlet);

  constructor() {
    afterNextRender(() => {
      this.service.registerOutlet(this.outlet());
    });
  }
}
```

```typescript
// trigger.component.ts – tief verschachtelte Kindkomponente
import { Component, inject } from '@angular/core';
import { NotificationPortalService } from './notification-portal.service';

@Component({
  selector: 'app-trigger',
  standalone: true,
  template: `
    <button (click)="notify()">Speichern</button>
  `,
})
export class TriggerComponent {
  private notifications = inject(NotificationPortalService);

  notify() {
    this.notifications.show('Erfolgreich gespeichert!');
  }
}
```

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { NotificationBoardComponent } from './notification-board.component';
import { TriggerComponent } from './trigger.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [NotificationBoardComponent, TriggerComponent],
  template: `
    <h1>CDK Portal Demo</h1>
    <!-- Trigger liegt irgendwo im Baum -->
    <div style="padding: 40px">
      <app-trigger />
    </div>
    <!-- Board ist unabhängig davon, wo es im Template sitzt -->
    <app-notification-board />
  `,
})
export class AppComponent {}
```

## Weiterführendes
- **CDK Overlay** baut auf dem Portal-System auf und bietet zusätzlich Positionierung, Backdrop und Scroll-Strategien – der nächste logische Schritt nach diesem Dojo.
- Für **mehrere gleichzeitige Notifications** (Stack) eignet sich `DomPortalOutlet` mit dynamisch erzeugten Container-Elementen, oder ein Signal-Array kombiniert mit `@for` und `CdkPortalOutlet` pro Eintrag.
- Offizielle Doku: [https://material.angular.io/cdk/portal/overview](https://material.angular.io/cdk/portal/overview)
