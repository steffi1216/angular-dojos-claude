# Angular Dojo: RxJS Retry Strategien mit exponentiellem Backoff
**Datum:** 2026-10-28
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem modernisierten `retry()`-Operator aus RxJS 7+ robuste HTTP-Anfragen mit exponentiellem Backoff und jitter implementierst – ein unverzichtbares Muster für resiliente Angular-Anwendungen.

## Hintergrund & Theorie

Transiente Netzwerkfehler sind in der Praxis unvermeidlich. Statt Fehler sofort an den User weiterzugeben, kann ein intelligentes Retry-Muster die Nutzererfahrung erheblich verbessern.

**`retry()` in RxJS 7+** akzeptiert eine Konfiguration statt nur einer Zahl:

```typescript
retry({
  count: 3,
  delay: (error, retryCount) => timer(1000 * Math.pow(2, retryCount))
})
```

Schlüsselkonzepte:

- **Exponential Backoff:** Wartezeit verdoppelt sich mit jedem Versuch (1s → 2s → 4s), um Server nicht zu überlasten.
- **Jitter:** Zufällige Verzögerung addiert auf die Wartezeit verhindert, dass viele Clients gleichzeitig retry ausführen ("Thundering Herd").
- **`shouldRetry`:** Selektives Retry nur bei bestimmten HTTP-Statuscodes (z.B. 503, 429 – nicht bei 401, 404).
- **`resetOnSuccess`:** Zähler zurücksetzen bei erfolgreichem Versuch in einem Long-lived Observable.

Dieses Muster lässt sich hervorragend in einem HTTP-Interceptor kapseln, sodass es automatisch für alle Anfragen greift.

## Aufgabe

Implementiere einen `RetryInterceptor` als funktionalen HTTP-Interceptor, der transiente Fehler (5xx, 429) mit exponentiellem Backoff + Jitter automatisch wiederholt. Ein nicht-retriable Fehler (4xx außer 429) soll sofort weitergegeben werden.

### Schritte

1. Erstelle einen Interceptor `retry.interceptor.ts` mit `HttpInterceptorFn`.
2. Implementiere eine Hilfsfunktion `getRetryDelay(retryCount: number): number`, die exponentiellen Backoff (Basis 1000ms, max 30s) plus zufälligen Jitter (0–1000ms) berechnet.
3. Implementiere eine Funktion `isRetryable(error: HttpErrorResponse): boolean`, die nur bei Statuscodes 429, 500, 502, 503, 504 `true` zurückgibt.
4. Nutze `retry({ count: 3, delay, shouldRetry })` im Interceptor.
5. Registriere den Interceptor in `app.config.ts` via `withInterceptors([retryInterceptor])`.
6. Schreibe einen Unit-Test mit `HttpTestingController`, der verifiziert, dass bei einem 503-Fehler insgesamt 4 Requests (1 original + 3 retries) gestellt werden.

## Hints

<details>
<summary>Hint 1 – Aufbau des retry()-Operators</summary>

```typescript
import { retry, timer } from 'rxjs';
import { HttpErrorResponse } from '@angular/common/http';

// In deinem Interceptor:
return next(req).pipe(
  retry({
    count: 3,
    delay: (error: HttpErrorResponse, retryCount: number) => {
      if (!isRetryable(error)) {
        throw error; // Sofort weiterwerfen, kein Retry
      }
      return timer(getRetryDelay(retryCount));
    },
  })
);
```

`delay` gibt ein Observable zurück – sobald es emittiert, wird der nächste Versuch gestartet. Wirfst du den Fehler innerhalb von `delay`, bricht der Retry ab.

</details>

<details>
<summary>Hint 2 – Jitter-Berechnung und Test mit fakeAsync</summary>

```typescript
// Jitter: deterministisch testbar mit Math.random-Mock
function getRetryDelay(retryCount: number): number {
  const exponential = Math.min(1000 * Math.pow(2, retryCount), 30_000);
  const jitter = Math.floor(Math.random() * 1000);
  return exponential + jitter;
}

// Im Test: fakeAsync + tick() für timer()-Delays
it('retries 3 times on 503', fakeAsync(() => {
  let result: any;
  service.getData().subscribe({ next: v => (result = v), error: () => {} });

  for (let i = 0; i <= 3; i++) {
    const req = httpMock.expectOne('/api/data');
    req.flush('error', { status: 503, statusText: 'Service Unavailable' });
    tick(35_000); // Warte länger als max Backoff + Jitter
  }
  httpMock.verify();
}));
```

Für exakte Delay-Tests kannst du `Math.random` im Test auf `() => 0` mocken.

</details>

## Beispiellösung

```typescript
// retry.interceptor.ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { retry, timer } from 'rxjs';

const RETRYABLE_STATUS_CODES = new Set([429, 500, 502, 503, 504]);
const MAX_RETRIES = 3;
const BASE_DELAY_MS = 1000;
const MAX_DELAY_MS = 30_000;

function isRetryable(error: unknown): boolean {
  return (
    error instanceof HttpErrorResponse &&
    RETRYABLE_STATUS_CODES.has(error.status)
  );
}

function getRetryDelay(retryCount: number): number {
  const exponential = Math.min(BASE_DELAY_MS * Math.pow(2, retryCount), MAX_DELAY_MS);
  const jitter = Math.floor(Math.random() * 1000);
  return exponential + jitter;
}

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    retry({
      count: MAX_RETRIES,
      delay: (error: unknown, retryCount: number) => {
        if (!isRetryable(error)) {
          throw error;
        }
        console.warn(`[RetryInterceptor] Attempt ${retryCount}/${MAX_RETRIES} for ${req.url} after ${getRetryDelay(retryCount)}ms`);
        return timer(getRetryDelay(retryCount));
      },
    })
  );
};
```

```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { retryInterceptor } from './retry.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([retryInterceptor])),
  ],
};
```

```typescript
// retry.interceptor.spec.ts
import { TestBed } from '@angular/core/testing';
import {
  HttpClient,
  provideHttpClient,
  withInterceptors,
} from '@angular/common/http';
import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';
import { fakeAsync, tick } from '@angular/core/testing';
import { retryInterceptor } from './retry.interceptor';

describe('retryInterceptor', () => {
  let http: HttpClient;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    // Jitter auf 0 fixieren für deterministische Tests
    jest.spyOn(Math, 'random').mockReturnValue(0);

    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withInterceptors([retryInterceptor])),
        provideHttpClientTesting(),
      ],
    });
    http = TestBed.inject(HttpClient);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
    jest.restoreAllMocks();
  });

  it('retries 3 times on 503, then errors', fakeAsync(() => {
    let errorReceived = false;
    http.get('/api/data').subscribe({ error: () => (errorReceived = true) });

    // Original request + 3 retries = 4 requests total
    for (let i = 0; i < 4; i++) {
      const req = httpMock.expectOne('/api/data');
      req.flush('error', { status: 503, statusText: 'Service Unavailable' });
      tick(35_000); // Überspringen des Backoff-Timers
    }

    expect(errorReceived).toBe(true);
  }));

  it('does NOT retry on 404', fakeAsync(() => {
    let errorReceived = false;
    http.get('/api/data').subscribe({ error: () => (errorReceived = true) });

    const req = httpMock.expectOne('/api/data');
    req.flush('not found', { status: 404, statusText: 'Not Found' });
    tick(1000);

    expect(errorReceived).toBe(true);
    // Sicherstellen, dass kein weiterer Request gestellt wurde
    httpMock.expectNone('/api/data');
  }));

  it('succeeds if retry resolves', fakeAsync(() => {
    let result: unknown;
    http.get('/api/data').subscribe({ next: v => (result = v) });

    // Erster Request schlägt fehl
    httpMock.expectOne('/api/data').flush('error', { status: 503, statusText: 'Service Unavailable' });
    tick(2000);

    // Zweiter Request (Retry 1) erfolgreich
    httpMock.expectOne('/api/data').flush({ id: 42 });
    tick(0);

    expect(result).toEqual({ id: 42 });
  }));
});
```

## Weiterführendes

- **`retryWhen` ist deprecated** seit RxJS 7 – migriere bestehenden Code auf `retry({ delay })`.
- Kombiniere den Interceptor mit einem **`retryBudget`** (globaler Zähler für Retries pro Zeitfenster), um den Server bei Ausfällen zu schonen.
- Für **offline-fähige Apps**: Kombiniere mit `fromEvent(window, 'online')` als Retry-Signal – retry erst, wenn die Netzwerkverbindung wieder da ist:
  ```typescript
  delay: (error) => {
    if (!isRetryable(error)) throw error;
    return fromEvent(window, 'online').pipe(take(1));
  }
  ```
- Offizielle Doku: [RxJS retry](https://rxjs.dev/api/operators/retry)
