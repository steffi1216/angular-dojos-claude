# Angular Dojo: Dynamic Component Loading
**Datum:** 2026-07-04
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie man in Angular Komponenten zur Laufzeit dynamisch erzeugt und in das DOM einfügt – ohne sie im Template zu referenzieren. Du verstehst dabei die Rollen von `ViewContainerRef`, `createComponent` und `ComponentRef`.

## Hintergrund & Theorie

Dynamic Component Loading ist die Technik, Komponenten programmatisch zu instanziieren – also nicht via Template-Syntax (`<app-foo />`), sondern imperative per TypeScript. Das ist nützlich für:

- **Modals & Dialoge** – dynamisch öffnen und schließen
- **Toast/Notification-Systeme** – Nachrichten zur Laufzeit einblenden
- **Plugin-Architekturen** – Komponenten erst zur Laufzeit bestimmen
- **Conditional Rendering** komplexer Layouts

Die wichtigsten APIs:

**`ViewContainerRef`** repräsentiert einen Platzhalter im DOM, in den Angular Komponenten oder Views einfügen kann. Man holt sie sich per Dependency Injection oder via `@ViewChild`.

**`createComponent(ComponentClass)`** (seit Angular 14) instantiiert eine Komponente und gibt einen `ComponentRef<T>` zurück. Über `componentRef.instance` greift man auf die Instanz zu, über `componentRef.setInput()` setzt man Inputs typsicher.

**`ComponentRef.destroy()`** räumt die Komponente inkl. Change Detection sauber auf.

Wichtig: Dynamisch erzeugte Komponenten müssen seit Angular 14 **Standalone** sein oder sich in einem `NgModule` befinden – Standalone ist der moderne Weg.

## Aufgabe

Baue ein einfaches **In-App Notification-System**. Ein `NotificationService` soll es ermöglichen, über eine Methode `show(message, type)` dynamisch Notification-Komponenten am unteren Bildschirmrand einzublenden, die sich nach 3 Sekunden selbst entfernen.

### Schritte

1. Erstelle eine Standalone-Komponente `NotificationComponent` mit den Inputs `message: string` und `type: 'success' | 'error' | 'info'`. Sie soll sich nach 3 Sekunden via `ComponentRef.destroy()` selbst zerstören.

2. Erstelle einen `NotificationService` der:
   - `ApplicationRef` und `EnvironmentInjector` per DI erhält
   - Eine Methode `show(message: string, type: ...)` besitzt
   - Darin einen Host-`div` ans `document.body` anhängt
   - Mit `createComponent()` eine `NotificationComponent` erzeugt und die Inputs per `setInput()` setzt
   - Die Komponente in den Host-`div` hostet (via `hostElement`-Option)

3. Rufe `show()` aus einer Demo-Komponente per Button-Click auf und beobachte das Ergebnis.

4. **Bonus:** Halte die aktiven Notifications in einem Array und zeige maximal 3 gleichzeitig an.

## Hints

<details>
<summary>Hint 1 – createComponent mit eigenem Host-Element</summary>

```typescript
import { createComponent, EnvironmentInjector, ApplicationRef } from '@angular/core';

const hostElement = document.createElement('div');
document.body.appendChild(hostElement);

const ref = createComponent(NotificationComponent, {
  environmentInjector: this.injector,
  hostElement,
});

ref.setInput('message', 'Hallo!');
ref.setInput('type', 'success');

this.appRef.attachView(ref.hostView); // Change Detection aktivieren
```

`attachView` ist nötig, damit Angular die dynamische Komponente in seine Change Detection einbezieht.
</details>

<details>
<summary>Hint 2 – Selbst-Zerstörung nach Timeout</summary>

In der `NotificationComponent` kannst du `ngOnInit` nutzen:

```typescript
import { ComponentRef } from '@angular/core';

// Wird vom Service gesetzt, nachdem die Komponente erstellt wurde:
componentRef!: ComponentRef<NotificationComponent>;

ngOnInit() {
  setTimeout(() => {
    this.componentRef.destroy();
    this.hostElement?.remove(); // Host-div aufräumen
  }, 3000);
}
```

Alternativ kannst du die Cleanup-Logik vollständig im Service in einem `setTimeout` kapseln.
</details>

<details>
<summary>Hint 3 – Ohne ViewContainerRef auskommen</summary>

Für einen globalen Service ohne feste Template-Verankerung ist der `ApplicationRef`-Ansatz besser als `ViewContainerRef`, weil du keinen Root-Container im DOM brauchst. `ApplicationRef.attachView()` registriert die View bei Angular, ohne sie in eine bestehende View einzuhängen.
</details>

## Beispiellösung

```typescript
// notification.component.ts
import { Component, Input, OnInit, ComponentRef } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  standalone: true,
  imports: [CommonModule],
  selector: 'app-notification',
  template: `
    <div [class]="'notification notification--' + type">
      {{ message }}
    </div>
  `,
  styles: [`
    .notification {
      position: fixed;
      bottom: 1rem;
      right: 1rem;
      padding: 0.75rem 1.25rem;
      border-radius: 4px;
      color: white;
      font-size: 0.9rem;
      z-index: 9999;
      animation: fadeIn 0.2s ease;
    }
    .notification--success { background: #2e7d32; }
    .notification--error   { background: #c62828; }
    .notification--info    { background: #1565c0; }
    @keyframes fadeIn { from { opacity: 0; transform: translateY(8px); } }
  `],
})
export class NotificationComponent implements OnInit {
  @Input() message = '';
  @Input() type: 'success' | 'error' | 'info' = 'info';

  // Wird vom Service nach Erstellung gesetzt
  selfRef!: { destroy: () => void };

  ngOnInit() {
    setTimeout(() => this.selfRef?.destroy(), 3000);
  }
}
```

```typescript
// notification.service.ts
import {
  Injectable, ApplicationRef, EnvironmentInjector, createComponent
} from '@angular/core';
import { NotificationComponent } from './notification.component';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  constructor(
    private appRef: ApplicationRef,
    private injector: EnvironmentInjector,
  ) {}

  show(message: string, type: 'success' | 'error' | 'info' = 'info'): void {
    const hostElement = document.createElement('div');
    document.body.appendChild(hostElement);

    const ref = createComponent(NotificationComponent, {
      environmentInjector: this.injector,
      hostElement,
    });

    ref.setInput('message', message);
    ref.setInput('type', type);

    // Callback für Selbst-Zerstörung
    ref.instance.selfRef = {
      destroy: () => {
        this.appRef.detachView(ref.hostView);
        ref.destroy();
        hostElement.remove();
      },
    };

    this.appRef.attachView(ref.hostView);
  }
}
```

```typescript
// demo.component.ts
import { Component, inject } from '@angular/core';
import { NotificationService } from './notification.service';

@Component({
  standalone: true,
  selector: 'app-demo',
  template: `
    <button (click)="notify('success')">Erfolg</button>
    <button (click)="notify('error')">Fehler</button>
    <button (click)="notify('info')">Info</button>
  `,
})
export class DemoComponent {
  private notifications = inject(NotificationService);

  notify(type: 'success' | 'error' | 'info') {
    this.notifications.show(`Das ist eine ${type}-Meldung!`, type);
  }
}
```

## Weiterführendes

- **Angular Docs – Dynamic component loader:** https://angular.dev/guide/components/dynamic-components
- **Tipp:** Für komplexere Overlay-Szenarien (Backdrop, Positioning, Focus-Trap) lohnt sich ein Blick auf das **Angular CDK Overlay**-Paket (`@angular/cdk/overlay`), das genau diesen Anwendungsfall produktionsreif abdeckt – heute's Dojo zeigt die Mechanik darunter.
- **Tipp:** Mit dem neuen `@ngrx/signals`-Store lässt sich der Zustand aktiver Notifications (Array, max. 3) elegant reaktiv verwalten – eine schöne Kombination beider Techniken.
