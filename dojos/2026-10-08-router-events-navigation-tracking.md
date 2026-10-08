# Angular Dojo: Router Events & Navigation Tracking
**Datum:** 2026-10-08
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, den `Router.events`-Stream gezielt auszuwerten, um einen globalen Ladezustand mit Signals zu verwalten, Navigations­fehler abzufangen und ein sauberes `NavigationTracker`-Service zu bauen.

## Hintergrund & Theorie
Angular emittiert bei jeder Navigation eine Folge von Events über `Router.events` (ein `Observable<Event>`). Die wichtigsten Typen sind:

| Event | Bedeutung |
|---|---|
| `NavigationStart` | Navigation beginnt |
| `RoutesRecognized` | Route wurde erkannt |
| `GuardsCheckStart/End` | Guards werden geprüft |
| `ResolveStart/End` | Resolver laufen |
| `NavigationEnd` | Navigation abgeschlossen |
| `NavigationCancel` | Guard hat abgebrochen |
| `NavigationError` | Fehler beim Laden (z. B. Lazy-Chunk) |
| `NavigationSkipped` | Navigation war redundant (Angular 16+) |

Mit `filter()` und dem `instanceof`-Operator lassen sich Events typsicher filtern. Seit Angular 17+ kann man `withNavigationErrorHandler()` als `provideRouter`-Feature nutzen, um Fehler zentral zu behandeln. Kombiniert mit Signals entsteht ein eleganter, reaktiver Ladezustand ohne manuelle Subscription-Verwaltung.

## Aufgabe
Erstelle einen `NavigationTrackerService`, der:
1. Eine Signal-basierte `isNavigating` Flag pflegt
2. Fehlgeschlagene Navigationen in einem Signal-Array speichert
3. Die Dauer der letzten Navigation misst
4. Sich selbst über `DestroyRef` aufräumt

Erstelle zusätzlich eine `LoadingBarComponent`, die `isNavigating` als visuelle Fortschrittsleiste darstellt.

### Schritte
1. Erstelle `NavigationTrackerService` mit `inject(Router)` und `inject(DestroyRef)`.
2. Abonniere `router.events` und filtere auf `NavigationStart`, `NavigationEnd`, `NavigationCancel`, `NavigationError`.
3. Setze das `isNavigating`-Signal bei `NavigationStart` auf `true`, bei den anderen drei auf `false`.
4. Miss die Dauer: Speichere den Zeitstempel bei `NavigationStart`, berechne `Date.now() - start` bei `NavigationEnd`.
5. Füge bei `NavigationError` / `NavigationCancel` einen Eintrag zum `errors`-Signal-Array hinzu.
6. Registriere die Cleanup-Funktion über `destroyRef.onDestroy(() => subscription.unsubscribe())`.
7. Erstelle `LoadingBarComponent` als Standalone-Component, die den Service injiziert und bei `isNavigating()` eine CSS-Klasse setzt.
8. Registriere `withNavigationErrorHandler` in `provideRouter`, das zum Service delegiert.

## Hints
<details>
<summary>Hint 1 – Events filtern</summary>

```typescript
import { NavigationStart, NavigationEnd, NavigationError, NavigationCancel } from '@angular/router';

router.events.pipe(
  filter(e =>
    e instanceof NavigationStart ||
    e instanceof NavigationEnd ||
    e instanceof NavigationError ||
    e instanceof NavigationCancel
  )
).subscribe(event => { /* ... */ });
```

`instanceof` gibt TypeScript den konkreten Typ — kein Cast nötig.
</details>

<details>
<summary>Hint 2 – Signal-Update und Cleanup</summary>

```typescript
// Mutable Signal für Array-Updates
const errors = signal<string[]>([]);
// Update per Mutation:
errors.update(prev => [...prev, message]);

// Cleanup via DestroyRef
const sub = router.events.pipe(...).subscribe(...);
destroyRef.onDestroy(() => sub.unsubscribe());
```
</details>

<details>
<summary>Hint 3 – withNavigationErrorHandler</summary>

```typescript
// app.config.ts
provideRouter(
  routes,
  withNavigationErrorHandler((error: NavigationError) => {
    inject(NavigationTrackerService).recordError(error);
  })
)
```

`withNavigationErrorHandler` führt die Callback-Funktion in einem Injection Context aus — `inject()` funktioniert dort direkt.
</details>

## Beispiellösung

```typescript
// navigation-tracker.service.ts
import { Injectable, inject, signal, DestroyRef } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError, NavigationCancel } from '@angular/router';
import { filter } from 'rxjs/operators';

export interface NavErrorEntry {
  url: string;
  message: string;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class NavigationTrackerService {
  private router = inject(Router);
  private destroyRef = inject(DestroyRef);

  readonly isNavigating = signal(false);
  readonly lastDurationMs = signal<number | null>(null);
  readonly errors = signal<NavErrorEntry[]>([]);

  private startTime = 0;

  constructor() {
    const sub = this.router.events.pipe(
      filter(e =>
        e instanceof NavigationStart ||
        e instanceof NavigationEnd ||
        e instanceof NavigationError ||
        e instanceof NavigationCancel
      )
    ).subscribe(event => {
      if (event instanceof NavigationStart) {
        this.isNavigating.set(true);
        this.startTime = Date.now();
      } else if (event instanceof NavigationEnd) {
        this.isNavigating.set(false);
        this.lastDurationMs.set(Date.now() - this.startTime);
      } else if (event instanceof NavigationError) {
        this.isNavigating.set(false);
        this.recordError(event);
      } else if (event instanceof NavigationCancel) {
        this.isNavigating.set(false);
      }
    });

    this.destroyRef.onDestroy(() => sub.unsubscribe());
  }

  recordError(event: NavigationError): void {
    this.errors.update(prev => [
      ...prev,
      {
        url: event.url,
        message: event.error?.message ?? String(event.error),
        timestamp: Date.now(),
      },
    ]);
  }

  clearErrors(): void {
    this.errors.set([]);
  }
}
```

```typescript
// loading-bar.component.ts
import { Component, inject } from '@angular/core';
import { NavigationTrackerService } from './navigation-tracker.service';

@Component({
  selector: 'app-loading-bar',
  standalone: true,
  template: `
    <div class="loading-bar" [class.active]="tracker.isNavigating()">
      <div class="progress"></div>
    </div>
    @if (tracker.lastDurationMs() !== null) {
      <small>Letzte Navigation: {{ tracker.lastDurationMs() }}ms</small>
    }
    @if (tracker.errors().length > 0) {
      <ul class="nav-errors">
        @for (err of tracker.errors(); track err.timestamp) {
          <li>{{ err.url }}: {{ err.message }}</li>
        }
      </ul>
    }
  `,
  styles: [`
    .loading-bar { height: 3px; background: transparent; }
    .loading-bar.active { background: linear-gradient(90deg, #3f51b5 0%, #9c27b0 100%); animation: slide 1s infinite; }
    @keyframes slide { from { transform: scaleX(0); } to { transform: scaleX(1); } }
  `],
})
export class LoadingBarComponent {
  readonly tracker = inject(NavigationTrackerService);
}
```

```typescript
// app.config.ts
import { provideRouter, withNavigationErrorHandler } from '@angular/router';
import { NavigationError } from '@angular/router';
import { inject } from '@angular/core';
import { NavigationTrackerService } from './navigation-tracker.service';

export const appConfig = {
  providers: [
    provideRouter(
      routes,
      withNavigationErrorHandler((error: NavigationError) => {
        inject(NavigationTrackerService).recordError(error);
      })
    ),
  ],
};
```

## Weiterführendes
- **`NavigationSkipped`** (Angular 16+): Wird emittiert, wenn eine Navigation zur selben URL ohne Änderungen abgebrochen wird — nützlich um unnötige Lade-Animationen zu vermeiden.
- **`router.getCurrentNavigation()`**: Gibt innerhalb einer laufenden Navigation das `Navigation`-Objekt zurück (inklusive `extras.state` für Navigationszustand).
- Kombination mit **`@angular/core/rxjs-interop`'s `toSignal()`**: Statt manueller Subscription kann der gesamte Events-Stream als Signal exponiert werden — `toSignal(router.events, { initialValue: null })`.
