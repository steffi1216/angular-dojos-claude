# Angular Dojo: RxJS-Signal Interoperabilität
**Datum:** 2026-07-12
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, Observables und Signals mit den offiziellen Interop-Funktionen `toSignal()`, `toObservable()` und `takeUntilDestroyed()` aus `@angular/core/rxjs-interop` zu verbinden – und weißt, wann du welche Brücke verwendest.

## Hintergrund & Theorie

Angular 16 führte `@angular/core/rxjs-interop` ein: ein offizielles Paket, das die Lücke zwischen dem klassischen RxJS-Ökosystem und den neuen Signals schließt. In der Praxis existieren beide Welten parallel – älterer Code nutzt Observables, neue Komponenten bevorzugen Signals. Ohne Interop müsstest du manuell subscriben und aufräumen.

### `toSignal(observable$, options?)`
Konvertiert ein Observable in ein readonly Signal. Intern wird automatisch subscribed und bei Zerstörung des Injection Context (z. B. Komponente) unsubscribed. Der Wert ist sofort synchron lesbar.

- Muss im Injection Context aufgerufen werden (Konstruktor, `inject()`-Kontext, `runInInjectionContext`)
- `options.initialValue` setzt den Anfangswert (sonst `undefined` bzw. `Signal<T | undefined>`)
- `options.requireSync: true` erzwingt, dass das Observable synchron einen Wert emittiert (kein `undefined`)

### `toObservable(signal, options?)`
Konvertiert ein Signal in ein Observable. Intern nutzt es `effect()`, um Signal-Änderungen in einen `Subject` zu pushen. Das Observable emittiert immer sofort den aktuellen Wert (wie `BehaviorSubject`).

### `takeUntilDestroyed(destroyRef?)`
Ein RxJS-Operator (kein Signal-Konzept), der eine Subscription automatisch beendet, wenn der aktuelle Injection Context zerstört wird. Ersetzt das klassische `Subject`-`takeUntil`-Pattern sauber.

```typescript
// Klassisch – umständlich
private destroy$ = new Subject<void>();
ngOnDestroy() { this.destroy$.next(); this.destroy$.complete(); }
obs$.pipe(takeUntil(this.destroy$)).subscribe(...)

// Modern – eine Zeile
obs$.pipe(takeUntilDestroyed()).subscribe(...)
```

## Aufgabe

Baue einen `SearchComponent`, der eine Sucheingabe (Signal) mit einem simulierten HTTP-Service (Observable) verbindet und das Ergebnis als Signal anzeigt – mit automatischem Cleanup.

### Schritte

1. Erstelle einen `SearchService` mit einer Methode `search(query: string): Observable<string[]>`, die mit `of([...]).pipe(delay(300))` eine asynchrone Suche simuliert.

2. Erstelle einen `SearchComponent` als Standalone-Komponente:
   - Ein `query = signal('')` für die Sucheingabe
   - Konvertiere `query` via `toObservable()` in ein Observable
   - Wende `debounceTime(300)` und `distinctUntilChanged()` an, dann `switchMap` auf `service.search()`
   - Konvertiere das Ergebnis-Observable via `toSignal()` zurück in ein Signal `results`

3. Zeige im Template die Suchergebnisse mit `@for` an. Binde das Input-Feld an `query` mit einem `(input)`-Event.

4. Füge eine zweite Subscription hinzu (z. B. für ein Logging-Observable), die du explizit mit `takeUntilDestroyed()` absicherst, um das Cleanup-Pattern zu demonstrieren.

5. Prüfe mit `console.log` und Angular DevTools, dass keine Subscriptions nach dem Destroy der Komponente aktiv bleiben.

### Schritte (Detail)

1. Service anlegen: `ng g s search --skip-tests`
2. `SearchComponent` mit `standalone: true` und `ChangeDetectionStrategy.OnPush` erstellen
3. Interop-Funktionen aus `@angular/core/rxjs-interop` importieren
4. `toObservable(query)` → `pipe(debounceTime, distinctUntilChanged, switchMap)` → `toSignal()`
5. Template mit `[(ngModel)]` oder Signal-Binding bauen, Ergebnisliste rendern

## Hints

<details>
<summary>Hint 1 – Injection Context beachten</summary>

`toSignal()` und `toObservable()` müssen im Injection Context aufgerufen werden. Der einfachste Ort ist der Klassenkörper direkt bei der Property-Deklaration oder im Konstruktor – nicht in `ngOnInit()`.

```typescript
@Component({ ... })
export class SearchComponent {
  private searchService = inject(SearchService);

  query = signal('');

  // Korrekt: im Klassenkörper (Injection Context)
  private query$ = toObservable(this.query);
  results = toSignal(
    this.query$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(q => this.searchService.search(q))
    ),
    { initialValue: [] as string[] }
  );
}
```

</details>

<details>
<summary>Hint 2 – takeUntilDestroyed außerhalb des Injection Context</summary>

Wenn du `takeUntilDestroyed()` außerhalb des Konstruktors nutzen willst (z. B. in einer Methode), musst du `DestroyRef` explizit injecten und übergeben:

```typescript
private destroyRef = inject(DestroyRef);

ngOnInit() {
  interval(1000)
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(n => console.log('tick', n));
}
```

Im Konstruktor oder bei Property-Deklarationen funktioniert `takeUntilDestroyed()` ohne Argument, weil der Injection Context automatisch verfügbar ist.

</details>

<details>
<summary>Hint 3 – SearchService Stub</summary>

```typescript
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay } from 'rxjs/operators';

const MOCK_DATA = ['Angular', 'Angular CDK', 'Angular Signals', 'AngularFire',
                   'RxJS', 'NgRx', 'TypeScript', 'Tailwind'];

@Injectable({ providedIn: 'root' })
export class SearchService {
  search(query: string): Observable<string[]> {
    if (!query.trim()) return of([]);
    const results = MOCK_DATA.filter(item =>
      item.toLowerCase().includes(query.toLowerCase())
    );
    return of(results).pipe(delay(300));
  }
}
```

</details>

## Beispiellösung

```typescript
// search.service.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay } from 'rxjs/operators';

const MOCK_DATA = ['Angular', 'Angular CDK', 'Angular Signals', 'AngularFire',
                   'RxJS', 'NgRx', 'TypeScript', 'Tailwind', 'Zone.js'];

@Injectable({ providedIn: 'root' })
export class SearchService {
  search(query: string): Observable<string[]> {
    if (!query.trim()) return of([]);
    return of(
      MOCK_DATA.filter(item => item.toLowerCase().includes(query.toLowerCase()))
    ).pipe(delay(300));
  }
}

// search.component.ts
import { Component, ChangeDetectionStrategy, DestroyRef, inject, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { toSignal, toObservable, takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, switchMap, tap } from 'rxjs/operators';
import { interval } from 'rxjs';
import { SearchService } from './search.service';

@Component({
  selector: 'app-search',
  standalone: true,
  imports: [CommonModule, FormsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <input
      type="text"
      placeholder="Suchen..."
      [value]="query()"
      (input)="query.set($any($event.target).value)"
    />

    @if (results() === undefined) {
      <p>Lädt...</p>
    } @else if (results()!.length === 0) {
      <p>Keine Ergebnisse.</p>
    } @else {
      <ul>
        @for (item of results(); track item) {
          <li>{{ item }}</li>
        }
      </ul>
    }
  `
})
export class SearchComponent {
  private searchService = inject(SearchService);
  private destroyRef = inject(DestroyRef);

  query = signal('');

  // Signal → Observable → transformieren → Signal
  private query$ = toObservable(this.query);

  results = toSignal(
    this.query$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(q => this.searchService.search(q))
    ),
    { initialValue: [] as string[] }
  );

  constructor() {
    // takeUntilDestroyed ohne Argument funktioniert im Injection Context (Konstruktor)
    interval(5000)
      .pipe(
        takeUntilDestroyed(),
        tap(n => console.log(`[SearchComponent] Heartbeat #${n}`))
      )
      .subscribe();

    // Alternativ: DestroyRef explizit übergeben (nötig außerhalb des Konstruktors)
    this.query$
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        tap(q => console.log('[SearchComponent] Query geändert:', q))
      )
      .subscribe();
  }
}
```

## Weiterführendes

- **`outputFromObservable()`** (Angular 17.3+): Konvertiert ein Observable direkt in einen `output()` – nützlich für reaktive Event-Emitter ohne manuelle Subscription.
- **`outputToObservable()`**: Konvertiert einen `output()` zurück in ein Observable für externe RxJS-Pipelines.
- **Tipp:** Verwende `toSignal()` mit `{ requireSync: true }` nur bei Observables, die garantiert synchron emittieren (z. B. `BehaviorSubject`), da die App sonst zur Laufzeit einen Fehler wirft.
- **Tipp:** In zoneless-Anwendungen ist `toSignal()` besonders wertvoll, weil es Observables in das Signal-basierte Change-Detection-System einbindet, ohne `markForCheck()` manuell aufrufen zu müssen.
- [Angular Docs – RxJS Interop](https://angular.dev/guide/rxjs-interop)
