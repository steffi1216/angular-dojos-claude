# Angular Dojo: NgZone – runOutsideAngular und run()
**Datum:** 2026-08-31
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `NgZone.runOutsideAngular()` und `NgZone.run()` die Change-Detection-Auslösung präzise steuern kannst, um Performance-Engpässe durch unnötige Render-Zyklen zu vermeiden.

## Hintergrund & Theorie

Angular nutzt standardmäßig Zone.js, das asynchrone Operationen (setTimeout, Promises, EventListener, XHR) abfängt und nach jeder davon automatisch Change Detection auslöst. Das ist komfortabel, kann aber zum Problem werden, wenn viele hochfrequente Events (z. B. `mousemove`, `scroll`, `WebSocket-Nachrichten`, `requestAnimationFrame`) Change Detection im ganzen Komponentenbaum triggern – selbst wenn sich gar nichts Relevantes verändert hat.

`NgZone` bietet zwei zentrale Methoden:

- **`runOutsideAngular(fn)`**: Führt `fn` außerhalb der Angular-Zone aus. Zone.js patcht keine Events oder Timers innerhalb dieses Callbacks → Change Detection wird **nicht** ausgelöst.
- **`run(fn)`**: Führt `fn` innerhalb der Angular-Zone aus. Nützlich, um nach einer Outside-Operation wieder in die Zone zurückzukehren und Change Detection zu triggern (z. B. nach dem letzten Event).

Typische Anwendungsfälle:
- `requestAnimationFrame`-Loops für Canvas/Animationen
- Hochfrequente DOM-Events (`mousemove`, `scroll`, `resize`)
- WebSocket- oder Worker-Messages, die zunächst aggregiert werden
- Third-Party-Bibliotheken, die intern viele Timer verwenden

Das Gegenstück dazu – Zonen komplett zu entfernen – ist bereits im Dojo zu *zoneless Angular* behandelt. `runOutsideAngular` ist der pragmatische Mittelweg in bestehenden Apps.

## Aufgabe

Erstelle eine `MouseTrackerComponent`, die die aktuelle Mausposition auf dem Bildschirm anzeigt. Die Komponente soll:

1. Den `mousemove`-EventListener **außerhalb der Angular-Zone** registrieren, damit nicht bei jeder Mausbewegung Change Detection läuft.
2. Den internen Zähler (wie viele Moves seit letztem Render) hochzählen – ebenfalls außerhalb der Zone.
3. Die Anzeige nur alle **100 ms** in der Angular-Zone aktualisieren (Throttling-Effekt via `setInterval`).
4. Beim Zerstören der Komponente alle Listener sauber aufräumen.

### Schritte

1. Erzeuge eine neue Standalone-Komponente `MouseTrackerComponent` und injiziere `NgZone`.
2. Registriere im `ngOnInit` einen `mousemove`-Listener auf `window` **via `this.zone.runOutsideAngular()`**.
3. Speichere `clientX` / `clientY` in privaten Feldern (kein Signal, keine Change Detection nötig).
4. Starte ebenfalls außerhalb der Zone ein `setInterval` mit 100 ms. Im Callback: rufe `this.zone.run()` auf und aktualisiere ein sichtbares `Signal` mit den aktuellen Koordinaten.
5. Räume in `ngOnDestroy` den EventListener und das Interval auf.
6. Zeige in der Template-View die Koordinaten und einen Zähler an, wie viele `mousemove`-Events seit dem letzten Render eingegangen sind.

## Hints

<details>
<summary>Hint 1 – Grundstruktur</summary>

```typescript
import { Component, OnInit, OnDestroy, NgZone, signal } from '@angular/core';

@Component({
  standalone: true,
  selector: 'app-mouse-tracker',
  template: `...`
})
export class MouseTrackerComponent implements OnInit, OnDestroy {
  private zone = inject(NgZone);
  position = signal({ x: 0, y: 0 });
  eventsSinceLastRender = signal(0);

  private _x = 0;
  private _y = 0;
  private _pendingCount = 0;
  private _intervalId: ReturnType<typeof setInterval> | null = null;
  private _onMouseMove!: (e: MouseEvent) => void;
}
```

</details>

<details>
<summary>Hint 2 – runOutsideAngular und run()</summary>

```typescript
ngOnInit() {
  this.zone.runOutsideAngular(() => {
    this._onMouseMove = (e: MouseEvent) => {
      this._x = e.clientX;
      this._y = e.clientY;
      this._pendingCount++;
    };
    window.addEventListener('mousemove', this._onMouseMove);

    this._intervalId = setInterval(() => {
      const count = this._pendingCount;
      this._pendingCount = 0;
      if (count > 0) {
        this.zone.run(() => {
          this.position.set({ x: this._x, y: this._y });
          this.eventsSinceLastRender.set(count);
        });
      }
    }, 100);
  });
}

ngOnDestroy() {
  window.removeEventListener('mousemove', this._onMouseMove);
  if (this._intervalId !== null) clearInterval(this._intervalId);
}
```

</details>

## Beispiellösung

```typescript
import {
  Component, OnInit, OnDestroy, NgZone, inject, signal, ChangeDetectionStrategy
} from '@angular/core';

@Component({
  standalone: true,
  selector: 'app-mouse-tracker',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="tracker">
      <p>Position: {{ position().x }} / {{ position().y }}</p>
      <p>Events seit letztem Render: {{ eventsSinceLastRender() }}</p>
    </div>
  `,
  styles: [`
    .tracker { font-family: monospace; padding: 1rem; border: 1px solid #ccc; }
  `]
})
export class MouseTrackerComponent implements OnInit, OnDestroy {
  private zone = inject(NgZone);

  position = signal({ x: 0, y: 0 });
  eventsSinceLastRender = signal(0);

  private _x = 0;
  private _y = 0;
  private _pendingCount = 0;
  private _intervalId: ReturnType<typeof setInterval> | null = null;
  private _onMouseMove!: (e: MouseEvent) => void;

  ngOnInit() {
    this.zone.runOutsideAngular(() => {
      this._onMouseMove = (e: MouseEvent) => {
        this._x = e.clientX;
        this._y = e.clientY;
        this._pendingCount++;
      };
      window.addEventListener('mousemove', this._onMouseMove);

      this._intervalId = setInterval(() => {
        const count = this._pendingCount;
        this._pendingCount = 0;
        if (count > 0) {
          // Nur bei tatsächlicher Bewegung zurück in die Zone
          this.zone.run(() => {
            this.position.set({ x: this._x, y: this._y });
            this.eventsSinceLastRender.set(count);
          });
        }
      }, 100);
    });
  }

  ngOnDestroy() {
    window.removeEventListener('mousemove', this._onMouseMove);
    if (this._intervalId !== null) clearInterval(this._intervalId);
  }
}
```

**Verbesserung mit Angular DevTools überprüfen**: Öffne Angular DevTools → „Profiler" und vergleiche die Change-Detection-Häufigkeit mit und ohne `runOutsideAngular`. Der Unterschied ist bei schneller Mausbewegung deutlich sichtbar.

## Weiterführendes

- Kombiniere dieses Muster mit `ChangeDetectionStrategy.OnPush` für maximale Kontrolle – OnPush verhindert ungewollte CD-Zyklen von oben, `runOutsideAngular` verhindert sie von Events unten.
- Lies die offizielle Angular-Doku zu [NgZone](https://angular.dev/api/core/NgZone) sowie den Guide zu [Zone-Pollution](https://angular.dev/guide/change-detection/zone-pollution).
- Für Greenfield-Projekte ist `provideExperimentalZonelessChangeDetection()` + Signals die modernere Alternative – das Dojo vom 2026-07-03 zeigt den Einstieg.
