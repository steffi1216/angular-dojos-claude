# Angular Dojo: ApplicationRef und Anwendungslebensyklus
**Datum:** 2026-11-17
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie `ApplicationRef` dir direkten Zugriff auf den Anwendungslebensyklus gibt – von manuellem Change-Detection-Triggering bis hin zur Erkennung von Stabilitätsmomenten, die für SSR, Service Workers und End-to-End-Tests entscheidend sind.

## Hintergrund & Theorie
`ApplicationRef` ist der interne Angular-Dienst, der die laufende Anwendung repräsentiert. Er ist über `inject(ApplicationRef)` oder Konstruktor-DI verfügbar und bietet folgende Kernfähigkeiten:

**`tick()`** löst einen manuellen Change-Detection-Zyklus für alle registrierten Component-Trees aus. Nützlich, wenn Änderungen außerhalb der Angular-Zone stattfinden (z. B. in nativen DOM-Event-Handlern oder third-party Libraries) und `NgZone` deaktiviert ist.

**`isStable`** ist ein `Observable<boolean>`, das `true` emittiert, sobald keine ausstehenden asynchronen Operationen (Makro- und Mikrotasks) mehr vorhanden sind. Angular Universal (SSR) nutzt dies intern, um den richtigen Serialisierungszeitpunkt zu bestimmen.

**`whenStable()`** gibt ein `Promise<void>` zurück, das resolvet, wenn die App stabil wird. Praktisch in Tests und beim manuellen Serialisieren.

**`components`** ist ein Array aller `ComponentRef`-Instanzen, die als Root-Komponenten bootstrapped wurden.

**`APP_BOOTSTRAP_LISTENER`** ist ein `InjectionToken`, mit dem du Code ausführen kannst, sobald eine Root-Komponente bootstrapped wird – ideal für Initialisierungslogik, die auf das DOM angewiesen ist.

Seit Angular 18 ist `ApplicationRef` auch bei zoneless-Apps die zentrale Stelle, um Change Detection manuell zu steuern.

## Aufgabe
Baue einen **Stabilitäts-Monitor-Service**, der:

1. Die App-Stabilität beobachtet und in einem Signal exponiert.
2. Eine Methode bereitstellt, um auf Stabilität zu warten (z. B. vor Screenshot oder Serialisierung).
3. `APP_BOOTSTRAP_LISTENER` nutzt, um eine initiale Aktion (Logging) beim Bootstrap auszulösen.
4. In einer Demo-Komponente den aktuellen Stabilitätsstatus anzeigt und einen Button bereitstellt, der einen manuellen `tick()` auslöst (simuliert für eine zoneless-App).

### Schritte
1. Erstelle `StabilityMonitorService` mit `inject(ApplicationRef)`.
2. Konvertiere `applicationRef.isStable` mit `toSignal()` in ein Signal (Startwert `false`).
3. Implementiere `waitForStability(): Promise<void>` mittels `applicationRef.whenStable()`.
4. Registriere einen `APP_BOOTSTRAP_LISTENER` Provider in `app.config.ts`, der beim Bootstrap eine Meldung loggt und `waitForStability()` aufruft.
5. Erstelle eine `StabilityDemoComponent`, die den Signal-Wert anzeigt und einen Button für manuellen `tick()` hat.

## Hints

<details>
<summary>Hint 1 – isStable als Signal</summary>

`ApplicationRef.isStable` ist ein `Observable<boolean>`. Nutze `toSignal()` aus `@angular/core/rxjs-interop`:

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

readonly isStable = toSignal(this.appRef.isStable, { initialValue: false });
```

</details>

<details>
<summary>Hint 2 – APP_BOOTSTRAP_LISTENER registrieren</summary>

`APP_BOOTSTRAP_LISTENER` erwartet eine Factory-Funktion, die eine Callback-Funktion zurückgibt. Der Callback erhält die gerade bootstrappte `ComponentRef` als Argument:

```typescript
import { APP_BOOTSTRAP_LISTENER, ComponentRef } from '@angular/core';

{
  provide: APP_BOOTSTRAP_LISTENER,
  multi: true,
  useFactory: (monitor: StabilityMonitorService) => (compRef: ComponentRef<unknown>) => {
    console.log('[Bootstrap] Komponente registriert:', compRef.componentType.name);
    monitor.waitForStability().then(() => console.log('[Bootstrap] App ist stabil'));
  },
  deps: [StabilityMonitorService],
}
```

</details>

<details>
<summary>Hint 3 – Manuelles tick() in zoneless-Apps</summary>

In einer zoneless-App (konfiguriert mit `provideExperimentalZonelessChangeDetection()`) löst Angular Change Detection nur noch bei Signal-Mutations oder explizitem `tick()` aus. Um den Effekt sichtbar zu machen, ändere einen Wert außerhalb eines Signals und rufe danach `appRef.tick()` auf:

```typescript
triggerManualTick() {
  // Simuliert eine Änderung außerhalb der Zone
  this.externalValue = Date.now();
  this.appRef.tick();
}
```

</details>

## Beispiellösung

```typescript
// stability-monitor.service.ts
import { inject, Injectable } from '@angular/core';
import { ApplicationRef } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';

@Injectable({ providedIn: 'root' })
export class StabilityMonitorService {
  private readonly appRef = inject(ApplicationRef);

  readonly isStable = toSignal(this.appRef.isStable, { initialValue: false });

  waitForStability(): Promise<void> {
    return this.appRef.whenStable();
  }

  tick(): void {
    this.appRef.tick();
  }

  getBootstrappedComponents() {
    return this.appRef.components.map(ref => ref.componentType.name);
  }
}
```

```typescript
// app.config.ts
import { ApplicationConfig, APP_BOOTSTRAP_LISTENER, ComponentRef } from '@angular/core';
import { provideRouter } from '@angular/router';
import { StabilityMonitorService } from './stability-monitor.service';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter([]),
    {
      provide: APP_BOOTSTRAP_LISTENER,
      multi: true,
      useFactory: (monitor: StabilityMonitorService) =>
        (compRef: ComponentRef<unknown>) => {
          const name = compRef.componentType.name;
          console.log(`[Bootstrap] "${name}" wurde registriert`);
          monitor.waitForStability().then(() => {
            console.log(`[Bootstrap] App stabil. Root-Komponenten:`, monitor.getBootstrappedComponents());
          });
        },
      deps: [StabilityMonitorService],
    },
  ],
};
```

```typescript
// stability-demo.component.ts
import { Component, inject, signal } from '@angular/core';
import { StabilityMonitorService } from './stability-monitor.service';

@Component({
  selector: 'app-stability-demo',
  standalone: true,
  template: `
    <h2>Anwendungsstabilität</h2>
    <p>Status: <strong>{{ monitor.isStable() ? '✅ Stabil' : '⏳ Ausstehende Tasks' }}</strong></p>
    <p>Root-Komponenten: {{ components().join(', ') }}</p>
    <button (click)="triggerTick()">Manuelles tick()</button>
    <button (click)="checkStability()">Auf Stabilität warten</button>
    <p *ngIf="lastStabilityMs()">Stabil nach: {{ lastStabilityMs() }}ms</p>
  `,
})
export class StabilityDemoComponent {
  readonly monitor = inject(StabilityMonitorService);
  readonly components = signal<string[]>([]);
  readonly lastStabilityMs = signal<number | null>(null);

  constructor() {
    this.components.set(this.monitor.getBootstrappedComponents());
  }

  triggerTick() {
    this.monitor.tick();
  }

  async checkStability() {
    const start = performance.now();
    await this.monitor.waitForStability();
    this.lastStabilityMs.set(Math.round(performance.now() - start));
  }
}
```

## Weiterführendes
- **Angular SSR & Stabilität:** `ApplicationRef.whenStable()` ist der Mechanismus hinter `renderApplication()` – wirf einen Blick in `@angular/platform-server` Quellcode, um zu sehen, wie das Serialisierungsfenster bestimmt wird.
- **`PendingTasks` Service (Angular 18+):** Für feingranulare Kontrolle über die App-Stabilität (bereits im Dojo `2026-11-13` behandelt) kombiniere `PendingTasks` mit `ApplicationRef.isStable` für eigene Stabilitäts-Gates.
- **Zoneless Change Detection:** Mit `provideExperimentalZonelessChangeDetection()` wird `tick()` zur einzigen Möglichkeit, CD von außen zu triggern – verstehe den Unterschied zu `ChangeDetectorRef.detectChanges()` (komponentenlokal) vs. `ApplicationRef.tick()` (global).
