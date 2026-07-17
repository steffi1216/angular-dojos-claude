# Angular Dojo: afterRender & afterNextRender
**Datum:** 2026-07-17
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit den Angular 17+ Hooks `afterRender` und `afterNextRender` sicher auf das DOM zugreifst, DOM-Messungen durchführst und Third-Party-Bibliotheken initialisierst – ohne auf `ngAfterViewInit` oder direkte `document`-Zugriffe angewiesen zu sein.

## Hintergrund & Theorie

Seit Angular 17 gibt es zwei neue Funktionen für Post-Render-Logik:

**`afterNextRender(callback)`** – führt den Callback genau **einmal** nach dem nächsten Render-Zyklus aus. Ideal für One-Time-Initialisierungen (z. B. Chart-Bibliothek starten, initiale DOM-Messung).

**`afterRender(callback)`** – führt den Callback nach **jedem** Render-Zyklus aus. Ideal für reaktive DOM-Beobachtungen (z. B. Höhe neu messen, wenn sich Daten ändern).

Beide Hooks akzeptieren optional ein **Phase-Argument**, das steuert, wann innerhalb des Render-Zyklus sie ausgeführt werden:
- `EarlyRead` – vor DOM-Mutations lesen (Layout-Reads)
- `Write` – DOM schreiben (Styles, Attribute setzen)
- `MixedReadWrite` (default) – kombiniert, aber weniger performant
- `Read` – nach DOM-Mutations lesen

Diese Phasen verhindern **Layout Thrashing** (abwechselndes Lesen und Schreiben des DOMs), was bei `ngAfterViewInit` leicht passiert.

Wichtig: Beide Hooks sind **nur in einem Injektionskontext** aufrufbar (Konstruktor, `inject()`-Kontext). Sie sind außerdem **SSR-sicher** – auf dem Server werden sie übersprungen.

```typescript
import { afterNextRender, afterRender, AfterRenderPhase } from '@angular/core';
```

## Aufgabe

Baue eine `ResizableCardComponent`, die:
1. Ihre eigene Breite nach jedem Render-Zyklus misst und als Signal speichert
2. Beim ersten Render eine animierte Einblend-Klasse per DOM-Manipulation setzt
3. Die gemessene Breite im Template anzeigt und bei Fenster-Resize aktuell hält

### Schritte

1. Erstelle eine neue Standalone-Komponente `ResizableCardComponent` mit einem `ElementRef`-Inject.
2. Nutze `afterNextRender` mit Phase `Write`, um beim ersten Rendern eine CSS-Klasse `is-visible` auf das Host-Element zu setzen.
3. Nutze `afterRender` mit Phase `Read`, um nach jedem Render die aktuelle Breite (`offsetWidth`) des Host-Elements zu lesen und in einem `Signal<number>` zu speichern.
4. Löse einen Re-Render durch einen Button aus, der einen internen Zählerwert (Signal) hochzählt – beobachte, wie `afterRender` die Breite neu einliest.
5. Registriere einen `resize`-EventListener im `afterNextRender`-Block und verwende `DestroyRef` für das Cleanup.

### Bonus
- Verwende `untracked()` im `afterRender`-Callback, um Schreib-Zugriffe auf Signals nicht als Abhängigkeiten zu registrieren und Endlos-Loops zu vermeiden.

## Hints

<details>
<summary>Hint 1 – Grundstruktur mit afterNextRender</summary>

```typescript
import { Component, ElementRef, inject, signal, afterNextRender, afterRender, AfterRenderPhase, DestroyRef, untracked } from '@angular/core';

@Component({
  selector: 'app-resizable-card',
  standalone: true,
  template: `...`
})
export class ResizableCardComponent {
  private el = inject(ElementRef<HTMLElement>);
  private destroyRef = inject(DestroyRef);

  constructor() {
    afterNextRender({ write: () => {
      this.el.nativeElement.classList.add('is-visible');
    }});
  }
}
```

</details>

<details>
<summary>Hint 2 – afterRender mit untracked für Signal-Writes</summary>

Signal-Writes innerhalb von `afterRender` würden Angular in eine Schleife zwingen (Render → afterRender → Signal ändert sich → Render → …). `untracked()` bricht diesen Zyklus, indem der Write als "nicht reaktiv" markiert wird:

```typescript
readonly width = signal(0);
readonly counter = signal(0);

constructor() {
  afterRender({ read: () => {
    const w = this.el.nativeElement.offsetWidth;
    untracked(() => this.width.set(w));
  }});
}
```

</details>

<details>
<summary>Hint 3 – Resize-Listener mit DestroyRef cleanup</summary>

```typescript
afterNextRender({ write: () => {
  const handler = () => {
    // Erzwinge einen Re-Render durch Signal-Änderung
    this.counter.update(v => v + 1);
  };
  window.addEventListener('resize', handler);
  this.destroyRef.onDestroy(() => window.removeEventListener('resize', handler));
}});
```

</details>

## Beispiellösung

```typescript
import {
  Component, ElementRef, inject, signal,
  afterNextRender, afterRender, DestroyRef, untracked
} from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-resizable-card',
  standalone: true,
  imports: [CommonModule],
  styles: [`
    :host {
      display: block;
      padding: 1rem;
      border: 2px solid #3f51b5;
      border-radius: 8px;
      opacity: 0;
      transition: opacity 0.4s ease;
    }
    :host.is-visible { opacity: 1; }
  `],
  template: `
    <h3>Resizable Card</h3>
    <p>Aktuelle Breite: <strong>{{ width() }}px</strong></p>
    <p>Render-Zyklus: {{ counter() }}</p>
    <button (click)="increment()">Re-Render auslösen</button>
  `
})
export class ResizableCardComponent {
  private el = inject(ElementRef<HTMLElement>);
  private destroyRef = inject(DestroyRef);

  readonly width = signal(0);
  readonly counter = signal(0);

  constructor() {
    // Einmalig: CSS-Klasse setzen + Resize-Listener registrieren
    afterNextRender({
      write: () => {
        this.el.nativeElement.classList.add('is-visible');

        const handler = () => this.counter.update(v => v + 1);
        window.addEventListener('resize', handler);
        this.destroyRef.onDestroy(() => window.removeEventListener('resize', handler));
      }
    });

    // Nach jedem Render: Breite einlesen
    afterRender({
      read: () => {
        const w = this.el.nativeElement.offsetWidth;
        // untracked verhindert reaktive Abhängigkeit → kein Endlos-Loop
        untracked(() => this.width.set(w));
      }
    });
  }

  increment(): void {
    this.counter.update(v => v + 1);
  }
}
```

```typescript
// app.component.ts – Einbinden
import { Component } from '@angular/core';
import { ResizableCardComponent } from './resizable-card.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ResizableCardComponent],
  template: `
    <div style="padding: 2rem; resize: horizontal; overflow: auto; border: 1px dashed #999; min-width: 200px;">
      <app-resizable-card />
    </div>
  `
})
export class AppComponent {}
```

## Weiterführendes

- Die offizielle Angular-Doku erklärt die **Render-Phasen** ausführlich: `EarlyRead → Write → MixedReadWrite → Read` – nutze immer die engste Phase, um Layout Thrashing zu vermeiden.
- Kombiniere `afterRender` mit dem **Resource API** (aus dem Dojo vom 08.07.) für reaktive Datenlades-Strategien, die nach dem DOM-Update ausgeführt werden.
- Für komplexe DOM-Messungen (ResizeObserver, IntersectionObserver) ist `afterNextRender` + `DestroyRef` das empfohlene Muster in modernem zoneless Angular.
- Angular-Blogpost: [Angular v17 – afterRender & afterNextRender](https://blog.angular.io/angular-v17-is-now-available-41b130bdef3f)
