# Angular Dojo: afterRender & afterNextRender
**Datum:** 2026-07-19
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst, wann und wie du `afterRender` und `afterNextRender` einsetzt, um sicher auf das DOM zuzugreifen – ohne Zone.js-Probleme und ohne auf `ngAfterViewInit` oder manuelle `setTimeout`-Hacks angewiesen zu sein.

## Hintergrund & Theorie

Angular 16 führte zwei neue Lifecycle-Hooks ein, die speziell für **DOM-Interaktionen nach dem Rendering** gedacht sind:

**`afterNextRender(callback)`** – Führt den Callback genau **einmal** aus, nach dem nächsten Render-Zyklus. Ideal für Initialisierungen: Drittbibliotheken einbinden, initiale DOM-Messungen, Canvas-Setup.

**`afterRender(callback)`** – Führt den Callback nach **jedem** Render-Zyklus aus, solange die Komponente existiert. Gut für fortlaufende DOM-Reaktionen (z. B. Scroll-Positionen aktualisieren, Chart-Daten neu zeichnen).

Beide Hooks arbeiten mit **Phasen** (`EarlyRead`, `Read`, `MixedReadWrite`, `Write`), um Read/Write-Batching zu ermöglichen und Layout-Thrashing zu verhindern:

```typescript
afterRender({
  read: () => { /* DOM messen */ },
  write: () => { /* DOM schreiben */ },
});
```

**Wichtige Eigenschaften:**
- Laufen nur im Browser (kein SSR-Problem)
- Müssen im **Injection Context** aufgerufen werden (Konstruktor, `inject()`-Call, oder mit `runInInjectionContext`)
- Ersetzen viele `ngAfterViewInit` + `ChangeDetectorRef.detectChanges()`-Muster
- Kompatibel mit zoneless Angular

## Aufgabe

Baue eine `ResizableBoxComponent`, die ihre eigene Breite in Pixeln als Signal trackt und bei jeder Größenänderung des Fensters aktualisiert. Visualisiere den Wert im Template. Nutze `afterRender` mit der `read`-Phase, um DOM-Abmessungen zu lesen, und `afterNextRender` für die einmalige Initialisierung eines einfachen Canvas-Zeichnungseffekts.

### Schritte

1. **Komponente anlegen** – Erstelle `ResizableBoxComponent` als Standalone-Komponente mit einem `<div #box>` und einem `<canvas #canvas>`.

2. **Signal für Breite** – Deklariere ein `width = signal(0)` in der Komponente.

3. **`afterNextRender` – Canvas initialisieren** – Zeichne beim ersten Render einen Rahmen in den Canvas (nutze `ElementRef` oder `viewChild`).

4. **`afterRender` mit Phasen** – Lies in der `read`-Phase die aktuelle Breite des `<div>` via `getBoundingClientRect()` und schreibe sie in das Signal. Sorge dafür, dass das bei jedem Render passiert.

5. **Resize-Trigger** – Füge einen `HostListener` für `window:resize` hinzu, der `ChangeDetectionRef.markForCheck()` (oder bei Signals: einfach eine dummy Signal-Mutation) auslöst, damit Angular einen neuen Render-Zyklus startet.

6. **Template** – Zeige `{{ width() }}px` an und style den `<div>` mit `width: 60%; border: 2px solid steelblue;`.

## Hints

<details>
<summary>Hint 1 – afterRender mit read/write Phasen</summary>

```typescript
afterRender({
  read: () => {
    const rect = this.boxRef.nativeElement.getBoundingClientRect();
    this._measuredWidth = rect.width; // Zwischenspeicher, kein Signal-Write in read!
  },
  write: () => {
    this.width.set(Math.round(this._measuredWidth));
  },
});
```

Schreibe **nie** in Signals oder das DOM in der `read`-Phase – das erzeugt ExpressionChangedAfterItHasBeenChecked-Fehler oder Layout-Thrashing.
</details>

<details>
<summary>Hint 2 – afterNextRender für Canvas-Initialisierung</summary>

```typescript
afterNextRender(() => {
  const canvas = this.canvasRef.nativeElement as HTMLCanvasElement;
  const ctx = canvas.getContext('2d')!;
  canvas.width = canvas.offsetWidth;
  canvas.height = 40;
  ctx.strokeStyle = 'steelblue';
  ctx.lineWidth = 2;
  ctx.strokeRect(1, 1, canvas.width - 2, 38);
  ctx.fillStyle = 'steelblue';
  ctx.font = '14px sans-serif';
  ctx.fillText('Canvas initialisiert ✓', 8, 26);
});
```
</details>

<details>
<summary>Hint 3 – Resize-Zyklus anstoßen</summary>

Mit Signals reicht es, ein reaktives State-Signal zu touchen, damit Angular neu rendert:

```typescript
private _tick = signal(0);

@HostListener('window:resize')
onResize() {
  this._tick.update(v => v + 1); // triggert neuen Render-Zyklus
}
```

`afterRender` wird dann automatisch erneut ausgeführt.
</details>

## Beispiellösung

```typescript
import {
  Component, signal, viewChild, ElementRef,
  ChangeDetectionStrategy, HostListener, afterRender, afterNextRender,
} from '@angular/core';

@Component({
  selector: 'app-resizable-box',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div #box style="width: 60%; border: 2px solid steelblue; padding: 16px; box-sizing: border-box;">
      <p>Aktuelle Breite: <strong>{{ width() }}px</strong></p>
      <canvas #canvas style="width:100%; display:block; margin-top:8px;"></canvas>
    </div>
  `,
})
export class ResizableBoxComponent {
  protected readonly boxRef = viewChild.required<ElementRef<HTMLDivElement>>('box');
  protected readonly canvasRef = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');

  readonly width = signal(0);

  private _tick = signal(0);
  private _measuredWidth = 0;

  constructor() {
    afterNextRender(() => {
      const canvas = this.canvasRef().nativeElement;
      const ctx = canvas.getContext('2d')!;
      canvas.width = canvas.offsetWidth;
      canvas.height = 40;
      ctx.strokeStyle = 'steelblue';
      ctx.lineWidth = 2;
      ctx.strokeRect(1, 1, canvas.width - 2, 38);
      ctx.fillStyle = 'steelblue';
      ctx.font = '14px sans-serif';
      ctx.fillText('Canvas initialisiert ✓', 8, 26);
    });

    afterRender({
      read: () => {
        this._measuredWidth = this.boxRef().nativeElement.getBoundingClientRect().width;
        this._tick(); // Abhängigkeit registrieren, damit bei resize neu läuft
      },
      write: () => {
        this.width.set(Math.round(this._measuredWidth));
      },
    });
  }

  @HostListener('window:resize')
  onResize() {
    this._tick.update(v => v + 1);
  }
}
```

**Bootstrapping in `main.ts`:**

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { ResizableBoxComponent } from './app/resizable-box.component';

bootstrapApplication(ResizableBoxComponent, {
  providers: [provideExperimentalZonelessChangeDetection()],
});
```

## Weiterführendes

- **Offizielle Docs:** [angular.dev/api/core/afterRender](https://angular.dev/api/core/afterRender) – vollständige Phasen-Referenz (`EarlyRead`, `Read`, `MixedReadWrite`, `Write`)
- **Best Practice:** Bevorzuge `afterRender` mit expliziten Phasen gegenüber `MixedReadWrite`, um Layout-Thrashing in komplexen Komponenten zu vermeiden.
- **Fortgeschritten:** Kombiniere `afterRender` mit `ResizeObserver` (statt `window:resize`), um elementspezifische Größenänderungen reaktiv und performant zu tracken – kein globaler Event-Listener nötig.
- `afterRender` / `afterNextRender` funktionieren auch in **Direktiven** und **Services mit `inject()`**, nicht nur in Komponenten.
