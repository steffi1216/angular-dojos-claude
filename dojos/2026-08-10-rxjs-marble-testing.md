# Angular Dojo: RxJS Marble Testing
**Datum:** 2026-08-10
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, komplexe asynchrone RxJS-Streams mit der `TestScheduler`-API und Marble-Diagrammen präzise und deterministisch zu testen, ohne echte Timer oder Delays abwarten zu müssen.

## Hintergrund & Theorie

Marble Testing ist eine Technik, um RxJS-Observables mit virtueller Zeit zu testen. Anstatt `setTimeout` oder `fakeAsync` zu nutzen, beschreibt man den zeitlichen Verlauf von Streams als ASCII-String (ein sogenanntes „Marble Diagram"):

```
'-a--b---c|'
```

Bedeutung der Zeichen:
- `-` = 10ms vergehen (ein Frame)
- `a`, `b`, `c` = emittierte Werte (werden über ein `values`-Objekt gemappt)
- `|` = Stream completed
- `#` = Stream throws error
- `(ab)` = mehrere Werte im selben Frame

Die `TestScheduler`-API aus `rxjs/testing` erlaubt es, diese virtuellen Zeitframes zu steuern und Erwartungen zu formulieren:

```typescript
scheduler.run(({ cold, hot, expectObservable }) => {
  const source$ = cold('-a--b|', { a: 1, b: 2 });
  expectObservable(source$.pipe(map(x => x * 2))).toBe('-a--b|', { a: 2, b: 4 });
});
```

`cold()` erstellt ein Observable, das erst bei Subscription startet.  
`hot()` erstellt ein Observable, das unabhängig von Subscriptions „bereits läuft" (z. B. ein Subject/EventStream).

Mit Marble Testing kannst du Operatoren wie `debounceTime`, `switchMap`, `delay`, `throttleTime` vollständig deterministisch und ohne Wartezeiten testen.

## Aufgabe

Du hast einen `SearchService`, der eine HTTP-Anfrage simuliert, und eine Suchkomponente, die mit `debounceTime` + `switchMap` auf Eingaben reagiert. Schreibe Unit-Tests mit Marble Diagrams, die das korrekte Verhalten dieser Pipe verifizieren:

1. Suchanfragen werden 300ms debounced
2. Nur die letzte Anfrage wird ausgeführt (switchMap-Verhalten)
3. Ein leerer Suchstring wird herausgefiltert

### Schritte

1. Erstelle die Datei `search.service.ts` mit einer Methode `search(query: string): Observable<string[]>`, die nach 200ms ein Ergebnis-Array emittiert (simuliere das mit `delay`).

2. Erstelle `search.pipe.ts` – eine reine Funktion (oder Methode), die einen `Observable<string>` (Eingabe-Stream) entgegennimmt und ihn durch folgende Pipe verarbeitet:
   ```
   filter(q => q.length > 0),
   debounceTime(300),
   switchMap(q => searchService.search(q))
   ```

3. Erstelle `search.pipe.spec.ts` und schreibe mit dem `TestScheduler` folgende Tests:
   - **Test 1:** Ein einzelner Suchterm nach 300ms Wartezeit gibt ein Ergebnis zurück
   - **Test 2:** Zwei schnell aufeinanderfolgende Terme (Abstand < 300ms) führen nur zu einem Ergebnis (debounce + switchMap)
   - **Test 3:** Ein leerer String wird gefiltert und löst keine Anfrage aus

## Hints

<details>
<summary>Hint 1 – TestScheduler-Setup</summary>

Importiere `TestScheduler` aus `rxjs/testing` und initialisiere ihn im `beforeEach`:

```typescript
import { TestScheduler } from 'rxjs/testing';

let scheduler: TestScheduler;

beforeEach(() => {
  scheduler = new TestScheduler((actual, expected) => {
    expect(actual).toEqual(expected);
  });
});
```

Wrapp alle Marble-Assertions in `scheduler.run(({ cold, hot, expectObservable }) => { ... })`.

</details>

<details>
<summary>Hint 2 – Zeit-Syntax im run()-Block</summary>

Innerhalb von `scheduler.run()` gilt die „neue" Syntax: 1 Frame = 1ms (statt 10ms außerhalb).  
`debounceTime(300)` entspricht also `300ms` = 300 Frames = `'300ms '` im Marble-String.

Für Delays nutze die Kurzform:
```typescript
const source$ = cold('300ms a', { a: 'angular' });
```

Für den `switchMap`-Test mit zwei schnellen Eingaben:
```typescript
const input$ = hot('a 100ms b', { a: 'ang', b: 'angular' });
// Nur 'angular' sollte die Anfrage auslösen, weil 100ms < 300ms debounce
```

</details>

<details>
<summary>Hint 3 – searchService mocken</summary>

Erstelle ein Mock-Objekt und nutze `cold()` als Rückgabewert für `search()`:

```typescript
scheduler.run(({ cold, hot, expectObservable }) => {
  const mockService = {
    search: (q: string) => cold('200ms r', { r: [`result for ${q}`] })
  };

  const input$ = hot('300ms a', { a: 'angular' });
  const result$ = applySearchPipe(input$, mockService);

  // debounce 300ms + delay 200ms = 500ms nach Start
  expectObservable(result$).toBe('500ms r', { r: ['result for angular'] });
});
```

</details>

## Beispiellösung

```typescript
// search.service.ts
import { Injectable } from '@angular/core';
import { Observable, delay, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SearchService {
  search(query: string): Observable<string[]> {
    return of([`Result for "${query}"`]).pipe(delay(200));
  }
}
```

```typescript
// search.pipe.ts
import { Observable } from 'rxjs';
import { debounceTime, filter, switchMap } from 'rxjs/operators';
import { SearchService } from './search.service';

export function applySearchPipe(
  input$: Observable<string>,
  service: Pick<SearchService, 'search'>
): Observable<string[]> {
  return input$.pipe(
    filter(q => q.length > 0),
    debounceTime(300),
    switchMap(q => service.search(q))
  );
}
```

```typescript
// search.pipe.spec.ts
import { TestScheduler } from 'rxjs/testing';
import { applySearchPipe } from './search.pipe';

describe('applySearchPipe', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('gibt nach debounce + delay ein Ergebnis zurück', () => {
    scheduler.run(({ cold, hot, expectObservable }) => {
      const mockService = {
        search: (_q: string) => cold('200ms r', { r: ['Result for "angular"'] }),
      };

      // Eingabe nach 300ms (debounce-Grenze exakt erreicht)
      const input$ = hot('300ms a', { a: 'angular' });
      const result$ = applySearchPipe(input$, mockService);

      // 300ms debounce + 200ms delay = 500ms
      expectObservable(result$).toBe('500ms r', { r: ['Result for "angular"'] });
    });
  });

  it('ignoriert früheren Term bei schnell aufeinanderfolgenden Eingaben (switchMap)', () => {
    scheduler.run(({ cold, hot, expectObservable }) => {
      let callCount = 0;
      const mockService = {
        search: (_q: string) => {
          callCount++;
          return cold('200ms r', { r: [`result-${callCount}`] });
        },
      };

      // Zwei Eingaben mit 100ms Abstand – weniger als debounce (300ms)
      // Nur die zweite ('angular') wird nach dem Debounce durchgelassen
      const input$ = hot('a 100ms b', { a: 'ang', b: 'angular' });
      const result$ = applySearchPipe(input$, mockService);

      // Debounce startet nach 'b' bei 101ms → feuert bei 401ms
      // + 200ms delay = 601ms
      expectObservable(result$).toBe('601ms r', { r: ['result-1'] });
    });
  });

  it('filtert leere Strings heraus und löst keine Anfrage aus', () => {
    scheduler.run(({ cold, hot, expectObservable }) => {
      const searchSpy = jest.fn().mockReturnValue(cold('200ms r', { r: [] }));
      const mockService = { search: searchSpy };

      const input$ = hot('a 400ms b', { a: '', b: '' });
      const result$ = applySearchPipe(input$, mockService);

      expectObservable(result$).toBe('');
      // Im Callback prüfen (nach scheduler.run):
      // expect(searchSpy).not.toHaveBeenCalled();
    });
  });
});
```

## Weiterführendes

- **Offizielle RxJS-Doku zu Marble Testing:** https://rxjs.dev/guide/testing/marble-testing — erklärt Hot vs. Cold Observables und alle Marble-Syntaxvarianten
- **Tipp:** Nutze die Hilfsmethode `expectSubscriptions()` aus dem `scheduler.run()`-Block, um zu prüfen, *wann* sich ein Observable subscribed und unsubscribed hat — besonders nützlich beim Debuggen von `switchMap`-Cancellations
- **Weiterführend:** Kombiniere Marble Testing mit `fakeAsync` für Komponenten-Tests, in denen sowohl Zone.js-Events als auch RxJS-Streams eine Rolle spielen
