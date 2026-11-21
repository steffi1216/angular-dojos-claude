# Angular Dojo: Zone.js Konfiguration & selektives Monkey-Patching

**Datum:** 2026-11-21
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie du Zone.js gezielt konfigurierst, um unnötiges Monkey-Patching zu deaktivieren und damit die Performance deiner Angular-App messbar zu verbessern — als pragmatischen Zwischenschritt auf dem Weg zu vollständig zoneless Angular.

## Hintergrund & Theorie

Zone.js überwacht asynchrone Operationen, indem es native Browser-APIs patcht: `setTimeout`, `setInterval`, `requestAnimationFrame`, `fetch`, `Promise`, Event-Listener und mehr. Bei jedem dieser Calls prüft Angular, ob eine Change Detection notwendig ist.

Das ist bequem, kostet aber Performance: Jeder gepatchte API-Aufruf trägt Overhead, selbst wenn Angular dabei gar nichts neu rendern muss. In großen Apps mit vielen Timern, WebSocket-Events oder Animationen summiert sich das.

Zone.js bietet deshalb **Flags**, die vor dem Import der Library gesetzt werden können, um einzelne Patches zu deaktivieren. Diese Flags werden in `src/zone-flags.ts` definiert und müssen **vor** dem Zone.js-Import in `polyfills.ts` geladen werden.

Typische Kandidaten zum Deaktivieren:
- `__Zone_disable_requestAnimationFrame` — für eigene RAF-Loops (z. B. Canvas, Three.js)
- `__Zone_disable_on_property` — deaktiviert Patching von `onclick`, `onerror` etc.
- `__Zone_disable_timer` — für Performance-kritische Timer-Nutzung außerhalb von Angular
- `__zone_symbol__UNPATCHED_EVENTS` — Liste von Events, die nicht gepacht werden sollen

Kombiniert mit `NgZone.runOutsideAngular()` lässt sich so sehr gezielt steuern, was Change Detection auslöst — ohne vollständig auf zoneless umsteigen zu müssen.

## Aufgabe

Erstelle eine Angular-Anwendung mit einem Canvas-Element, das kontinuierlich eine Animation rendert. Ohne Optimierung triggert jedes `requestAnimationFrame`-Callback eine Angular Change Detection. Deine Aufgabe ist es, Zone.js so zu konfigurieren, dass `requestAnimationFrame` und `scroll`-Events nicht mehr gepacht werden, und die Animation stattdessen explizit außerhalb der Zone zu betreiben.

### Schritte

1. **Zone-Flags Datei anlegen:** Erstelle `src/zone-flags.ts` und deaktiviere `requestAnimationFrame`-Patching sowie `scroll`-Events.

2. **Polyfills anpassen:** Stelle sicher, dass `zone-flags.ts` in `angular.json` **vor** `zone.js` geladen wird (im `polyfills`-Array).

3. **Canvas-Komponente erstellen:** Schreibe eine `CanvasAnimationComponent`, die über `requestAnimationFrame` ein rotierendes Quadrat zeichnet.

4. **NgZone integrieren:** Starte die Animation-Loop explizit mit `ngZone.runOutsideAngular()`. Reagiere auf einen Button-Click innerhalb der Zone, um einen Zähler im Template hochzuzählen — und beobachte, dass der Canvas weiterläuft, ohne Change Detection zu stören.

5. **Verifikation:** Öffne Angular DevTools, aktiviere den Profiler und vergleiche die Change-Detection-Häufigkeit mit und ohne die Zone-Flags.

## Hints

<details>
<summary>Hint 1 – zone-flags.ts Syntax</summary>

Die Flags sind globale Variablen, die du als `(window as any).__Zone_disable_xxx = true` oder auf `globalThis` setzen kannst. Zone.js liest sie beim ersten Import aus.

```typescript
// src/zone-flags.ts
(window as any).__Zone_disable_requestAnimationFrame = true;
(window as any).__zone_symbol__UNPATCHED_EVENTS = ['scroll', 'mousemove'];
```

In `angular.json` unter `projects > architect > build > options > polyfills`:
```json
"polyfills": ["src/zone-flags.ts", "zone.js"]
```

Die Reihenfolge ist entscheidend — `zone-flags.ts` **muss** vor `zone.js` stehen.

</details>

<details>
<summary>Hint 2 – Animation-Loop mit runOutsideAngular</summary>

Da `requestAnimationFrame` jetzt nicht mehr gepacht ist, löst es keine automatische Change Detection aus. Du musst es trotzdem `runOutsideAngular` starten, um sicherzustellen, dass kein anderer Mechanismus (z. B. Promise-Chaining) die Zone wieder einzieht:

```typescript
constructor(private ngZone: NgZone) {}

ngAfterViewInit() {
  this.ngZone.runOutsideAngular(() => {
    const loop = () => {
      this.draw();
      this.rafId = requestAnimationFrame(loop);
    };
    this.rafId = requestAnimationFrame(loop);
  });
}

ngOnDestroy() {
  cancelAnimationFrame(this.rafId);
}
```

Für explizite UI-Updates (z. B. den Klick-Zähler) rufst du `this.ngZone.run(() => this.counter++)` auf.

</details>

## Beispiellösung

```typescript
// src/zone-flags.ts
(window as any).__Zone_disable_requestAnimationFrame = true;
(window as any).__zone_symbol__UNPATCHED_EVENTS = ['scroll', 'mousemove'];
```

```typescript
// canvas-animation.component.ts
import {
  AfterViewInit,
  Component,
  ElementRef,
  NgZone,
  OnDestroy,
  ViewChild,
  signal,
} from '@angular/core';

@Component({
  selector: 'app-canvas-animation',
  standalone: true,
  template: `
    <canvas #canvas width="300" height="300" style="border:1px solid #ccc"></canvas>
    <div>
      <button (click)="increment()">Klick-Zähler: {{ counter() }}</button>
    </div>
    <p>Rotationswinkel (kein CD-Trigger): {{ angleDisplay }}</p>
  `,
})
export class CanvasAnimationComponent implements AfterViewInit, OnDestroy {
  @ViewChild('canvas') canvasRef!: ElementRef<HTMLCanvasElement>;

  counter = signal(0);
  angleDisplay = 0; // bewusst keine Signal — kein Re-render nötig
  private angle = 0;
  private rafId = 0;

  constructor(private ngZone: NgZone) {}

  ngAfterViewInit(): void {
    const ctx = this.canvasRef.nativeElement.getContext('2d')!;

    // Läuft vollständig außerhalb der Zone — kein Change-Detection-Overhead
    this.ngZone.runOutsideAngular(() => {
      const loop = () => {
        this.angle = (this.angle + 1) % 360;
        this.angleDisplay = this.angle; // wird erst beim nächsten CD sichtbar
        this.draw(ctx, this.angle);
        this.rafId = requestAnimationFrame(loop);
      };
      this.rafId = requestAnimationFrame(loop);
    });
  }

  increment(): void {
    // Click-Handler läuft bereits in der Zone dank (click)-Binding
    this.counter.update((c) => c + 1);
  }

  private draw(ctx: CanvasRenderingContext2D, angle: number): void {
    ctx.clearRect(0, 0, 300, 300);
    ctx.save();
    ctx.translate(150, 150);
    ctx.rotate((angle * Math.PI) / 180);
    ctx.fillStyle = '#6200ea';
    ctx.fillRect(-40, -40, 80, 80);
    ctx.restore();
  }

  ngOnDestroy(): void {
    cancelAnimationFrame(this.rafId);
  }
}
```

```json
// Ausschnitt angular.json
{
  "polyfills": ["src/zone-flags.ts", "zone.js"]
}
```

## Weiterführendes

- **Nächster Schritt Richtung Zoneless:** Ersetze `NgZone` durch `ChangeDetectorRef.detectChanges()` oder nutze Signals in der Komponente, um explizite Change-Detection-Triggers zu eliminieren — und teste dann `provideExperimentalZonelessChangeDetection()` als direkten Nachfolger dieses Musters.
- **Offizieller Zone.js-Konfigurations-Guide:** [github.com/angular/angular/blob/main/packages/zone.js/STANDARD-APIS.md](https://github.com/angular/angular/blob/main/packages/zone.js/STANDARD-APIS.md) — vollständige Liste aller verfügbaren Flags.
- **Tipp:** Nutze in der Produktion `source-map-explorer` nach dem Build, um zu prüfen, ob Zone.js-Patches noch im Bundle landen, obwohl Flags gesetzt sind.
