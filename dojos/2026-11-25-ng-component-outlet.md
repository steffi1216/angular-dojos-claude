# Angular Dojo: NgComponentOutlet – Deklaratives dynamisches Rendering
**Datum:** 2026-11-25
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `NgComponentOutlet` Komponenten deklarativ und dynamisch im Template renderst, Inputs weitergibst und einen eigenen Injector konfigurierst – ohne `ViewContainerRef` manuell zu verwalten.

## Hintergrund & Theorie

`NgComponentOutlet` ist eine strukturelle Direktive aus `@angular/common`, die es ermöglicht, eine Komponente direkt im Template dynamisch zu rendern. Sie ist der deklarative Counterpart zur imperativen `createComponent()`-API mit `ViewContainerRef`.

```html
<ng-container *ngComponentOutlet="myComponentClass" />
```

Seit Angular 16+ unterstützt die Direktive auch das Weiterreichen von Inputs, Outputs und eines Custom Injectors – ohne boilerplate. Im modernen Angular (ab v17) kann sie auch mit der neuen `@`-Control-Flow-Syntax kombiniert werden.

**Wann NgComponentOutlet?**
- Widget-Systeme, bei denen der Komponententyp erst zur Laufzeit bekannt ist
- Dynamische Formulare oder Dashboard-Tiles
- Wenn die Logik zur Auswahl der Komponente im Controller bleiben soll, das Template aber sauber und deklarativ sein soll

**Vorteile gegenüber manueller ViewContainerRef-Nutzung:**
- Kein manuelles `createComponent()`, keine `ComponentRef`-Verwaltung
- Lifecycle wird automatisch verwaltet (destroy bei Komponentenwechsel)
- Inputs können deklarativ gebunden werden (`ngComponentOutletInputs`)

## Aufgabe

Erstelle ein dynamisches **Notification-System** mit `NgComponentOutlet`. Je nach Typ (`info`, `success`, `error`) soll eine andere Benachrichtigungs-Komponente gerendert werden, ohne `*ngIf`-Kaskaden im Template.

### Schritte

1. **Erstelle drei schlanke Benachrichtigungs-Komponenten:**
   - `InfoNotificationComponent` (zeigt ein blaues Info-Banner)
   - `SuccessNotificationComponent` (zeigt ein grünes Erfolgs-Banner)
   - `ErrorNotificationComponent` (zeigt ein rotes Fehler-Banner)
   
   Alle drei empfangen ein `message`-Input (String).

2. **Erstelle einen `NotificationService`**, der einen Signal-basierten State hält:
   ```typescript
   interface Notification {
     type: 'info' | 'success' | 'error';
     message: string;
   }
   ```

3. **Erstelle eine `NotificationHostComponent`**, die `NgComponentOutlet` nutzt:
   - Mappe `type` → Komponentenklasse in einem `Record<string, Type<unknown>>`
   - Übergib `message` via `ngComponentOutletInputs`
   - Wenn keine Notification vorliegt, zeige nichts

4. **Erstelle eine `DemoComponent`**, die über drei Buttons verschiedene Notifications auslöst, und binde `NotificationHostComponent` ein.

5. **Bonus:** Übergib einen Custom Injector via `ngComponentOutletInjector`, der einen eigenen `NotificationToken` bereitstellt (z. B. für Styling-Optionen).

### Schritte (Detail)

1. Drei Notification-Komponenten anlegen (standalone, minimal):
   ```typescript
   @Component({ selector: 'app-info-notification', standalone: true,
     template: `<div class="info">ℹ️ {{ message() }}</div>` })
   export class InfoNotificationComponent {
     message = input.required<string>();
   }
   ```
2. `NotificationService` mit `signal<Notification | null>(null)` anlegen.
3. `NotificationHostComponent` mit `NgComponentOutlet` verdrahten.
4. `DemoComponent` mit drei Buttons und Bindings fertigstellen.
5. App in `app.component.ts` verdrahten und testen.

## Hints

<details>
<summary>Hint 1 – Komponenten-Map</summary>

Lege eine Map als Klassenfeld an:

```typescript
protected readonly componentMap: Record<string, Type<unknown>> = {
  info: InfoNotificationComponent,
  success: SuccessNotificationComponent,
  error: ErrorNotificationComponent,
};
```

Dann im Template:

```html
@if (notification(); as n) {
  <ng-container
    *ngComponentOutlet="componentMap[n.type];
      inputs: { message: n.message }"
  />
}
```

</details>

<details>
<summary>Hint 2 – ngComponentOutletInputs (ab Angular 16)</summary>

Die `inputs`-Bindung entspricht dem `ngComponentOutletInputs`-Alias:

```html
<ng-container
  *ngComponentOutlet="ActiveComponent; inputs: inputsObj"
/>
```

```typescript
get inputsObj() {
  return { message: this.notification()?.message ?? '' };
}
```

Achte darauf, dass der Input-Name in der Child-Komponente exakt mit dem Schlüssel im Objekt übereinstimmt.

</details>

<details>
<summary>Hint 3 – Custom Injector (Bonus)</summary>

```typescript
import { Injector, inject } from '@angular/core';

export const NOTIFICATION_THEME = new InjectionToken<string>('NOTIFICATION_THEME');

// In der Host-Komponente:
private parentInjector = inject(Injector);

protected get customInjector(): Injector {
  return Injector.create({
    providers: [{ provide: NOTIFICATION_THEME, useValue: 'compact' }],
    parent: this.parentInjector,
  });
}
```

```html
<ng-container
  *ngComponentOutlet="ActiveComponent;
    injector: customInjector;
    inputs: inputsObj"
/>
```

</details>

## Beispiellösung

```typescript
// notification.service.ts
import { Injectable, signal } from '@angular/core';

export interface Notification {
  type: 'info' | 'success' | 'error';
  message: string;
}

@Injectable({ providedIn: 'root' })
export class NotificationService {
  readonly notification = signal<Notification | null>(null);

  show(type: Notification['type'], message: string) {
    this.notification.set({ type, message });
  }

  clear() {
    this.notification.set(null);
  }
}

// info-notification.component.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-info-notification',
  standalone: true,
  template: `<div style="background:#dbeafe;padding:8px;border-radius:4px">ℹ️ {{ message() }}</div>`,
})
export class InfoNotificationComponent {
  message = input.required<string>();
}

// success-notification.component.ts
@Component({
  selector: 'app-success-notification',
  standalone: true,
  template: `<div style="background:#dcfce7;padding:8px;border-radius:4px">✅ {{ message() }}</div>`,
})
export class SuccessNotificationComponent {
  message = input.required<string>();
}

// error-notification.component.ts
@Component({
  selector: 'app-error-notification',
  standalone: true,
  template: `<div style="background:#fee2e2;padding:8px;border-radius:4px">❌ {{ message() }}</div>`,
})
export class ErrorNotificationComponent {
  message = input.required<string>();
}

// notification-host.component.ts
import { Component, Type, inject } from '@angular/core';
import { NgComponentOutlet } from '@angular/common';
import { NotificationService } from './notification.service';

@Component({
  selector: 'app-notification-host',
  standalone: true,
  imports: [NgComponentOutlet],
  template: `
    @if (svc.notification(); as n) {
      <ng-container
        *ngComponentOutlet="componentMap[n.type]; inputs: { message: n.message }"
      />
    }
  `,
})
export class NotificationHostComponent {
  protected svc = inject(NotificationService);

  protected readonly componentMap: Record<string, Type<unknown>> = {
    info: InfoNotificationComponent,
    success: SuccessNotificationComponent,
    error: ErrorNotificationComponent,
  };
}

// demo.component.ts
import { Component, inject } from '@angular/core';
import { NotificationService } from './notification.service';
import { NotificationHostComponent } from './notification-host.component';

@Component({
  selector: 'app-demo',
  standalone: true,
  imports: [NotificationHostComponent],
  template: `
    <app-notification-host />
    <div style="margin-top:16px;display:flex;gap:8px">
      <button (click)="svc.show('info', 'Neue Version verfügbar.')">Info</button>
      <button (click)="svc.show('success', 'Speichern erfolgreich.')">Erfolg</button>
      <button (click)="svc.show('error', 'Verbindung unterbrochen.')">Fehler</button>
      <button (click)="svc.clear()">Ausblenden</button>
    </div>
  `,
})
export class DemoComponent {
  protected svc = inject(NotificationService);
}
```

## Weiterführendes
- **Offizielle Doku:** [NgComponentOutlet – Angular Docs](https://angular.dev/api/common/NgComponentOutlet) – Details zu `inputs`, `injector`, `content`-Projektion und `ngComponentOutletNgModule` (für Legacy-Module)
- **Tipp:** Ab Angular 18 kann `NgComponentOutlet` auch `outputs` deklarativ verdrahten – praktisch für Event-Bubbling ohne `ComponentRef.instance`-Zugriff.
- **Vertiefung:** Kombiniere `NgComponentOutlet` mit einem Signal-basierten Routing-Store, um ein vollständig deklaratives Widget-System ohne `RouterOutlet` zu bauen.
