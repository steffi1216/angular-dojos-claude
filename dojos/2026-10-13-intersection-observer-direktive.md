# Angular Dojo: IntersectionObserver-Direktive
**Datum:** 2026-10-13
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du baust eine wiederverwendbare Angular-Direktive, die den nativen `IntersectionObserver` kapselt und per Output-Signal oder Observable meldet, wann ein Element in den Viewport eintritt oder ihn verlässt.

## Hintergrund & Theorie
Der `IntersectionObserver` ist eine native Browser-API, die asynchron Sichtbarkeitsänderungen eines Elements relativ zu einem Root-Container beobachtet. Im Gegensatz zu Scroll-Events ist er performant (kein Hauptthread-Blocking) und ideal für Lazy Loading, Infinite Scroll oder Scroll-triggered Animationen.

In Angular kapselt man ihn am besten in einer Direktive:
- Die Direktive injiziert `ElementRef` und `DestroyRef` für Lifecycle-Management
- Sie erstellt den Observer im `afterNextRender`-Hook (oder `ngAfterViewInit` als Alternative)
- Events werden über einen `OutputEmitter` (neue Output API) oder ein `Subject` nach außen gegeben
- Mit `takeUntilDestroyed()` und `DestroyRef.onDestroy()` wird der Observer automatisch beim Destroy bereinigt

Wichtige Optionen des `IntersectionObserver`:
- `threshold`: Schwellwert (0–1) oder Array von Schwellwerten, ab wann der Callback feuert
- `rootMargin`: CSS-Margin-ähnliche Erweiterung des Viewport (z.B. `"100px 0px"` für frühzeitiges Laden)
- `root`: Anderer Scroll-Container als Viewport

## Aufgabe
Erstelle eine Direktive `appInView`, die erkennt, wann ein Element in den Viewport eintritt. Verwende sie in einer Demo-Komponente, um beim Scrollen lazy geladene Karten sichtbar zu machen (CSS-Animation) und um Infinite-Scroll zu triggern.

### Schritte

1. **Direktive erstellen** – Erstelle `InViewDirective` als Standalone-Direktive mit dem Selektor `[appInView]`.

2. **IntersectionObserver kapseln** – Injiziere `ElementRef` und `DestroyRef`. Erstelle den Observer in `afterNextRender`. Bereinige ihn in `DestroyRef.onDestroy()`.

3. **Outputs definieren** – Definiere mit der neuen `output()`-API zwei Outputs:
   - `inView` – feuert mit `true`/`false`, wenn sich der Sichtbarkeitsstatus ändert
   - `inViewEntry` – gibt das vollständige `IntersectionObserverEntry`-Objekt aus

4. **Inputs für Konfiguration** – Nutze `input()` für `threshold` (Default: `0.1`) und `rootMargin` (Default: `'0px'`).

5. **Demo-Komponente** – Erstelle eine Liste mit 20 Karten. Jede Karte hat `[appInView]` und wird mit einer CSS-Klasse `visible` animiert, sobald `inView` auf `true` wechselt. Am Ende der Liste platziere ein Sentinel-Element, das Infinite Scroll triggert.

## Hints

<details>
<summary>Hint 1 – Observer-Erstellung und Cleanup</summary>

```typescript
export class InViewDirective {
  private el = inject(ElementRef);
  private destroyRef = inject(DestroyRef);

  ngAfterViewInit() {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          this.inView.emit(entry.isIntersecting);
          this.inViewEntry.emit(entry);
        });
      },
      { threshold: this.threshold(), rootMargin: this.rootMargin() }
    );

    observer.observe(this.el.nativeElement);
    this.destroyRef.onDestroy(() => observer.disconnect());
  }
}
```

`afterNextRender` ist die modernere Alternative zu `ngAfterViewInit`, wenn du nur einmalig nach dem ersten Render brauchst.

</details>

<details>
<summary>Hint 2 – Input und Output mit neuer API</summary>

```typescript
import { input, output } from '@angular/core';

threshold = input<number | number[]>(0.1);
rootMargin = input<string>('0px');

inView = output<boolean>();
inViewEntry = output<IntersectionObserverEntry>();
```

In der Template-Seite:
```html
<div
  appInView
  [threshold]="0.2"
  (inView)="onVisibilityChange($event, card)"
>
  {{ card.title }}
</div>
```

</details>

<details>
<summary>Hint 3 – CSS-Animation für Fade-in</summary>

```scss
.card {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.4s ease, transform 0.4s ease;

  &.visible {
    opacity: 1;
    transform: translateY(0);
  }
}
```

In der Komponente:
```typescript
onVisibilityChange(isVisible: boolean, card: Card) {
  if (isVisible) {
    card.visible = true; // einmal sichtbar, bleibt sichtbar
  }
}
```

```html
<div class="card" [class.visible]="card.visible" [appInView] (inView)="onVisibilityChange($event, card)">
```

</details>

## Beispiellösung

```typescript
// in-view.directive.ts
import {
  afterNextRender,
  DestroyRef,
  Directive,
  ElementRef,
  inject,
  input,
  output,
} from '@angular/core';

@Directive({
  selector: '[appInView]',
  standalone: true,
})
export class InViewDirective {
  private el = inject(ElementRef);
  private destroyRef = inject(DestroyRef);

  threshold = input<number | number[]>(0.1);
  rootMargin = input<string>('0px');

  inView = output<boolean>();
  inViewEntry = output<IntersectionObserverEntry>();

  constructor() {
    afterNextRender(() => {
      const observer = new IntersectionObserver(
        (entries) => {
          entries.forEach((entry) => {
            this.inView.emit(entry.isIntersecting);
            this.inViewEntry.emit(entry);
          });
        },
        {
          threshold: this.threshold(),
          rootMargin: this.rootMargin(),
        }
      );

      observer.observe(this.el.nativeElement);
      this.destroyRef.onDestroy(() => observer.disconnect());
    });
  }
}

// scroll-demo.component.ts
import { Component, signal } from '@angular/core';
import { InViewDirective } from './in-view.directive';

interface DemoCard {
  id: number;
  title: string;
  visible: boolean;
}

@Component({
  selector: 'app-scroll-demo',
  standalone: true,
  imports: [InViewDirective],
  template: `
    <div class="scroll-container">
      @for (card of cards(); track card.id) {
        <div
          class="card"
          [class.visible]="card.visible"
          appInView
          [threshold]="0.15"
          rootMargin="50px"
          (inView)="onCardVisible($event, card)"
        >
          <h3>{{ card.title }}</h3>
          <p>Karte #{{ card.id }}</p>
        </div>
      }

      <!-- Sentinel für Infinite Scroll -->
      <div
        class="sentinel"
        appInView
        rootMargin="200px"
        (inView)="onSentinelVisible($event)"
      ></div>

      @if (loading()) {
        <div class="loading">Lade mehr...</div>
      }
    </div>
  `,
  styles: [`
    .scroll-container { padding: 1rem; }

    .card {
      padding: 1rem;
      margin-bottom: 1rem;
      border: 1px solid #ddd;
      border-radius: 8px;
      opacity: 0;
      transform: translateY(20px);
      transition: opacity 0.4s ease, transform 0.4s ease;
    }

    .card.visible {
      opacity: 1;
      transform: translateY(0);
    }

    .sentinel { height: 1px; }
    .loading { text-align: center; padding: 1rem; }
  `],
})
export class ScrollDemoComponent {
  cards = signal<DemoCard[]>(this.generateCards(1, 10));
  loading = signal(false);
  private nextId = 11;

  onCardVisible(isVisible: boolean, card: DemoCard) {
    if (isVisible && !card.visible) {
      card.visible = true;
    }
  }

  onSentinelVisible(isVisible: boolean) {
    if (isVisible && !this.loading()) {
      this.loadMore();
    }
  }

  private loadMore() {
    this.loading.set(true);
    // Simulierter API-Call
    setTimeout(() => {
      const newCards = this.generateCards(this.nextId, this.nextId + 9);
      this.cards.update((current) => [...current, ...newCards]);
      this.nextId += 10;
      this.loading.set(false);
    }, 800);
  }

  private generateCards(from: number, to: number): DemoCard[] {
    return Array.from({ length: to - from + 1 }, (_, i) => ({
      id: from + i,
      title: `Dynamische Karte ${from + i}`,
      visible: false,
    }));
  }
}
```

## Weiterführendes
- **Performance-Tipp**: Setze `{ once: true }` als Observer-Option (noch kein W3C-Standard, aber in manchen Polyfills verfügbar) oder trenne den Observer manuell nach der ersten Intersection, um Memory-Overhead zu reduzieren.
- **Signal-Variante**: Statt Outputs kann die Direktive ein `Signal<boolean>` über `toSignal(fromEvent(...))` exponieren – kombiniere dafür `fromEvent` mit einem `Subject`, das der Observer füttert.
- **Angular CDK `ObserversModule`**: Für `MutationObserver`-ähnliche Funktionalität bietet das CDK `CdkObserveContent`. Der `IntersectionObserver` fehlt im CDK, lässt sich aber ähnlich kapseln.
- **Ressource**: [MDN – IntersectionObserver API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)
