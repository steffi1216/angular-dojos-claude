# Angular Dojo: EnvironmentInjector & makeEnvironmentProviders
**Datum:** 2026-08-25
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie der `EnvironmentInjector` funktioniert, wie du mit `makeEnvironmentProviders` wiederverwendbare, tree-shakeable Provider-Sets erstellst, und wie du Injection-Kontexte außerhalb der Komponentenstruktur steuerst.

## Hintergrund & Theorie

Angular kennt zwei Arten von Injektoren: **`ElementInjector`** (gebunden an Komponenten/Direktiven) und **`EnvironmentInjector`** (gebunden an das Bootstrap-Level bzw. Lazy-Loaded Routes). Mit Angular 14+ können Entwickler eigene `EnvironmentInjector`-Instanzen programmatisch erstellen — was besonders für Library-Autoren, Microfrontend-Setups und fortgeschrittene Testisolation wichtig ist.

`makeEnvironmentProviders()` erzeugt ein opakes `EnvironmentProviders`-Objekt, das **ausschließlich** in `bootstrapApplication()`, `provideEnvironmentInitializer()`, Route-`providers` oder `TestBed.configureTestingModule()` eingesetzt werden kann — nicht in `@Component.providers`. Das verhindert, dass "environment-level"-Services versehentlich im Element-Baum landen.

Mit `EnvironmentInjector.runInContext()` lassen sich Functions ausführen, die `inject()` benötigen, ohne dabei in einem Komponenten-Konstruktor zu sein — z. B. in Factories oder beim programmatischen Bootstrapping.

**Schlüssel-APIs:**
- `createEnvironmentInjector(providers, parent)` — erstellt einen Kind-Injector
- `makeEnvironmentProviders(providers)` — kapselt Provider-Arrays für Libraries
- `EnvironmentInjector.runInContext(fn)` — führt `fn` mit Injection-Kontext aus
- `inject(TOKEN)` — funktioniert innerhalb von `runInContext`

## Aufgabe

Erstelle eine wiederverwendbare **Analytics-Feature-Library** als Angular-Provider-Set mit `makeEnvironmentProviders`. Die Library soll:

1. Einen `AnalyticsService` anbieten, der Events trackt.
2. Einen `HTTP_INTERCEPTORS`-Interceptor registrieren, der alle Requests loggt.
3. Programmatisch mit `createEnvironmentInjector` einen isolierten Test-Injector aufbauen und darin per `runInContext` den Service instanziieren.

### Schritte

1. **`AnalyticsService` erstellen:** Ein Service mit einer Methode `track(event: string, data?: unknown): void`, die Events in eine interne `signal`-basierte Liste speichert (`events = signal<string[]>([])`).

2. **`analyticsInterceptor` erstellen:** Ein funktionaler HTTP-Interceptor (`HttpInterceptorFn`), der den Request-URL loggt, indem er `inject(AnalyticsService).track(...)` aufruft, bevor er `next(req)` weitergibt.

3. **`provideAnalytics()` mit `makeEnvironmentProviders` exportieren:**
   ```typescript
   export function provideAnalytics(): EnvironmentProviders {
     return makeEnvironmentProviders([
       AnalyticsService,
       withInterceptors([analyticsInterceptor]), // oder provideHttpClient(...)
     ]);
   }
   ```

4. **Isolierten Injector erstellen:** In einer `main.ts`-ähnlichen Datei (oder einer Spec) einen `createEnvironmentInjector`-Call schreiben, der `provideAnalytics()` und `provideHttpClient()` enthält. Nutze anschließend `injector.runInContext(() => inject(AnalyticsService))` um den Service zu holen und `track('app_start')` aufzurufen.

5. **Lebenszyklus beachten:** Rufe `injector.destroy()` auf, wenn der Injector nicht mehr benötigt wird, um Memory Leaks zu vermeiden.

## Hints

<details>
<summary>Hint 1 – makeEnvironmentProviders Signatur</summary>

`makeEnvironmentProviders` akzeptiert ein Array aus `Provider | EnvironmentProviders`. Du kannst also andere `provide*`-Funktionen direkt darin verschachteln:

```typescript
import { makeEnvironmentProviders, EnvironmentProviders } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export function provideAnalytics(): EnvironmentProviders {
  return makeEnvironmentProviders([
    AnalyticsService,
    provideHttpClient(withInterceptors([analyticsInterceptor])),
  ]);
}
```

Beachte: `provideHttpClient()` gibt selbst `EnvironmentProviders` zurück — daher darf es nur auf Environment-Level registriert werden.

</details>

<details>
<summary>Hint 2 – createEnvironmentInjector & runInContext</summary>

```typescript
import { createEnvironmentInjector, inject } from '@angular/core';
import { bootstrapApplication } from '@angular/platform-browser';

// Parent-Injector holen (in Tests: TestBed.inject(EnvironmentInjector))
const platformInjector = ...; // z. B. aus ApplicationRef

const childInjector = createEnvironmentInjector(
  [provideAnalytics()],
  platformInjector
);

const analytics = childInjector.runInContext(() => inject(AnalyticsService));
analytics.track('app_start', { source: 'main' });

// Cleanup
childInjector.destroy();
```

In einem Unit-Test kannst du `TestBed.inject(EnvironmentInjector)` als Parent verwenden.

</details>

<details>
<summary>Hint 3 – Funktionaler Interceptor mit inject()</summary>

Funktionale Interceptors laufen automatisch in einem Injection-Kontext, daher funktioniert `inject()` dort direkt:

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const analyticsInterceptor: HttpInterceptorFn = (req, next) => {
  const analytics = inject(AnalyticsService);
  analytics.track('http_request', { url: req.url, method: req.method });
  return next(req);
};
```

</details>

## Beispiellösung

```typescript
// analytics.service.ts
import { Injectable, signal, computed } from '@angular/core';

@Injectable()
export class AnalyticsService {
  private readonly _events = signal<Array<{ event: string; data?: unknown; ts: number }>>([]);

  readonly events = this._events.asReadonly();
  readonly eventCount = computed(() => this._events().length);

  track(event: string, data?: unknown): void {
    this._events.update(prev => [...prev, { event, data, ts: Date.now() }]);
    console.log(`[Analytics] ${event}`, data);
  }
}
```

```typescript
// analytics.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AnalyticsService } from './analytics.service';
import { tap } from 'rxjs';

export const analyticsInterceptor: HttpInterceptorFn = (req, next) => {
  const analytics = inject(AnalyticsService);
  const started = Date.now();
  analytics.track('http_request_start', { url: req.url, method: req.method });

  return next(req).pipe(
    tap({
      next: response => {
        const duration = Date.now() - started;
        analytics.track('http_request_success', {
          url: req.url,
          status: (response as any).status,
          duration,
        });
      },
      error: err => {
        analytics.track('http_request_error', {
          url: req.url,
          error: err.message,
        });
      },
    })
  );
};
```

```typescript
// provide-analytics.ts
import { EnvironmentProviders, makeEnvironmentProviders } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AnalyticsService } from './analytics.service';
import { analyticsInterceptor } from './analytics.interceptor';

export function provideAnalytics(): EnvironmentProviders {
  return makeEnvironmentProviders([
    AnalyticsService,
    provideHttpClient(withInterceptors([analyticsInterceptor])),
  ]);
}
```

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideAnalytics } from './analytics/provide-analytics';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter([]),
    provideAnalytics(), // sauber isoliert auf Environment-Level
  ],
};
```

```typescript
// isolierter Injector (z. B. in einem Unit-Test oder Skript)
import {
  createEnvironmentInjector,
  EnvironmentInjector,
  inject,
} from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { provideAnalytics } from './provide-analytics';
import { AnalyticsService } from './analytics.service';

describe('AnalyticsService (isolierter EnvironmentInjector)', () => {
  let childInjector: EnvironmentInjector;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    const parentInjector = TestBed.inject(EnvironmentInjector);
    childInjector = createEnvironmentInjector([provideAnalytics()], parentInjector);
  });

  afterEach(() => childInjector.destroy());

  it('trackt Events über runInContext', () => {
    const analytics = childInjector.runInContext(() => inject(AnalyticsService));

    analytics.track('app_start', { version: '1.0.0' });
    analytics.track('user_login', { userId: 42 });

    expect(analytics.eventCount()).toBe(2);
    expect(analytics.events()[0].event).toBe('app_start');
  });
});
```

## Weiterführendes

- **Angular Docs – Hierarchical Injectors:** [angular.dev/guide/di/hierarchical-dependency-injection](https://angular.dev/guide/di/hierarchical-dependency-injection) erklärt den Unterschied zwischen Element- und Environment-Injektoren im Detail.
- **Library-Autoren-Pattern:** Jede Angular Material- oder CDK-Funktion wie `provideAnimations()` oder `provideDaterangePicker()` nutzt intern `makeEnvironmentProviders` — lies deren Quellcode auf GitHub für inspirierende Patterns.
- **`ENVIRONMENT_INITIALIZER`:** Mit `{ provide: ENVIRONMENT_INITIALIZER, multi: true, useValue: () => { ... } }` kannst du Bootstrapping-Logik direkt in deinen `EnvironmentProviders` kapseln, ohne den App-Initializer zu benötigen.
