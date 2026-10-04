# Angular Dojo: RxJS Polling Strategien
**Datum:** 2026-10-04
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit RxJS verschiedene Polling-Strategien (einfaches Intervall-Polling, bedingtes Polling und Exponential Backoff) in Angular Services implementierst und dabei sauber Ressourcen freigibst.

## Hintergrund & Theorie

Polling ist das regelmäßige Abfragen eines Servers nach neuen Daten. In Angular gibt es dafür mehrere RxJS-Ansätze:

**Einfaches Intervall-Polling** kombiniert `timer()` oder `interval()` mit `switchMap()`, um in regelmäßigen Abständen einen HTTP-Request auszulösen. `switchMap` storniert dabei den laufenden Request, falls der Timer erneut feuert, bevor die Antwort eintrifft.

**Bedingtes Polling** nutzt `repeat({ delay })` oder `delayWhen()`, um erst dann erneut zu pollen, wenn der vorherige Request abgeschlossen ist (Back-Pressure-sicher). Damit vermeidest du Anfragen-Stau bei langsamen Servern.

**Exponential Backoff** mit `retryWhen()` (Legacy) oder dem modernen `retry({ delay })` in RxJS 7+ wartet nach einem Fehler exponentiell länger (1s, 2s, 4s, 8s …), bevor ein erneuter Versuch gestartet wird.

Für das saubere Beenden des Pollings in Angular-Komponenten nutzt du `takeUntilDestroyed()` (ab Angular 16) oder `DestroyRef`.

Alle Strategien lassen sich mit `signal()` kombinieren, um den aktuellen Status reaktiv in der UI anzuzeigen.

## Aufgabe

Implementiere einen `PollingService`, der drei wiederverwendbare Polling-Methoden bereitstellt. Erstelle anschließend eine `StatusDashboardComponent`, die alle drei Varianten demonstriert.

### Schritte

1. **`PollingService` erstellen** mit drei Methoden:
   - `pollInterval<T>(url, intervalMs)` – feuert alle `intervalMs` Millisekunden; bricht den laufenden Request ab und startet einen neuen (`switchMap`).
   - `pollSequential<T>(url, delayMs)` – wartet, bis der vorherige Request abgeschlossen ist, und wartet dann `delayMs` Millisekunden (`repeat({ delay })`).
   - `pollWithBackoff<T>(url, baseDelayMs, maxRetries)` – bei HTTP-Fehler wird mit Exponential Backoff erneut versucht; bei Erfolg wird nach `baseDelayMs` erneut gepollt.

2. **`StatusDashboardComponent` erstellen** (Standalone):
   - Abonniere alle drei Streams mit `takeUntilDestroyed()`.
   - Speichere Ergebnisse und Fehler in Signals.
   - Zeige im Template den letzten Wert, den letzten Fehler und einen Poll-Zähler an.

3. **Fehler-Handling einbauen**: Verwende `catchError` im Service, damit ein einzelner HTTP-Fehler den Poll-Stream **nicht** beendet.

4. **Teste manuell**, indem du den Server im DevTools throttelst oder offline schaltest und beobachtest, wie der Backoff-Stream reagiert.

## Hints

<details>
<summary>Hint 1 – Intervall-Polling mit switchMap</summary>

```typescript
import { timer, switchMap } from 'rxjs';

pollInterval<T>(url: string, intervalMs = 5000): Observable<T> {
  return timer(0, intervalMs).pipe(
    switchMap(() => this.http.get<T>(url)),
    catchError(err => {
      console.error('Poll error:', err);
      return EMPTY; // Stream endet – für echtes Polling besser retry nutzen!
    })
  );
}
```

Achtung: `catchError` mit `EMPTY` beendet den Stream bei Fehler. Kombiniere es stattdessen mit `retry` oder gib einen Fallback-Wert zurück, damit der Timer weiterläuft.

</details>

<details>
<summary>Hint 2 – Sequentielles Polling mit repeat</summary>

```typescript
import { defer, timer } from 'rxjs';
import { repeat, catchError, of } from 'rxjs/operators';

pollSequential<T>(url: string, delayMs = 3000): Observable<T> {
  return defer(() => this.http.get<T>(url)).pipe(
    catchError(err => of(null as unknown as T)), // Fehler nicht weiterwerfen
    repeat({ delay: delayMs })
  );
}
```

`defer()` stellt sicher, dass bei jeder Wiederholung ein neues Observable erzeugt wird (neuer Request). `repeat({ delay })` startet den gesamten upstream nach dem Abschluss neu – aber erst nach `delayMs` Millisekunden.

</details>

<details>
<summary>Hint 3 – Exponential Backoff mit retry</summary>

```typescript
import { retry, timer } from 'rxjs';

pollWithBackoff<T>(url: string, baseDelayMs = 1000, maxRetries = 5): Observable<T> {
  return defer(() => this.http.get<T>(url)).pipe(
    retry({
      count: maxRetries,
      delay: (error, retryCount) => timer(baseDelayMs * Math.pow(2, retryCount - 1))
    }),
    repeat({ delay: baseDelayMs })
  );
}
```

`retry({ delay: fn })` ist seit RxJS 7 verfügbar und ersetzt das veraltete `retryWhen`. Die `delay`-Funktion erhält `retryCount` (ab 1), sodass du einfach `baseDelay * 2^(retryCount-1)` rechnen kannst.

</details>

## Beispiellösung

```typescript
// polling.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, defer, timer, EMPTY } from 'rxjs';
import { switchMap, catchError, repeat, retry, share } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class PollingService {
  private http = inject(HttpClient);

  pollInterval<T>(url: string, intervalMs = 5000): Observable<T> {
    return timer(0, intervalMs).pipe(
      switchMap(() =>
        this.http.get<T>(url).pipe(
          catchError(() => EMPTY) // Einzelfehler ignorieren, nächster Tick kommt
        )
      ),
      share()
    );
  }

  pollSequential<T>(url: string, delayMs = 3000): Observable<T | null> {
    return defer(() => this.http.get<T>(url)).pipe(
      catchError(() => defer(() => Promise.resolve(null))),
      repeat({ delay: delayMs }),
      share()
    );
  }

  pollWithBackoff<T>(url: string, baseDelayMs = 1000, maxRetries = 5): Observable<T> {
    return defer(() =>
      this.http.get<T>(url).pipe(
        retry({
          count: maxRetries,
          delay: (_err, attempt) => timer(baseDelayMs * Math.pow(2, attempt - 1))
        })
      )
    ).pipe(
      repeat({ delay: baseDelayMs }),
      share()
    );
  }
}
```

```typescript
// status-dashboard.component.ts
import { Component, signal, inject, OnInit } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { PollingService } from './polling.service';

interface StatusResponse {
  status: string;
  timestamp: number;
}

@Component({
  selector: 'app-status-dashboard',
  standalone: true,
  template: `
    <h2>Polling Dashboard</h2>

    <section>
      <h3>Intervall-Polling (switchMap, 3s)</h3>
      <p>Letzter Wert: {{ intervalResult() | json }}</p>
      <p>Fehler: {{ intervalError() }}</p>
    </section>

    <section>
      <h3>Sequentielles Polling (repeat, 2s Pause)</h3>
      <p>Letzter Wert: {{ sequentialResult() | json }}</p>
      <p>Anzahl Polls: {{ sequentialCount() }}</p>
    </section>

    <section>
      <h3>Backoff-Polling (retry, max 5 Versuche)</h3>
      <p>Letzter Wert: {{ backoffResult() | json }}</p>
      <p>Fehler: {{ backoffError() }}</p>
    </section>
  `
})
export class StatusDashboardComponent implements OnInit {
  private polling = inject(PollingService);

  intervalResult = signal<StatusResponse | null>(null);
  intervalError = signal<string | null>(null);
  sequentialResult = signal<StatusResponse | null>(null);
  sequentialCount = signal(0);
  backoffResult = signal<StatusResponse | null>(null);
  backoffError = signal<string | null>(null);

  private destroyRef = takeUntilDestroyed();

  ngOnInit() {
    this.polling
      .pollInterval<StatusResponse>('/api/status', 3000)
      .pipe(this.destroyRef)
      .subscribe({
        next: v => this.intervalResult.set(v),
        error: e => this.intervalError.set(e.message)
      });

    this.polling
      .pollSequential<StatusResponse>('/api/status', 2000)
      .pipe(this.destroyRef)
      .subscribe(v => {
        if (v) this.sequentialResult.set(v);
        this.sequentialCount.update(c => c + 1);
      });

    this.polling
      .pollWithBackoff<StatusResponse>('/api/status', 1000, 5)
      .pipe(this.destroyRef)
      .subscribe({
        next: v => this.backoffResult.set(v),
        error: e => this.backoffError.set(e.message)
      });
  }
}
```

## Weiterführendes

- **`webSocket()` aus `rxjs/webSocket`**: Statt Polling kann eine WebSocket-Verbindung Server-Push ermöglichen – effizienter bei häufigen Updates (siehe Dojo 2026-08-17).
- **`repeat({ count, delay })` Dokumentation**: [rxjs.dev/api/operators/repeat](https://rxjs.dev/api/operators/repeat) – `delay` akzeptiert auch eine Funktion `(value, count) => ObservableInput`, für dynamische Abstände.
- **`retry({ delay })` RxJS 7+**: Ersetzt das veraltete `retryWhen` vollständig; der zweite Parameter der `delay`-Funktion ist die Anzahl der bisherigen Versuche (1-basiert).
- **Kombination mit Signals**: Nutze `toSignal(pollStream$, { initialValue: null })` aus `@angular/core/rxjs-interop`, um Polling-Ergebnisse direkt in Signals umzuwandeln und Subscriptions automatisch zu verwalten.
