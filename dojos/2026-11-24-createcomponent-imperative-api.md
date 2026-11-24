# Angular Dojo: `createComponent()` – Die imperative Component API
**Datum:** 2026-11-24
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst die low-level `createComponent()`-Funktion kennen und verstehst, wie du Angular-Komponenten imperativ – ohne Template oder `ViewContainerRef` – erstellen, konfigurieren und zerstören kannst.

## Hintergrund & Theorie

Seit Angular 14 gibt es die Funktion `createComponent()` aus `@angular/core`, die `ComponentFactory.create()` ablöst. Sie erlaubt es, Komponenten vollständig imperativ zu erstellen – nützlich für:

- Rendering in einem DOM-Knoten außerhalb des Angular-Baums (z. B. Toast-Notifications, Modals, Portals)
- Plugin-Architekturen und dynamische Dashboards
- Rendering in Web-Components oder Microfrontend-Shells

Das API sieht so aus:

```typescript
const ref = createComponent(MyComponent, {
  environmentInjector: this.environmentInjector,
  hostElement: document.getElementById('target')!,
});
```

Der zurückgegebene `ComponentRef<T>` ermöglicht:
- **Inputs setzen** via `ref.setInput('name', value)`
- **Outputs abonnieren** via `ref.instance.myOutput.subscribe(...)`
- **Change Detection auslösen** via `ref.changeDetectorRef.detectChanges()`
- **Ans ApplicationRef hängen** via `appRef.attachView(ref.hostView)` – erst dann rendert Angular die Komponente in die `tick()`-Schleife
- **Zerstören** via `ref.destroy()`

Ohne `appRef.attachView()` wird die Komponente nicht in Angular's Change-Detection-Zyklus eingebunden. Das ist ein häufiger Fehler.

## Aufgabe

Erstelle einen `NotificationService`, der Toast-Benachrichtigungen imperativ in das DOM einfügt – ohne `*ngIf`, ohne `ViewContainerRef` im Template, ohne Router.

### Schritte

1. **Erstelle die Toast-Komponente** `ToastComponent` als Standalone-Komponente mit einem `@Input() message: string` und einem `@Output() dismissed`. Sie soll einen einfachen `<div>` mit dem Text und einem Close-Button rendern.

2. **Erstelle den `NotificationService`** als `Injectable({ providedIn: 'root' })`. Injiziere `ApplicationRef` und `EnvironmentInjector`. Implementiere eine Methode `show(message: string)`:
   - Erstelle ein neues `<div>` und hänge es an `document.body`
   - Rufe `createComponent(ToastComponent, { environmentInjector, hostElement })` auf
   - Setze das Input mit `ref.setInput('message', message)`
   - Registriere `appRef.attachView(ref.hostView)` damit Angular die Komponente rendert
   - Abonniere das `dismissed`-Output: bei Emission `ref.destroy()` aufrufen und das Host-Element entfernen

3. **Teste den Service** in einer Komponente mit einem Button, der `notificationService.show('Gespeichert!')` aufruft.

4. **Stelle sicher, dass kein Memory-Leak entsteht**: Halte eine Liste aller erstellten `ComponentRef`s und zerstöre sie beim Destroy des Services mit `OnDestroy`.

## Hints

<details>
<summary>Hint 1 – createComponent Signatur</summary>

```typescript
import { createComponent, ApplicationRef, EnvironmentInjector } from '@angular/core';

const hostElement = document.createElement('div');
document.body.appendChild(hostElement);

const ref = createComponent(ToastComponent, {
  environmentInjector: this.environmentInjector,
  hostElement,
});

this.appRef.attachView(ref.hostView);
ref.setInput('message', 'Hallo Welt');
```

Ohne `attachView` rendert Angular die Komponente nicht in seinen Change-Detection-Zyklus.
</details>

<details>
<summary>Hint 2 – Output abonnieren und Cleanup</summary>

```typescript
// Output-EventEmitter direkt auf der Instanz abonnieren
const sub = ref.instance.dismissed.subscribe(() => {
  this.appRef.detachView(ref.hostView);
  ref.destroy();
  hostElement.remove();
  sub.unsubscribe();
});
```

Für Signal-based Outputs (`output()` API) nutze `outputToObservable()` aus `@angular/core/rxjs-interop`:

```typescript
import { outputToObservable } from '@angular/core/rxjs-interop';

outputToObservable(ref.instance.dismissed).subscribe(() => {
  // cleanup
});
```
</details>

## Beispiellösung

```typescript
// toast.component.ts
import { Component, input, output } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-toast',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="toast">
      <span>{{ message() }}</span>
      <button (click)="dismissed.emit()">✕</button>
    </div>
  `,
  styles: [`
    .toast {
      position: fixed;
      bottom: 1rem;
      right: 1rem;
      background: #333;
      color: #fff;
      padding: 0.75rem 1rem;
      border-radius: 4px;
      display: flex;
      gap: 1rem;
      align-items: center;
      z-index: 9999;
    }
    button { background: none; border: none; color: #fff; cursor: pointer; font-size: 1rem; }
  `]
})
export class ToastComponent {
  message = input.required<string>();
  dismissed = output<void>();
}
```

```typescript
// notification.service.ts
import {
  Injectable,
  ApplicationRef,
  EnvironmentInjector,
  ComponentRef,
  OnDestroy,
  createComponent,
} from '@angular/core';
import { outputToObservable } from '@angular/core/rxjs-interop';
import { ToastComponent } from './toast.component';

@Injectable({ providedIn: 'root' })
export class NotificationService implements OnDestroy {
  private refs: Array<{ ref: ComponentRef<ToastComponent>; host: HTMLElement }> = [];

  constructor(
    private appRef: ApplicationRef,
    private environmentInjector: EnvironmentInjector,
  ) {}

  show(message: string): void {
    const hostElement = document.createElement('div');
    document.body.appendChild(hostElement);

    const ref = createComponent(ToastComponent, {
      environmentInjector: this.environmentInjector,
      hostElement,
    });

    ref.setInput('message', message);
    this.appRef.attachView(ref.hostView);

    const entry = { ref, host: hostElement };
    this.refs.push(entry);

    const sub = outputToObservable(ref.instance.dismissed).subscribe(() => {
      this.destroyToast(entry);
      sub.unsubscribe();
    });
  }

  private destroyToast(entry: { ref: ComponentRef<ToastComponent>; host: HTMLElement }): void {
    this.appRef.detachView(entry.ref.hostView);
    entry.ref.destroy();
    entry.host.remove();
    this.refs = this.refs.filter(r => r !== entry);
  }

  ngOnDestroy(): void {
    this.refs.forEach(entry => this.destroyToast(entry));
  }
}
```

```typescript
// usage in any component
@Component({
  selector: 'app-root',
  standalone: true,
  template: `<button (click)="notify()">Benachrichtigung senden</button>`,
})
export class AppComponent {
  private notifications = inject(NotificationService);

  notify(): void {
    this.notifications.show('Daten erfolgreich gespeichert!');
  }
}
```

## Weiterführendes

- **`createComponent` vs `ViewContainerRef.createComponent`**: `ViewContainerRef` integriert die Komponente in den Angular-Baum (Hierarchie, Injektoren, Query-Systeme). `createComponent` ist bewusst losgelöst – ideal für "außerhalb des Baums"-Szenarien, aber du verlierst Content-Projection und View-Queries.
- **`reflectComponentType()`**: Companion-Funktion zu `createComponent` – liest Metadaten einer Komponente (Inputs, Outputs, Selector) zur Laufzeit ohne sie zu instanziieren. Nützlich für Plugin-Systeme.
- Offizielle Docs: [createComponent](https://angular.dev/api/core/createComponent)
