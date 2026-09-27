# Angular Dojo: RxJS Buffer & Window Operators
**Datum:** 2026-09-27
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst den Unterschied zwischen Buffer- und Window-Operatoren in RxJS und kannst sie gezielt einsetzen, um Datenströme zu bündeln, zu drosseln und effizienter zu verarbeiten.

## Hintergrund & Theorie

Buffer- und Window-Operatoren sind mächtige RxJS-Werkzeuge, um einen Datenstrom in Gruppen aufzuteilen. Der Unterschied:

- **Buffer-Operatoren** sammeln Werte und emittieren sie als **Array** erst wenn eine Bedingung erfüllt ist (z. B. eine bestimmte Zeit vergangen ist oder eine Anzahl erreicht wurde). Der ursprüngliche Stream wird dabei unterbrochen.
- **Window-Operatoren** arbeiten ähnlich, emittieren aber **Observable-Streams** statt Arrays – jedes „Fenster" ist selbst ein Observable. Das ermöglicht flexiblere Weiterverarbeitung, z. B. mit `mergeMap`.

Die wichtigsten Operatoren:

| Operator | Beschreibung |
|---|---|
| `bufferTime(ms)` | Sammelt Werte für `ms` Millisekunden, gibt Array |
| `bufferCount(n)` | Sammelt `n` Werte, gibt Array |
| `bufferWhen(() => obs)` | Puffert bis ein Trigger-Observable emittiert |
| `windowTime(ms)` | Öffnet Fenster-Observables alle `ms` ms |
| `windowCount(n)` | Öffnet neue Fenster alle `n` Werte |

**Typische Use Cases in Angular:**
- Ereignisse bündeln (z. B. mehrere Klicks innerhalb von 500 ms = Doppelklick-Erkennung)
- Batch-Requests: mehrere API-Aufrufe zu einem zusammenfassen
- Real-Time-Daten: Websocket-Events gruppieren, bevor sie die UI updaten
- Undo-Verlauf: Aktionen zeitlich bündeln

## Aufgabe

Baue einen **Batch-Logger Service** in Angular, der User-Aktionen (z. B. Button-Klicks) puffert und erst nach 2 Sekunden (oder sobald 5 Aktionen aufgelaufen sind) als Batch an einen (simulierten) API-Endpoint sendet.

Zusätzlich: Baue eine **Doppelklick-Erkennung** mit `bufferTime` oder `bufferCount`.

### Schritte

1. Erstelle einen `ActionLoggerService` mit einem `Subject<string>` für eingehende Aktionen.
2. Nutze `bufferTime(2000)` um Aktionen alle 2 Sekunden zu bündeln.
3. Filtere leere Buffer heraus (wenn in 2 Sekunden keine Aktion kam, soll kein leerer Array verarbeitet werden).
4. Simuliere einen Batch-API-Call mit `tap()` (z. B. `console.log('Batch senden:', batch)`).
5. Erstelle in einer Komponente zwei Buttons: „Aktion ausführen" (mehrfach klickbar) und einen „Doppelklick erkannt!"-Button.
6. Implementiere die Doppelklick-Erkennung: Nutze `bufferTime(300)` auf einem `fromEvent`-Stream; wenn der Buffer genau 2 Einträge enthält, gilt es als Doppelklick.

```typescript
// Ziel-API des Service
@Injectable({ providedIn: 'root' })
export class ActionLoggerService {
  private action$ = new Subject<string>();

  logAction(name: string): void { /* ... */ }

  getBatches$(): Observable<string[]> { /* mit bufferTime(2000) */ }
}
```

## Hints

<details>
<summary>Hint 1 – bufferTime mit Filter</summary>

```typescript
this.action$.pipe(
  bufferTime(2000),
  filter(batch => batch.length > 0)
)
```

Ohne das `filter` würde alle 2 Sekunden ein leeres Array emittiert – auch wenn nichts passiert ist.
</details>

<details>
<summary>Hint 2 – Doppelklick mit bufferTime</summary>

```typescript
fromEvent(button, 'click').pipe(
  bufferTime(300),
  filter(clicks => clicks.length === 2)
).subscribe(() => console.log('Doppelklick erkannt!'));
```

Alternativ mit `bufferCount(2, 1)` (Sliding Window): Das zweite Argument ist der `startBufferEvery`-Parameter – damit überlappt jeder Buffer und du erkennst zwei aufeinanderfolgende Klicks.
</details>

<details>
<summary>Hint 3 – windowTime vs. bufferTime</summary>

`windowTime` gibt Observables zurück, keine Arrays. Nutze `mergeMap` um auf das innere Observable zuzugreifen:

```typescript
source$.pipe(
  windowTime(2000),
  mergeMap(window$ => window$.pipe(toArray())),
  filter(arr => arr.length > 0)
)
```

Nützlich wenn du innerhalb des Fensters noch Operatoren (z. B. `take`, `reduce`) anwenden willst, bevor du das Ergebnis weiterverarbeitest.
</details>

## Beispiellösung

```typescript
// action-logger.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { bufferTime, filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class ActionLoggerService implements OnDestroy {
  private action$ = new Subject<string>();

  logAction(name: string): void {
    this.action$.next(name);
  }

  getBatches$(): Observable<string[]> {
    return this.action$.pipe(
      bufferTime(2000),
      filter(batch => batch.length > 0)
    );
  }

  ngOnDestroy(): void {
    this.action$.complete();
  }
}
```

```typescript
// batch-demo.component.ts
import { Component, ElementRef, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { fromEvent, Subscription } from 'rxjs';
import { bufferTime, filter } from 'rxjs/operators';
import { ActionLoggerService } from './action-logger.service';

@Component({
  selector: 'app-batch-demo',
  standalone: true,
  template: `
    <button (click)="logAction('Button geklickt')">Aktion ausführen</button>
    <button #dblBtn>Hier doppelklicken</button>
    <p>{{ statusMessage }}</p>
    <h3>Gesendete Batches:</h3>
    <ul>
      @for (batch of sentBatches; track $index) {
        <li>Batch #{{ $index + 1 }}: {{ batch.join(', ') }}</li>
      }
    </ul>
  `
})
export class BatchDemoComponent implements OnInit, OnDestroy {
  @ViewChild('dblBtn', { static: true }) dblBtn!: ElementRef<HTMLButtonElement>;

  statusMessage = '';
  sentBatches: string[][] = [];

  private subs = new Subscription();

  constructor(private logger: ActionLoggerService) {}

  ngOnInit(): void {
    // Batch-Logging
    this.subs.add(
      this.logger.getBatches$().subscribe(batch => {
        console.log('Batch senden:', batch);
        this.sentBatches.push(batch);
      })
    );

    // Doppelklick-Erkennung
    this.subs.add(
      fromEvent(this.dblBtn.nativeElement, 'click').pipe(
        bufferTime(300),
        filter(clicks => clicks.length === 2)
      ).subscribe(() => {
        this.statusMessage = '✅ Doppelklick erkannt! (' + new Date().toLocaleTimeString() + ')';
      })
    );
  }

  logAction(name: string): void {
    this.logger.logAction(name);
    this.statusMessage = `Aktion geloggt: "${name}"`;
  }

  ngOnDestroy(): void {
    this.subs.unsubscribe();
  }
}
```

```typescript
// Bonus: bufferCount für Sliding-Window Doppelklick
fromEvent(button, 'click').pipe(
  bufferCount(2, 1), // Fenster von 2, bei jedem Klick neu starten
  filter(([first, second]) => {
    const timeDiff = (second as MouseEvent).timeStamp - (first as MouseEvent).timeStamp;
    return timeDiff < 300;
  })
).subscribe(() => console.log('Präziser Doppelklick!'));
```

## Weiterführendes
- **`bufferWhen` & `bufferToggle`**: Für dynamisch gesteuerte Puffer – z. B. „puffere solange ein Modal offen ist".
- **`pairwise()`**: Wenn nur das aktuelle und das vorherige Element verglichen werden sollen (einfacher als `bufferCount(2, 1)`).
- **RxJS-Docs**: [bufferTime](https://rxjs.dev/api/operators/bufferTime) | [windowTime](https://rxjs.dev/api/operators/windowTime)
- **Tipp**: `windowTime` ist effizienter als `bufferTime` bei sehr hohem Event-Volumen, weil die Daten direkt weiterfließen können ohne auf das Ende des Puffers warten zu müssen.
