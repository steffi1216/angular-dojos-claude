# Angular Dojo: fakeAsync, tick und flushMicrotasks
**Datum:** 2026-09-12
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, asynchronen Angular-Code (Timers, Promises, setTimeout, Intervals) in Unit Tests mit `fakeAsync`, `tick` und `flushMicrotasks` synchron und deterministisch zu testen – ohne echte Wartezeiten.

## Hintergrund & Theorie

Angular-Tests laufen in Zone.js, und `fakeAsync` nutzt die **fake async zone** um die Zeit vollständig zu kontrollieren. Innerhalb eines `fakeAsync`-Blocks gibt es eine künstliche Uhr:

- **`tick(millis?)`** – lässt die gefakte Uhr um `millis` Millisekunden vorrücken und führt alle bis dahin fälligen `setTimeout`/`setInterval`-Callbacks aus. Ohne Argument: 0 ms (Microtask-Flush + nächster Tick).
- **`flushMicrotasks()`** – leert nur die Microtask-Queue (Promises, `queueMicrotask`), ohne die Makrotask-Queue (setTimeout) anzufassen.
- **`flush()`** (aus `@angular/core/testing`) – führt *alle* ausstehenden Makrotasks aus, egal wie weit in der Zukunft sie liegen.
- **`discardPeriodicTasks()`** – verhindert den Fehler „X timer(s) still in the queue" bei offenen `setInterval`-Aufrufen am Ende eines Tests.

Der Unterschied zu `async`/`await` in Tests ist entscheidend: `async` wartet in Echtzeit, `fakeAsync` setzt die Zeit vor. Das macht Tests schnell und deterministisch.

> **Achtung:** `fakeAsync` unterstützt seit Angular 16+ nativ auch echte Promises und `async/await`-Code – kein `flushMicrotasks()` mehr zwingend nötig, aber hilfreich für explizite Kontrolle.

## Aufgabe

Implementiere einen `CountdownService` mit folgender API:

```typescript
class CountdownService {
  value = signal(10);
  start(from: number): void { ... }  // zählt jede Sekunde runter
  stop(): void { ... }               // stoppt den Countdown
}
```

Der Service verwendet `setInterval` intern. Schreibe dann **fünf Tests** mit `fakeAsync`:

1. Startet bei `from` (Wert nach `start(5)` = 5)
2. Zählt nach 1 Sekunde um 1 runter
3. Zählt nach 3 Sekunden auf 0 runter (von 3 gestartet)
4. Stoppt korrekt (kein weiteres Herunterzählen nach `stop()`)
5. Negativer Wert: Wird kein Timer-Leak-Fehler geworfen, wenn `stop()` vor Ende aufgerufen wird?

### Schritte

1. Erstelle `countdown.service.ts` mit `signal` und `setInterval`-Logik
2. Erstelle `countdown.service.spec.ts` mit `TestBed`-Setup
3. Schreibe die Tests 1–3 mit `fakeAsync` + `tick(1000)`
4. Schreibe Test 4: rufe `stop()` nach 1 Tick auf, ticke weitere 2 Sekunden und prüfe, dass sich der Wert nicht mehr ändert
5. Schreibe Test 5: stelle sicher, dass kein `discardPeriodicTasks()` nötig ist, weil `stop()` den Timer korrekt löscht

## Hints

<details>
<summary>Hint 1 – Service-Struktur</summary>

```typescript
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CountdownService {
  value = signal(0);
  private intervalId: ReturnType<typeof setInterval> | null = null;

  start(from: number): void {
    this.stop();
    this.value.set(from);
    this.intervalId = setInterval(() => {
      this.value.update(v => v - 1);
    }, 1000);
  }

  stop(): void {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}
```

</details>

<details>
<summary>Hint 2 – fakeAsync-Teststruktur</summary>

```typescript
import { fakeAsync, tick, TestBed } from '@angular/core/testing';
import { CountdownService } from './countdown.service';

describe('CountdownService', () => {
  let service: CountdownService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(CountdownService);
  });

  it('should start at the given value', fakeAsync(() => {
    service.start(5);
    expect(service.value()).toBe(5);
    service.stop(); // kein Timer-Leak
  }));

  it('should decrement after 1 second', fakeAsync(() => {
    service.start(5);
    tick(1000);
    expect(service.value()).toBe(4);
    service.stop();
  }));
});
```

</details>

<details>
<summary>Hint 3 – Promise + flushMicrotasks</summary>

Wenn dein Service statt `setInterval` ein `Promise` nutzt, braucht man `flushMicrotasks()`:

```typescript
it('should resolve after promise', fakeAsync(() => {
  let result: string | undefined;
  Promise.resolve('done').then(v => result = v);

  expect(result).toBeUndefined(); // noch nicht aufgelöst!
  flushMicrotasks();
  expect(result).toBe('done');    // jetzt aufgelöst
}));
```

Für `async/await` in `fakeAsync` reicht oft ein `tick(0)` oder `await Promise.resolve()` – Angular 16+ flush-t Microtasks automatisch nach `tick()`.

</details>

## Beispiellösung

```typescript
// countdown.service.ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CountdownService {
  value = signal(0);
  private intervalId: ReturnType<typeof setInterval> | null = null;

  start(from: number): void {
    this.stop();
    this.value.set(from);
    this.intervalId = setInterval(() => {
      this.value.update(v => v - 1);
    }, 1000);
  }

  stop(): void {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}

// countdown.service.spec.ts
import { fakeAsync, tick, TestBed } from '@angular/core/testing';
import { CountdownService } from './countdown.service';

describe('CountdownService', () => {
  let service: CountdownService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(CountdownService);
  });

  // Test 1: Startwert korrekt
  it('should initialize with the given value', fakeAsync(() => {
    service.start(5);
    expect(service.value()).toBe(5);
    service.stop();
  }));

  // Test 2: Nach 1 Sekunde 1 weniger
  it('should decrement by 1 after one second', fakeAsync(() => {
    service.start(5);
    tick(1000);
    expect(service.value()).toBe(4);
    service.stop();
  }));

  // Test 3: Nach 3 Sekunden auf 0 (von 3 gestartet)
  it('should count down to 0 after 3 seconds', fakeAsync(() => {
    service.start(3);
    tick(3000);
    expect(service.value()).toBe(0);
    service.stop();
  }));

  // Test 4: stop() verhindert weiteres Zählen
  it('should stop counting when stop() is called', fakeAsync(() => {
    service.start(5);
    tick(1000);
    expect(service.value()).toBe(4);

    service.stop();
    tick(2000); // 2 weitere Sekunden – sollte nichts passieren
    expect(service.value()).toBe(4); // unverändert!
  }));

  // Test 5: Kein Timer-Leak – kein discardPeriodicTasks() nötig
  it('should not leak timers when stop() is called before countdown ends', fakeAsync(() => {
    service.start(10);
    tick(500);
    service.stop(); // räumt den Interval auf
    // Kein discardPeriodicTasks() nötig → kein Fehler
    expect(service.value()).toBe(10); // noch kein Tick
  }));
});
```

## Weiterführendes

- **`flush()` vs `tick(Infinity)`**: `flush()` aus `@angular/core/testing` führt alle ausstehenden Makrotasks aus (auch weit in der Zukunft), `tick(Infinity)` entspricht dem, löst aber einen Overflow-Fehler aus wenn die Queue leer ist – lieber `flush()` nutzen.
- **`jasmine.clock()` vs `fakeAsync`**: In Jasmine-Projekten gibt es beides; `fakeAsync` ist Angular-spezifisch, integriert sich mit Zones und ist für Angular-Services und -Komponenten vorzuziehen.
- **Offizielle Doku**: [Angular Testing Async Code](https://angular.dev/guide/testing/components-scenarios#component-with-async-service)
- **Tipp**: Kombiniere `fakeAsync` mit `TestBed.flushEffects()` (ab Angular 18), um Signal-Effects in Tests deterministisch zu testen.
