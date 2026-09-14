# Angular Dojo: RxJS Schedulers
**Datum:** 2026-09-14
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie RxJS Schedulers den *Zeitpunkt* und den *Ausführungskontext* von Observable-Emissionen steuern, und setzt `observeOn`, `subscribeOn` sowie `animationFrameScheduler` gezielt in einer Angular-Komponente ein.

## Hintergrund & Theorie

RxJS-Operatoren wie `interval` oder `timer` arbeiten standardmäßig mit dem `asyncScheduler` (basiert auf `setInterval`/`setTimeout`). Ein **Scheduler** kapselt drei Dinge: *wann* wird Arbeit ausgeführt, in welcher *Reihenfolge*, und auf welcher *Queue*.

Die vier eingebauten Schedulers:

| Scheduler               | Mechanismus               | Typischer Einsatz                          |
|-------------------------|---------------------------|--------------------------------------------|
| `asyncScheduler`        | `setTimeout` / `setInterval` | Standard-async, Timer, `delay()`        |
| `asapScheduler`         | Microtask (`Promise.resolve`) | Schnellstmöglich nach aktuellem Task    |
| `queueScheduler`        | Synchron, FIFO            | Recursive-safe Operationen                 |
| `animationFrameScheduler` | `requestAnimationFrame` | Smooth Animations, 60fps-Updates          |

**`observeOn(scheduler)`** bestimmt, auf welchem Scheduler die *Werte* eines Streams verarbeitet werden (downstream).  
**`subscribeOn(scheduler)`** bestimmt, auf welchem Scheduler die *Subscription* selbst startet (upstream-Initialisierung).

Der wichtigste Unterschied: `observeOn` beeinflusst alle Operatoren *nach* ihm in der Kette, `subscribeOn` wirkt sich auf den Startkontext aus – egal wo es in der Kette steht.

## Aufgabe

Erstelle eine Angular Standalone-Komponente `SchedulerDemoComponent`, die drei unabhängige Demos in Tabs zeigt:

### Schritte

1. **Setup**: Erstelle `scheduler-demo.component.ts` mit drei Signals: `asyncLog`, `asapLog`, `animFrameLog` (jeweils `string[]`).

2. **Demo 1 – Scheduler-Reihenfolge**: Drücke einen Button, der synchron drei `scheduled()`-Observables feuert – eines mit `queueScheduler`, eines mit `asapScheduler`, eines mit `asyncScheduler`. Logge die Reihenfolge der Emissionen in `asyncLog`. Erwarte: `queueScheduler` → `asapScheduler` → `asyncScheduler`.

3. **Demo 2 – `animationFrameScheduler`**: Erstelle mit `interval(0, animationFrameScheduler)` einen Stream, der ~60-mal pro Sekunde einen Zähler hochzählt. Zeige den aktuellen Wert als Fortschrittsbalken (0–100, dann reset). Nutze `takeUntilDestroyed()` für Cleanup.

4. **Demo 3 – `observeOn` vs. `subscribeOn`**: Erstelle einen `of(1, 2, 3)`-Stream, der mit `subscribeOn(asyncScheduler)` verzögert startet, aber mit `observeOn(queueScheduler)` synchron Werte verarbeitet. Logge vor und nach dem Subscribe die aktuelle Zeit, um den asynchronen Start sichtbar zu machen.

### Schritte (Code)

1. Importiere `{ asyncScheduler, asapScheduler, queueScheduler, animationFrameScheduler, scheduled, interval, of }` aus `rxjs`
2. Importiere `{ observeOn, subscribeOn, tap, take, map }` aus `rxjs/operators`
3. Nutze `takeUntilDestroyed()` für den animationFrame-Stream

## Hints

<details>
<summary>Hint 1 – scheduled() Syntax</summary>

`scheduled()` ist der generische Weg, einen Scheduler für eine beliebige Quelle zu setzen:

```typescript
scheduled([42], asyncScheduler).subscribe(v => console.log('async:', v));
scheduled([42], queueScheduler).subscribe(v => console.log('queue:', v));
```

Beachte: `queueScheduler` ist *synchron* – der `console.log` wird sofort ausgeführt, bevor die nächste Zeile Code läuft.
</details>

<details>
<summary>Hint 2 – animationFrameScheduler Zähler</summary>

```typescript
const counter = signal(0);
interval(0, animationFrameScheduler).pipe(
  map(i => i % 101),
  takeUntilDestroyed()
).subscribe(v => counter.set(v));
```

`interval(0, animationFrameScheduler)` emittiert jeden Frame (ca. 16ms bei 60Hz). Das `takeUntilDestroyed()` muss im Injection Context aufgerufen werden (also im Constructor oder als Field-Initializer).
</details>

<details>
<summary>Hint 3 – observeOn vs. subscribeOn Reihenfolge</summary>

```typescript
console.log('vor subscribe');
of(1, 2, 3).pipe(
  subscribeOn(asyncScheduler),   // Subscription startet async
  observeOn(queueScheduler),     // Werte werden sync verarbeitet
  tap(v => console.log('Wert:', v))
).subscribe(() => {});
console.log('nach subscribe');   // wird VOR den Werten geloggt
```

`subscribeOn(asyncScheduler)` bedeutet: der `of()`-Stream startet seinen subscribe-Aufruf nicht sofort, sondern in einem `setTimeout`. Daher erscheint "nach subscribe" im Log vor "Wert: 1".
</details>

## Beispiellösung

```typescript
import { Component, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import {
  asyncScheduler, asapScheduler, queueScheduler,
  animationFrameScheduler, scheduled, interval, of
} from 'rxjs';
import { observeOn, subscribeOn, tap, map } from 'rxjs/operators';

@Component({
  selector: 'app-scheduler-demo',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>RxJS Schedulers Demo</h2>

    <section>
      <h3>Demo 1: Ausführungsreihenfolge</h3>
      <button (click)="runOrderDemo()">Reihenfolge testen</button>
      <pre>{{ orderLog().join('\n') }}</pre>
    </section>

    <section>
      <h3>Demo 2: animationFrameScheduler</h3>
      <div class="progress-bar">
        <div class="bar" [style.width.%]="animFrame()"></div>
      </div>
      <span>Frame: {{ animFrame() }}</span>
    </section>

    <section>
      <h3>Demo 3: observeOn vs. subscribeOn</h3>
      <button (click)="runSchedulerDemo()">Demo starten</button>
      <pre>{{ schedulerLog().join('\n') }}</pre>
    </section>
  `,
  styles: [`
    .progress-bar { width: 300px; height: 20px; background: #eee; border-radius: 4px; }
    .bar { height: 100%; background: steelblue; border-radius: 4px; transition: none; }
  `]
})
export class SchedulerDemoComponent {
  orderLog = signal<string[]>([]);
  animFrame = signal(0);
  schedulerLog = signal<string[]>([]);

  constructor() {
    // animationFrameScheduler im Injection Context initialisieren
    interval(0, animationFrameScheduler).pipe(
      map(i => i % 101),
      takeUntilDestroyed()
    ).subscribe(v => this.animFrame.set(v));
  }

  runOrderDemo(): void {
    const log: string[] = [];

    // queueScheduler: synchron
    scheduled(['queue'], queueScheduler)
      .subscribe(v => log.push(`1. ${v}Scheduler (synchron)`));

    // asapScheduler: Microtask
    scheduled(['asap'], asapScheduler)
      .subscribe(v => log.push(`2. ${v}Scheduler (Microtask)`));

    // asyncScheduler: setTimeout
    scheduled(['async'], asyncScheduler)
      .subscribe(v => log.push(`3. ${v}Scheduler (setTimeout)`));

    log.push('--- synchron nach scheduled() Aufrufen ---');

    // asap und async kommen nach dem aktuellen synchronen Block
    // Daher: queue → (sync) → asap → async
    setTimeout(() => this.orderLog.set([...log]), 50);
  }

  runSchedulerDemo(): void {
    const log: string[] = [];

    log.push(`[${Date.now() % 10000}ms] Vor subscribe`);

    of(1, 2, 3).pipe(
      subscribeOn(asyncScheduler),    // startet async (nach setTimeout)
      observeOn(queueScheduler),      // verarbeitet Werte synchron
      tap(v => log.push(`[${Date.now() % 10000}ms] Wert: ${v}`))
    ).subscribe({
      complete: () => {
        log.push(`[${Date.now() % 10000}ms] Complete`);
        this.schedulerLog.set([...log]);
      }
    });

    log.push(`[${Date.now() % 10000}ms] Nach subscribe (synchron)`);
    // Werte kommen noch nicht – subscribeOn(async) verzögert den Start
  }
}
```

## Weiterführendes

- **Virtueller Time-Scheduler für Tests**: `TestScheduler` aus `rxjs/testing` erlaubt das deterministische Testen von zeitbasiertem Code via Marble-Strings – kombinierbar mit dem Dojo `2026-08-10-rxjs-marble-testing.md`.
- **Custom Scheduler**: Implementiere `SchedulerLike` mit eigenem `schedule()`-Mechanismus, z.B. für Web Workers (Arbeit off-thread schedulen).
- **Offizieller Guide**: [RxJS – Scheduler](https://rxjs.dev/guide/scheduler) erklärt die internen Queues und Ausführungsmodelle detailliert.
