# Angular Dojo: makeEnvironmentProviders – Saubere Provider-APIs für Libraries
**Datum:** 2026-11-15
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `makeEnvironmentProviders` und `EnvironmentProviders` wiederverwendbare, tree-shakeable Provider-Pakete baust – so wie es Angular selbst (z.B. `provideRouter`, `provideHttpClient`) macht.

## Hintergrund & Theorie

Seit Angular 15 gibt es den Typ `EnvironmentProviders`, der sicherstellt, dass bestimmte Provider **nur auf Umgebungsebene** (also in `bootstrapApplication` oder `ApplicationConfig`) registriert werden können – nicht in Komponenten-Providern. Das verhindert Bugs, bei denen Services versehentlich auf Komponentenebene instanziiert werden.

`makeEnvironmentProviders()` ist die Fabrikfunktion, mit der du diesen Typ erzeugst:

```typescript
import { makeEnvironmentProviders, EnvironmentProviders } from '@angular/core';

export function provideMyLibrary(config: MyConfig): EnvironmentProviders {
  return makeEnvironmentProviders([
    { provide: MY_CONFIG_TOKEN, useValue: config },
    MyService,
    { provide: HTTP_INTERCEPTORS, useClass: MyInterceptor, multi: true },
  ]);
}
```

Das TypeScript-Typsystem verhindert dann automatisch, dass jemand `provideMyLibrary()` in `@Component({ providers: [...] })` einsetzt – ein Kompilierfehler tritt auf, da `EnvironmentProviders` kein `Provider[]` ist.

Ergänzend gibt es `importProvidersFrom()`, um NgModule-basierte Provider in das Standalone-API zu überführen, und `provideExperimentalZonelessChangeDetection()` als Beispiel aus dem Angular-Core selbst.

Typische Anwendungsfälle:
- Library-APIs (`provideMyLibrary(config)`)
- Feature-Konfiguration (`provideFeatureX({ enabled: true })`)
- Kombination von Services, Tokens und Interceptors als ein logisches Paket

## Aufgabe

Entwickle eine kleine Logging-Library mit einer sauberen `provide`-API nach dem Angular-Pattern.

### Schritte

1. **Token und Konfigurationsinterface anlegen**
   Erstelle `InjectionToken<LogConfig>` mit einem Interface, das `level: 'debug' | 'warn' | 'error'` und `prefix: string` enthält.

2. **LoggingService implementieren**
   Erstelle einen `LoggingService` (providedin: `null`), der den Token injiziert und je nach Level `console.debug`, `console.warn` oder `console.error` mit Prefix aufruft.

3. **provideLogging() schreiben**
   Implementiere `provideLogging(config: LogConfig): EnvironmentProviders` mit `makeEnvironmentProviders`. Setze einen Default-Merge mit `Partial<LogConfig>`.

4. **LoggingInterceptor hinzufügen**
   Schreibe einen `HttpInterceptorFn`, der jede ausgehende Request per `LoggingService` loggt (URL und Methode). Registriere ihn in `provideLogging()` über `withInterceptors([loggingInterceptor])` von `provideHttpClient`.

5. **In bootstrapApplication verwenden**
   Registriere `provideLogging({ level: 'debug', prefix: '[App]' })` in `app.config.ts` und verifiziere, dass ein direkter Einsatz in `@Component({ providers: [] })` einen Typfehler produziert.

## Hints

<details>
<summary>Hint 1 – InjectionToken mit Default</summary>

```typescript
export interface LogConfig {
  level: 'debug' | 'warn' | 'error';
  prefix: string;
}

const DEFAULT_LOG_CONFIG: LogConfig = { level: 'warn', prefix: '[Log]' };

export const LOG_CONFIG = new InjectionToken<LogConfig>('LOG_CONFIG', {
  providedIn: 'root',
  factory: () => DEFAULT_LOG_CONFIG,
});
```

Mit `useValue: { ...DEFAULT_LOG_CONFIG, ...config }` in `makeEnvironmentProviders` überschreibst du den root-Default.
</details>

<details>
<summary>Hint 2 – makeEnvironmentProviders mit HttpInterceptorFn</summary>

```typescript
import { makeEnvironmentProviders, EnvironmentProviders } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export function provideLogging(config: Partial<LogConfig> = {}): EnvironmentProviders {
  const merged: LogConfig = { ...DEFAULT_LOG_CONFIG, ...config };
  return makeEnvironmentProviders([
    { provide: LOG_CONFIG, useValue: merged },
    provideHttpClient(withInterceptors([loggingInterceptor])),
  ]);
}
```

**Achtung:** Wenn `provideHttpClient` bereits global registriert ist, verwende stattdessen nur `withInterceptors` als separaten Feature-Call oder übergib den Interceptor über ein `ENVIRONMENT_INITIALIZER`-Pattern.
</details>

<details>
<summary>Hint 3 – LoggingInterceptor als Funktion</summary>

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const cfg = inject(LOG_CONFIG);
  const msg = `${cfg.prefix} ${req.method} ${req.url}`;
  if (cfg.level === 'debug') console.debug(msg);
  else if (cfg.level === 'warn') console.warn(msg);
  else console.error(msg);
  return next(req);
};
```
</details>

## Beispiellösung

```typescript
// log-config.token.ts
import { InjectionToken } from '@angular/core';

export interface LogConfig {
  level: 'debug' | 'warn' | 'error';
  prefix: string;
}

const DEFAULT_LOG_CONFIG: LogConfig = { level: 'warn', prefix: '[Log]' };

export const LOG_CONFIG = new InjectionToken<LogConfig>('LOG_CONFIG', {
  factory: () => DEFAULT_LOG_CONFIG,
});

export { DEFAULT_LOG_CONFIG };
```

```typescript
// logging.service.ts
import { inject, Injectable } from '@angular/core';
import { LOG_CONFIG, LogConfig } from './log-config.token';

@Injectable()
export class LoggingService {
  private cfg: LogConfig = inject(LOG_CONFIG);

  log(message: string): void {
    const output = `${this.cfg.prefix} ${message}`;
    if (this.cfg.level === 'debug') console.debug(output);
    else if (this.cfg.level === 'warn') console.warn(output);
    else console.error(output);
  }
}
```

```typescript
// logging.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { LoggingService } from './logging.service';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  inject(LoggingService).log(`${req.method} ${req.url}`);
  return next(req);
};
```

```typescript
// provide-logging.ts
import { EnvironmentProviders, makeEnvironmentProviders } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { DEFAULT_LOG_CONFIG, LOG_CONFIG, LogConfig } from './log-config.token';
import { LoggingService } from './logging.service';
import { loggingInterceptor } from './logging.interceptor';

export function provideLogging(config: Partial<LogConfig> = {}): EnvironmentProviders {
  return makeEnvironmentProviders([
    { provide: LOG_CONFIG, useValue: { ...DEFAULT_LOG_CONFIG, ...config } },
    LoggingService,
    provideHttpClient(withInterceptors([loggingInterceptor])),
  ]);
}
```

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideLogging } from './logging/provide-logging';

export const appConfig: ApplicationConfig = {
  providers: [
    provideLogging({ level: 'debug', prefix: '[MyApp]' }),
    // provideRouter([...]) etc.
  ],
};

// Das hier würde einen TypeScript-Fehler erzeugen:
// @Component({ providers: [provideLogging()] }) // ❌ EnvironmentProviders ist kein Provider
```

## Weiterführendes

- Schau dir an, wie Angular intern `provideRouter()` und `provideHttpClient()` aufgebaut sind – beide nutzen `makeEnvironmentProviders` und zeigen das vollständige Pattern mit optionalen Feature-Funktionen (`withXyz()`).
- Das Muster `provideX(...features: XFeature[])` mit einem eigenen Feature-Typ macht APIs noch flexibler und composable – ideal für größere Libraries.
- Offizielle Docs: [Angular – Dependency Injection – Environment Providers](https://angular.dev/guide/di/dependency-injection-providers#using-an-injectiontoken-object)
