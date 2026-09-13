# Angular Dojo: RxJS Error Handling Strategies
**Datum:** 2026-09-13
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, HTTP-Fehler in Angular-Services mit RxJS robust abzufangen und mit Strategien wie `catchError`, `retry`, `retryWhen` und exponentiellem Backoff professionell zu behandeln.

## Hintergrund & Theorie
In produktiven Angular-Anwendungen reicht ein einfaches `.subscribe(data => ..., err => ...)` nicht aus. RxJS bietet eine Reihe von Operatoren, um Fehler elegant direkt in der Observable-Pipeline zu behandeln:

- **`catchError`**: Fängt einen Fehler ab und gibt entweder einen Fallback-Observable oder ein leeres Observable zurück. Der Stream terminiert danach.
- **`retry(n)`**: Abonniert die Quelle automatisch bis zu `n`-mal neu, wenn ein Fehler auftritt.
- **`retryWhen(errors$)`** (deprecated seit RxJS 7, aber noch weit verbreitet): Erlaubt komplexe Wiederholungslogik anhand eines Error-Streams.
- **`retry({ count, delay })`**: Die moderne Alternative ab RxJS 7 – unterstützt direkt `delay` (feste Wartezeit) oder eine `delay`-Funktion für exponentielles Backoff.

Eine häufige Produktionsstrategie ist **exponentielles Backoff**: Der erste Retry erfolgt nach 1 s, der zweite nach 2 s, der dritte nach 4 s usw. Dadurch wird ein überlastetes Backend nicht sofort mit erneuten Anfragen überhäuft.

Wichtig: `catchError` und `retry` müssen in der richtigen Reihenfolge gesetzt werden. `retry` vor `catchError` bedeutet: erst mehrfach versuchen, dann den verbleibenden Fehler abfangen.

## Aufgabe
Baue einen `DataService`, der einen HTTP-Endpunkt aufruft und dabei eine robuste Fehlerbehandlung implementiert:

1. Bei einem transienten Fehler (z. B. 503) soll der Request mit exponentiellem Backoff **bis zu 3-mal** wiederholt werden.
2. Nach allen fehlgeschlagenen Versuchen soll `catchError` ein leeres Array zurückgeben, damit die UI nicht abstürzt.
3. Schreibe einen Unit-Test mit `HttpTestingController`, der verifiziert, dass der Service bei einem Fehler das Fallback-Array zurückgibt.

### Schritte
1. Erstelle einen `DataService` mit einer Methode `getItems(): Observable<Item[]>`, die `HttpClient.get<Item[]>` aufruft.
2. Baue die Pipeline mit `retry({ count: 3, delay: retryDelay })` auf, wobei `retryDelay` exponentielles Backoff berechnet.
3. Füge nach dem `retry` ein `catchError` ein, das `of([])` zurückgibt und den Fehler optional in einen `ErrorService` loggt.
4. Schreibe den Unit-Test: Lass drei Requests mit 503 fehlschlagen und überprüfe, dass der vierte Request (falls vorhanden) oder das Fallback ausgegeben wird.

## Hints
<details>
<summary>Hint 1 – retry mit Backoff</summary>

Ab RxJS 7 akzeptiert `retry` ein Konfigurationsobjekt mit einer `delay`-Funktion:

```typescript
import { retry } from 'rxjs/operators';
import { timer } from 'rxjs';

retry({
  count: 3,
  delay: (error, retryCount) => timer(Math.pow(2, retryCount - 1) * 1000)
})
```

`retryCount` startet bei 1, also: 1 s → 2 s → 4 s.
</details>
<details>
<summary>Hint 2 – catchError und of([])</summary>

`catchError` erhält den Error und muss einen Observable zurückgeben.
Für einen sicheren Fallback:

```typescript
import { catchError } from 'rxjs/operators';
import { of } from 'rxjs';

catchError((err: HttpErrorResponse) => {
  console.error('Alle Versuche fehlgeschlagen:', err.status);
  return of([]);
})
```

Stelle sicher, dass `catchError` **nach** `retry` in der Pipe steht, damit erst alle Retries ausgeführt werden, bevor der Fehler abgefangen wird.
</details>
<details>
<summary>Hint 3 – Unit-Test mit HttpTestingController</summary>

In einem Unit-Test muss `fakeAsync` + `tick()` nicht zwingend verwendet werden, wenn du einfach den letzten Fehlerfall testest. Flush einfach alle erwarteten Requests mit einem Fehler:

```typescript
const req = httpMock.expectOne('/api/items');
req.flush('Server Error', { status: 503, statusText: 'Service Unavailable' });
// ... weitere Requests für Retries
httpMock.verify();
```

Beachte: Bei `retry({ count: 3 })` werden insgesamt **4 Requests** abgesetzt (1 Original + 3 Retries). Flush daher alle 4.
</details>

## Beispiellösung
```typescript
// data.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, of, timer } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';

export interface Item {
  id: number;
  name: string;
}

@Injectable({ providedIn: 'root' })
export class DataService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = '/api/items';

  getItems(): Observable<Item[]> {
    return this.http.get<Item[]>(this.apiUrl).pipe(
      retry({
        count: 3,
        delay: (_error, retryCount) => timer(Math.pow(2, retryCount - 1) * 1000),
      }),
      catchError((err: HttpErrorResponse) => {
        console.error(`Request failed after retries. Status: ${err.status}`);
        return of([] as Item[]);
      })
    );
  }
}
```

```typescript
// data.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { DataService, Item } from './data.service';

describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should return empty array after all retries fail', () => {
    let result: Item[] | undefined;

    service.getItems().subscribe(items => (result = items));

    // 1 Originalanfrage + 3 Retries = 4 Requests insgesamt
    for (let i = 0; i < 4; i++) {
      const req = httpMock.expectOne('/api/items');
      req.flush('Service Unavailable', { status: 503, statusText: 'Service Unavailable' });
    }

    expect(result).toEqual([]);
  });

  it('should return data on the first successful attempt', () => {
    const mockItems: Item[] = [{ id: 1, name: 'Test' }];
    let result: Item[] | undefined;

    service.getItems().subscribe(items => (result = items));

    const req = httpMock.expectOne('/api/items');
    req.flush(mockItems);

    expect(result).toEqual(mockItems);
  });
});
```

## Weiterführendes
- Kombiniere `retry` mit einem **Circuit-Breaker-Muster**: Zähle Fehler über Zeit und stoppe automatisch alle Requests für einen definierten Zeitraum – implementierbar mit `scan` + `throwError`.
- Sieh dir das RxJS-Dokument zu [`retry`](https://rxjs.dev/api/operators/retry) an, insbesondere den `resetOnSuccess`-Parameter, der den Retry-Zähler nach einem erfolgreichen Request zurücksetzt.
- Für globale HTTP-Fehlerbehandlung kombiniere dieses Muster mit einem **HTTP-Interceptor**, um Retry-Logik zentral für alle Requests bereitzustellen.
