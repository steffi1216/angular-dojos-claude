# Angular Dojo: `provideX` Functional API & ApplicationConfig
**Datum:** 2026-10-09
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie moderne Angular-Apps ohne `NgModule` konfiguriert werden – mit `bootstrapApplication`, `ApplicationConfig` und eigenen `provideX`-Factory-Funktionen, die sauber gekapselte Feature-Konfigurationen ermöglichen.

## Hintergrund & Theorie

Seit Angular 15/16 ist das neue Bootstrapping-Modell der Standard: Statt eines Root-`AppModule` wird `bootstrapApplication(AppComponent, config)` verwendet. Die Konfiguration erfolgt über ein `ApplicationConfig`-Objekt mit einem `providers`-Array.

Angular selbst liefert alle wichtigen Features als funktionale Provider-Factories:
- `provideRouter(routes, withHashLocation(), withPreloading(...))`
- `provideHttpClient(withInterceptors([...]))`
- `provideAnimationsAsync()`
- `provideClientHydration(withEventReplay())`

Das Muster `provideX()` ist bewusst so gestaltet: Eine Funktion gibt ein `EnvironmentProviders`-Objekt (oder `Provider[]`) zurück, das intern alle nötigen Services, Tokens und Konfigurationen registriert. Mit `makeEnvironmentProviders()` kann dasselbe Muster für eigene Libraries und Feature-Bereiche genutzt werden.

Der Vorteil: Tree-Shaking greift optimal, die API ist typsicher, und Features können per Feature-Flag oder Build-Konfiguration ein- und ausgeschaltet werden.

## Aufgabe

Erstelle ein Angular-Projekt (Standalone) und implementiere zwei eigene `provideX`-Functions für wiederverwendbare Feature-Konfigurationen:

1. **`provideLogging(config: LoggingConfig)`** – registriert einen `LoggingService` plus ein HTTP-Interceptor, der alle Requests/Responses loggt (mit konfigurierbarem Log-Level).
2. **`provideFeatureFlags(flags: Record<string, boolean>)`** – stellt Feature-Flags per `InjectionToken` bereit und bietet eine `injectFeatureFlag(key)` Helper-Funktion.

Beide Functions sollen in `app.config.ts` genutzt werden können.

### Schritte

1. Erstelle `app.config.ts` mit `ApplicationConfig` und nutze `provideRouter`, `provideHttpClient(withInterceptors([...]))` – verstehe den Aufbau.

2. Erstelle `logging/logging.config.ts` mit dem `LoggingConfig`-Interface und einem `LOGGING_CONFIG`-Token:
   ```typescript
   export interface LoggingConfig { level: 'debug' | 'info' | 'warn' | 'error' }
   export const LOGGING_CONFIG = new InjectionToken<LoggingConfig>('LOGGING_CONFIG');
   ```

3. Erstelle den `LoggingService`, der `LOGGING_CONFIG` per `inject()` verwendet, und ein funktionales Interceptor-Factory:
   ```typescript
   export function loggingInterceptor(req, next) { ... }
   ```

4. Erstelle die `provideLogging(config)`-Funktion mit `makeEnvironmentProviders()`:
   ```typescript
   export function provideLogging(config: LoggingConfig): EnvironmentProviders {
     return makeEnvironmentProviders([
       { provide: LOGGING_CONFIG, useValue: config },
       LoggingService,
       withInterceptors([loggingInterceptor]) // Achtung: geht nicht direkt!
     ]);
   }
   ```
   Hinweis: HTTP-Interceptors können nicht über `makeEnvironmentProviders` in `provideHttpClient` eingefügt werden – finde die korrekte Lösung (Hint 2).

5. Erstelle `provideFeatureFlags(flags)` mit einem `FEATURE_FLAGS`-Token und einer `injectFeatureFlag(key: string): boolean`-Helper-Funktion, die intern `inject(FEATURE_FLAGS)` nutzt.

6. Registriere beides in `app.config.ts` und verifiziere die Funktion in einer Komponente, die `LoggingService` und `injectFeatureFlag('darkMode')` verwendet.

## Hints

<details>
<summary>Hint 1 – makeEnvironmentProviders</summary>

`makeEnvironmentProviders()` erwartet ein Array von `Provider`-Objekten (nicht `EnvironmentProviders`). Es verhindert, dass Environment-only Provider in Komponenten-Providern landen:

```typescript
import { makeEnvironmentProviders, EnvironmentProviders } from '@angular/core';

export function provideFeatureFlags(flags: Record<string, boolean>): EnvironmentProviders {
  return makeEnvironmentProviders([
    { provide: FEATURE_FLAGS, useValue: flags }
  ]);
}
```

</details>

<details>
<summary>Hint 2 – HTTP-Interceptors und provideHttpClient</summary>

Funktionale HTTP-Interceptors müssen über `provideHttpClient(withInterceptors([...]))` registriert werden. Da `provideHttpClient` selbst `EnvironmentProviders` zurückgibt, kannst du es direkt in deine Provider-Liste zurückgeben:

```typescript
export function provideLogging(config: LoggingConfig): EnvironmentProviders[] {
  return [
    makeEnvironmentProviders([
      { provide: LOGGING_CONFIG, useValue: config },
      LoggingService,
    ]),
    provideHttpClient(withInterceptors([loggingInterceptor])),
  ] as unknown as EnvironmentProviders[];
}
```

Oder eleganter: Nutze den Spread-Operator in `app.config.ts`:
```typescript
providers: [
  ...provideLogging({ level: 'debug' }), // Array zurückgeben
]
```

Gib aus `provideLogging` einfach `Provider[]` zurück (kein `makeEnvironmentProviders`) wenn HTTP-Interceptors enthalten sein sollen.

</details>

<details>
<summary>Hint 3 – injectFeatureFlag Helper</summary>

Eine Helper-Funktion die `inject()` nutzt, muss in einem Injection Context aufgerufen werden (Konstruktor, `inject()`-Initializer, `runInInjectionContext`):

```typescript
export const FEATURE_FLAGS = new InjectionToken<Record<string, boolean>>('FEATURE_FLAGS', {
  providedIn: 'root',
  factory: () => ({})
});

export function injectFeatureFlag(key: string): boolean {
  const flags = inject(FEATURE_FLAGS);
  return flags[key] ?? false;
}
```

In einer Komponente als class field:
```typescript
@Component(...)
export class MyComponent {
  readonly isDarkMode = injectFeatureFlag('darkMode'); // boolean
}
```

</details>

## Beispiellösung

```typescript
// feature-flags/feature-flags.ts
import { inject, InjectionToken, makeEnvironmentProviders, EnvironmentProviders } from '@angular/core';

export const FEATURE_FLAGS = new InjectionToken<Record<string, boolean>>('FEATURE_FLAGS', {
  providedIn: 'root',
  factory: () => ({}),
});

export function injectFeatureFlag(key: string): boolean {
  return inject(FEATURE_FLAGS)[key] ?? false;
}

export function provideFeatureFlags(flags: Record<string, boolean>): EnvironmentProviders {
  return makeEnvironmentProviders([
    { provide: FEATURE_FLAGS, useValue: flags },
  ]);
}

// logging/logging.config.ts
import { InjectionToken } from '@angular/core';
export interface LoggingConfig { level: 'debug' | 'info' | 'warn' | 'error' }
export const LOGGING_CONFIG = new InjectionToken<LoggingConfig>('LOGGING_CONFIG');

// logging/logging.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { tap } from 'rxjs/operators';
import { LOGGING_CONFIG } from './logging.config';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const config = inject(LOGGING_CONFIG);
  if (config.level === 'debug') {
    console.debug(`[HTTP] ${req.method} ${req.url}`);
  }
  return next(req).pipe(
    tap(event => {
      if (config.level === 'debug') {
        console.debug('[HTTP] Response:', event);
      }
    })
  );
};

// logging/provide-logging.ts
import { Provider } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { LoggingConfig, LOGGING_CONFIG } from './logging.config';
import { loggingInterceptor } from './logging.interceptor';
import { LoggingService } from './logging.service';

export function provideLogging(config: LoggingConfig): Provider[] {
  return [
    { provide: LOGGING_CONFIG, useValue: config },
    LoggingService,
    provideHttpClient(withInterceptors([loggingInterceptor])),
  ];
}

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';
import { provideLogging } from './logging/provide-logging';
import { provideFeatureFlags } from './feature-flags/feature-flags';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    ...provideLogging({ level: 'debug' }),
    provideFeatureFlags({
      darkMode: true,
      betaFeature: false,
    }),
  ],
};

// app.component.ts
import { Component } from '@angular/core';
import { injectFeatureFlag } from './feature-flags/feature-flags';
import { LoggingService } from './logging/logging.service';

@Component({
  selector: 'app-root',
  template: `
    <p>Dark Mode: {{ isDarkMode }}</p>
    <p>Beta: {{ isBeta }}</p>
  `,
})
export class AppComponent {
  readonly isDarkMode = injectFeatureFlag('darkMode');
  readonly isBeta = injectFeatureFlag('betaFeature');

  constructor(private logger: LoggingService) {
    logger.log('AppComponent initialized');
  }
}
```

## Weiterführendes

- [Angular Docs: Dependency Injection – provideX pattern](https://angular.dev/guide/di/dependency-injection-providers)
- Schau dir an, wie Angular's eigenes `provideRouter()` intern `makeEnvironmentProviders()` und Feature-Funktionen wie `withPreloading()` implementiert – der Quellcode auf GitHub ist sehr lehrreich.
- Muster: Nutze `EnvironmentProviders` als Return-Typ wenn deine Provider nur auf Application-Ebene (nicht in Komponenten) funktionieren sollen – Angular erzwingt das zur Compile-Zeit.
